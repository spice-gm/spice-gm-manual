### Spice-Server

##### *Password* 口令验证 *SM2* 实现 --- Server端

首先使用 `openssl EVP` 接口实现 *SM2* 工具库，*Server* 需要 *CPP* 版本
> *sm2-EVP* 项目见 *<https://github.com/Sovea/sm2-EVP>*

##### *Ticket* 口令验证核心函数

```cpp
static bool reds_send_link_ack_sm2(RedsState *reds, RedLinkInfo *link) {
    struct {
        SpiceLinkHeader header;
        SpiceLinkReply ack;
    } msg;
    RedChannel *channel;
    const RedChannelCapabilities *channel_caps;
    BUF_MEM *bmBuf;
    BIO *bio = nullptr;
    int ret = FALSE;
    size_t hdr_size;

    SPICE_VERIFY(sizeof(msg) == sizeof(SpiceLinkHeader) + sizeof(SpiceLinkReply));

    msg.header.magic = SPICE_MAGIC;
    hdr_size = sizeof(msg.ack);
    msg.header.major_version = GUINT32_TO_LE(SPICE_VERSION_MAJOR);
    msg.header.minor_version = GUINT32_TO_LE(SPICE_VERSION_MINOR);

    msg.ack.error = GUINT32_TO_LE(SPICE_LINK_ERR_OK);

    channel = reds_find_channel(reds, link->link_mess->channel_type,
                                link->link_mess->channel_id);
    if (!channel) {
        if (link->link_mess->channel_type != SPICE_CHANNEL_MAIN) {
            spice_warning("Received wrong header: channel_type != SPICE_CHANNEL_MAIN");
            return FALSE;
        }
        spice_assert(reds->main_channel);
        channel = reds->main_channel.get();
    }

    reds_channel_init_auth_caps(link, channel); /* make sure common caps are set */

    channel_caps = channel->get_local_capabilities();
    msg.ack.num_common_caps = GUINT32_TO_LE(channel_caps->num_common_caps);
    msg.ack.num_channel_caps = GUINT32_TO_LE(channel_caps->num_caps);
    hdr_size += channel_caps->num_common_caps * sizeof(uint32_t);
    hdr_size += channel_caps->num_caps * sizeof(uint32_t);
    msg.header.size = GUINT32_TO_LE(hdr_size);
    msg.ack.caps_offset = GUINT32_TO_LE(sizeof(SpiceLinkReply));
    if (!reds->config->sasl_enabled
        || !red_link_info_test_capability(link, SPICE_COMMON_CAP_AUTH_SASL)) {

        if (!(bio = BIO_new(BIO_s_mem()))) {
            spice_warning("BIO new failed");
            red_dump_openssl_errors();
            return FALSE;
        }
        // SM2 handle
        // generate EC key pair(pem)
        sm2Handler.GenEcPairKey(link->tiTicketing.priKey, link->tiTicketing.pubKey);
        unsigned char *pubKeyArray = (unsigned char *)link->tiTicketing.pubKey.c_str();
        // Get EVP_PKEY
        sm2Handler.CreateEVP_PKEY(pubKeyArray, 1, &link->tiTicketing.evp_pkey);
        link->tiTicketing.ec_key = EVP_PKEY_get1_EC_KEY(link->tiTicketing.evp_pkey);
        i2d_EC_PUBKEY_bio(bio, link->tiTicketing.ec_key);
        BIO_get_mem_ptr(bio, &bmBuf);

        memcpy(msg.ack.pub_key, bmBuf->data, sizeof(msg.ack.pub_key));
    } else {
        /* if the client sets the AUTH_SASL cap, it indicates that it
         * supports SASL, and will use it if the server supports SASL as
         * well.
         */
        spice_warning("not initialising SM2 key");
        memset(msg.ack.pub_key, '\0', sizeof(msg.ack.pub_key));
    }

    if (!red_stream_write_all(link->stream, &msg, sizeof(msg)))
        goto end;
    for (unsigned int i = 0; i < channel_caps->num_common_caps; i++) {
        guint32 cap = GUINT32_TO_LE(channel_caps->common_caps[i]);
        if (!red_stream_write_all(link->stream, &cap, sizeof(cap)))
            goto end;
    }
    for (unsigned int i = 0; i < channel_caps->num_caps; i++) {
        guint32 cap = GUINT32_TO_LE(channel_caps->caps[i]);
        if (!red_stream_write_all(link->stream, &cap, sizeof(cap)))
            goto end;
    }

    ret = TRUE;

end:
    if (bio != nullptr) BIO_free(bio);
    return ret;
}

static void reds_handle_ticket_sm2(void *opaque) {
    auto link = static_cast<RedLinkInfo *>(opaque);
    RedsState *reds = link->reds;
    char *password;
    int password_size;

    string encrypted_data_str((char *)link->tiTicketing.encrypted_ticket.encrypted_data, 128);
    string decrypted_password;
    int len_plaint = 0;

    password_size = sm2Handler.Decrypt(encrypted_data_str, encrypted_data_str.length(), decrypted_password, len_plaint, link->tiTicketing.priKey);

    if (reds->config->ticketing_enabled && !link->skip_auth) {
        time_t ltime;
        bool expired;

        if (strlen(reds->config->taTicket.password) == 0) {
            spice_warning("Ticketing is enabled, but no password is set. "
                          "please set a ticket first");
            goto error;
        }

        ltime = spice_get_monotonic_time_ns() / NSEC_PER_SEC;
        expired = (reds->config->taTicket.expiration_time < ltime);

        if (expired) {
            spice_warning("Ticket has expired");
            // goto error;
        }

        if (strcmp(decrypted_password.c_str(), reds->config->taTicket.password) != 0) {
            spice_warning("Invalid password");
            goto error;
        }
    }

    reds_handle_link(link);
    return;

error:
    reds_send_link_result(link, SPICE_LINK_ERR_PERMISSION_DENIED);
    reds_link_free(link);
}

static void reds_get_spice_ticket_sm2(RedLinkInfo *link) {
    red_stream_async_read(
        link->stream,
        reinterpret_cast<uint8_t *>(&link->tiTicketing.encrypted_ticket.encrypted_data),
        128, reds_handle_ticket_sm2, link);
}
```
