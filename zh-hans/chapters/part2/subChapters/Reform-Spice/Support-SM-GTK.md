### Spice-GTK

https://github.com/spice-gm/spice-gtk-gm

##### 加密套件选择

对于客户端加密传输改造主要通过实现get_x509_from_PEM_file函数读取当前使用证书，使用check_x509_use_rsa函数通过其签名算法以此判断是否指定国密套件(国密套件的默认优先级低于原有的TLS1.3算法套件)。

##### 根据证书详情选择套件核心函数

```c
X509 *get_x509_from_PEM_file(const char* file_path) {
    X509 *cert = NULL;
    BIO *cert_bio = BIO_new(BIO_s_file());
    int ret = BIO_read_filename(cert_bio, file_path);
    if (! (cert = PEM_read_bio_X509(cert_bio, NULL, 0, NULL))) {
        g_warning("loading ca certs from default location failed");
        return NULL;
    }
    return cert;
}

int check_x509_use_rsa(X509 *cert) {
    EVP_PKEY *pk;
    pk = X509_get_pubkey(cert);
    if (EVP_PKEY_RSA == EVP_PKEY_base_id(pk)) {
        return 1;
    }
    return 0;
}

static int spice_channel_load_ca(SpiceChannel *channel)
{
    SpiceChannelPrivate *c = channel->priv;
    int i, count = 0;
    guint8 *ca;
    guint size;
    const gchar *ca_file;
    int rc;

    g_return_val_if_fail(c->ctx != NULL, 0);

    ca_file = spice_session_get_ca_file(c->session);
    spice_session_get_ca(c->session, &ca, &size);

    CHANNEL_DEBUG(channel, "Load CA, file: %s, data: %p", ca_file, ca);

    if (ca != NULL) {
        STACK_OF(X509_INFO) *inf;
        X509_STORE *store;
        BIO *in;

        store = SSL_CTX_get_cert_store(c->ctx);
        in = BIO_new_mem_buf(ca, size);
        inf = PEM_X509_INFO_read_bio(in, NULL, NULL, NULL);
        BIO_free(in);
        for (i = 0; i < sk_X509_INFO_num(inf); i++) {
            X509_INFO *itmp;
            itmp = sk_X509_INFO_value(inf, i);
            if (itmp->x509) {
                if (check_x509_use_rsa(itmp->x509)) ca_use_SM2 = FALSE;
                count++;
            }
            if (itmp->crl) {
                X509_STORE_add_crl(store, itmp->crl);
                count++;
            }
        }

        sk_X509_INFO_pop_free(inf, X509_INFO_free);
    }

    if (ca_file != NULL) {
        X509 *tmp_cert = get_x509_from_PEM_file(ca_file);
        if (tmp_cert) {
            if (check_x509_use_rsa(tmp_cert)) ca_use_SM2 = FALSE;
        }
        rc = SSL_CTX_load_verify_locations(c->ctx, ca_file, NULL);
        if (rc != 1)
            g_warning("loading ca certs from %s failed", ca_file);
        else
            count++;
    }

    if (count == 0) {
        X509 *tmp_cert = get_x509_from_PEM_file(ca_file);
        if (tmp_cert) {
            if (check_x509_use_rsa(tmp_cert)) ca_use_SM2 = FALSE;
        }
        rc = SSL_CTX_set_default_verify_paths(c->ctx);
        if (rc != 1)
            g_warning("loading ca certs from default location failed");
        else
            count++;
    }

    return count;
}

static void *spice_channel_coroutine(void *data) {
    ... ...
    if (c->tls) {
        c->ctx = SSL_CTX_new(TLS_method());
        if (c->ctx == NULL) {
            g_critical("SSL_CTX_new failed");
            c->event = SPICE_CHANNEL_ERROR_TLS;
            goto cleanup;
        }

        // SSL_CTX_set_options(c->ctx, ssl_options);
        SSL_CTX_set_min_proto_version(c->ctx, TLS1_3_VERSION);
        verify = spice_session_get_verify(c->session);
        if (verify &
            (SPICE_SESSION_VERIFY_SUBJECT | SPICE_SESSION_VERIFY_HOSTNAME)) {
            rc = spice_channel_load_ca(channel);
            if (ca_use_SM2 == TRUE) SSL_CTX_set_ciphersuites(c->ctx, "TLS_SM4_GCM_SM3");
            if (rc == 0) {
                g_warning("no cert loaded");
                if (verify & SPICE_SESSION_VERIFY_PUBKEY) {
                    g_warning("only pubkey active");
                    verify = SPICE_SESSION_VERIFY_PUBKEY;
                } else {
                    c->event = SPICE_CHANNEL_ERROR_TLS;
                    goto cleanup;
                }
            }
        }
    ... ...
}
```



##### 口令验证

Spice-gtk对应完成spice_channel_send_spice_ticket的逻辑拆分，利用sm2-EVP的C语言版本进行实现。使用spice-cmdline.c读取命令行参数，针对ticket-handler参数在列表中新增描述如下表所示。

