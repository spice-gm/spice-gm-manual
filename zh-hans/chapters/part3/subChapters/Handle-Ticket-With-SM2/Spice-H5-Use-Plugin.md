### Spice-HTML5 use sm-crypto-plugin-spice-html5

插件的使用集中在 *spice-html5* 连接 *spice-server* 过程中，包括 *SpiceLinkReply* 处理 *server* 消息，以及 *password* 口令加密发送过程中。

##### 环境搭建

首先是搭建 *sm-crypto-plugin-spice-html5* 服务

```sh
git clone https://github.com/Sovea/sm-crypto-plugin-spice-html5.git

cd sm-crypto-plugin-spice-html5

# 安装依赖
npm install

# 安装 node-gyp
npm install -g node-gyp

# node-gyp 生成配置
node-gyp configure

# node-gyp 编译
node-gyp build

# 启动 express 服务
node app.js
```

##### spice-html5 接入

> *SpiceLinkReply --> from_buffer*

```js
from_buffer: async function(a, at) {
    at = at || 0;
    var i;
    var orig_at = at;
    var dv = new SpiceDataView(a);
    this.error = dv.getUint32(at, true); at += 4;
    var keyData = dv.u8.slice(4, 4 + 162);
    let that = this;
    var url = 'https://127.0.0.1:18887/sm2-plugin/repair-pubKey'
    await $.ajax({
        url: url,
        data: {
            keyData: keyData.toString(),
        },
        dataType: 'json',
        type: 'POST',
        success: function (data) {
            that.pub_key = data.pubKey;
        }
    })
    at = 4;
    at += Constants.SPICE_TICKET_PUBKEY_BYTES;

    var num_common_caps = dv.getUint32(at, true); at += 4;
    var num_channel_caps  = dv.getUint32(at, true); at += 4;
    var caps_offset = dv.getUint32(at, true); at += 4;

    at = orig_at + caps_offset;
    this.common_caps = [];
    for (i = 0; i < num_common_caps; i++)
    {
        this.common_caps.unshift(dv.getUint32(at, true)); at += 4;
    }

    this.channel_caps = [];
    for (i = 0; i < num_channel_caps; i++)
    {
        this.channel_caps.unshift(dv.getUint32(at, true)); at += 4;
    }
}
```

> *SpiceConn --> process_inbound*

```js
case "link":
    this.reply_link = new SpiceLinkReply(mb);
    await this.reply_link.from_buffer(mb);

    if (this.reply_link.error) {
        this.state = "error";
        var e = new Error('Error: reply link error ' + this.reply_link.error);
        this.report_error(e);
    }
    else {
        let that = this;
        let encryptedPassword = "";
        var url = 'https://127.0.0.1:18887/sm2-plugin/encrypt-password-pubKey';
        await $.ajax({
            url: url,
            data: {
                pubKey: that.reply_link.pub_key,
                password: that.password + String.fromCharCode(0),
            },
            dataType: 'json',
            type: 'POST',
            success: function (data) {
                console.log(data);
                var dataArr = data.encryptedPassword.split(',');
                for (let i = 0; i < dataArr.length; i++) dataArr[i] = parseInt(dataArr[i]);
                encryptedPassword = dataArr;
            }
        })
        this.send_ticket(encryptedPassword);
        this.state = "ticket";
        this.wire_reader.request(SpiceLinkAuthReply.prototype.buffer_size());
    }
    break;
```
