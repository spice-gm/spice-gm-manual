### Spice-GTK

###### 指定加密套件
> src/spice-channel.c spice_channel_coroutine函数

```c
if (c->tls) {
... ...
SSL_CTX_set_options(c->ctx, ssl_options);

SSL_CTX_set_ciphersuites(c->ctx, "TLS_SM4_GCM_SM3");
... ...
}
```
##### *Password* 口令验证 *SM2* 实现 --- GTK客户端

首先使用 `openssl EVP` 接口实现 *SM2* 工具库，*GTK* 需要 *C* 语言版本
> *sm2-EVP* 项目见 *<https://github.com/Sovea/sm2-EVP>*

##### *Ticket* 口令验证核心函数
```C
static SpiceChannelEvent spice_channel_send_spice_ticket_sm2(SpiceChannel *channel) {
    SpiceChannelPrivate *c = channel->priv;
    EVP_PKEY *pubkey;
    int en_password_len;
    BIO *bioKey, *bioKey_pem;
    char *password, *en_password;
    uint8_t *encrypted;
    char *sm2_pubkey;
    int rc;
    SpiceChannelEvent ret = SPICE_CHANNEL_ERROR_LINK;
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