| 字段            | 值                               |
| --------------- | -------------------------------- |
| long_name       | "ticket-handler"                 |
| short_name      | 't'                              |
| arg             | G_OPTION_ARG_STRING              |
| arg_data        | &ticket_handler                  |
| description     | N_("Algorithm to handle ticket") |
| arg_description | N_("<ticket_handler>")           |

表 4-2 ticket-handler参数字段设置

客户端连接时配置大都保存在spice-session.c的_SpiceSessionPrivate结构体中，新增ticket_handler字段，并增加枚举类型PROP_TICKET_HANDLER。将set/get处理逻辑添加至spice_session_set_property、spice_session_get_property中。利用g_object_class_install_property进行参数注册。

spice_channel_recv_link_msg通过ticket-handler值选择spice_channel_send_spice_ticket处理逻辑。

##### *Password* 口令验证 *SM2* 实现 --- GTK客户端

使用 `openssl EVP` 接口实现 *SM2* 工具库，*GTK* 需要 *C* 语言版本

> *sm2-EVP* 项目见 *<https://github.com/Sovea/sm2-EVP>*

##### *Ticket* 口令验证核心函数

```C
static SpiceChannelEvent spice_channel_send_spice_ticket_sm2(SpiceChannel *channel)
{
    SpiceChannelPrivate *c = channel->priv;
    EVP_PKEY *pubkey;
    int en_password_len;
    BIO *bioKey, *bioKey_pem;
    char *password, *en_password;
    uint8_t *encrypted;
    char *sm2_pubkey;
    int rc;
    SpiceChannelEvent ret = SPICE_CHANNEL_ERROR_LINK;
    spice_warning("Send Ticket With SM2.");
    bioKey = BIO_new(BIO_s_mem());
    g_return_val_if_fail(bioKey != NULL, ret);
    // wirte pub_key into biokey
    BIO_write(bioKey, c->peer_msg->pub_key, SPICE_TICKET_PUBKEY_BYTES);
    // Use BIO to get DER pubKey
    pubkey = d2i_PUBKEY_bio(bioKey, NULL);
    encrypted = g_alloca(128);

    // SM2 handle password
    bioKey_pem = BIO_new(BIO_s_mem());
    PEM_write_bio_PUBKEY(bioKey_pem, pubkey);
    bio_to_string(bioKey_pem, &sm2_pubkey);
    g_object_get(c->session, "password", &password, NULL);
    if (password == NULL) password = g_strdup("");
    if (strlen(password) > SPICE_MAX_PASSWORD_LENGTH) {
        spice_channel_failed_spice_authentication(channel, TRUE);
        ret = SPICE_CHANNEL_ERROR_AUTH;
        goto cleanup;
    }
    rc = Encrypt(password, strlen(password) + 1, &en_password, &en_password_len, sm2_pubkey);
    g_warn_if_fail(rc > 0);
    encrypted = (uint8_t *)en_password;
    // printf("Debug: encrypted: %s \n", encrypted);
    spice_channel_write(channel, encrypted, 128);
    ret = SPICE_CHANNEL_NONE;
    return ret;
cleanup:
    memset(encrypted, 0, 128);
    EVP_PKEY_free(pubkey);
    BIO_free(bioKey);
    g_free(password);
    return ret;
}

void bio_to_string_with_maxlen(BIO *bio, int max_len, char **str) {
    char buffer[max_len];
    memset(buffer, 0, max_len);
    BIO_read(bio, buffer, max_len - 1);
    *str = buffer;
}

void bio_to_string(BIO *bio, char **str) {
    char *temp;
    int readSize = (int)BIO_get_mem_data(bio, &temp);
    *str = (char *)malloc(readSize);
    BIO_read(bio, *str, readSize);
}
```

##### 改造后增加SM2工具类，编译

> meson.build 修改

```makefile
spice_client_glib_headers = [
  spice_version_h,
  'channel-cursor.h',
  'channel-display.h',
  'channel-inputs.h',
  'channel-main.h',
   ... ...
  'spice-uri.h',
  'spice-util.h',
  'usb-device-manager.h',
  'sm2.h', # Add
]

spice_client_glib_introspection_sources = [
  spice_client_glib_headers,
  spice_client_glib_enums,
  'channel-cursor.c',
  'channel-display.c',
  'channel-inputs.c',
  'channel-main.c',
  ... ...
  'sm2.c', # Add
]

spice_client_glib_sources = [
  spice_marshals,
  spice_client_glib_introspection_sources,
  'bio-gio.c',
  'bio-gio.h',
  'channel-base.c',
  'channel-display-gst.c',
  'channel-display-priv.h',
  'channel-playback-priv.h',
  'channel-usbredir-priv.h',
   ... ...
  'vmcstream.c',
  'vmcstream.h',
  'sm2.h', # Add
  'sm2.c', # Add
]
```

