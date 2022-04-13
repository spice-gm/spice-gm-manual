### Spice-html5 改造

https://github.com/spice-gm/spice-html5-gm

https://github.com/spice-gm/sm-crypto-plugin-spice-html5

##### 改造方案

对应C/S口令验证改造，Spice-html5也需兼容SM2/RSA两种验证方案。通过Webassembly或Node.js Addons复用SM2-EVP，产出wasm或node插件。对于Webassembly，需要安装配置Emscripten等编译环境，对于Node.js Addons需在WSL中利用NVM安装Node及npm环境。

使用Webassembly，在编译复用前，需对依赖OpenSSL/BabaSSL使用emconfigure及emmake编译动态链接库。再通过emcc对复用逻辑指定上述链接库及头文件路径进行编译，对需导出函数使用EMSCRIPTEN_KEEPALIVE声明，产出wasm及js胶水文件。

使用Node.js Addons，利用node-gyp进行编译，将函数通过NODE_SET_METHOD进行导出。并使用express搭建简单后台服务，接入编译后node插件提供改造所需还原公钥、数据加密等能力。特别的，在使用Node.js Addons时需将参数类型进行转换，使两者能进行数据传输。 

口令身份验证的控制参数由Option标签实现，默认指定SM2算法。Spice-html5口令验证逻辑处理主要存在于SpiceConn类的process_inbound函数及SpiceLinkReply类的from_buffer函数，在两环节中通过ticket-handler值判断选用SM2/RSA。



##### 核心代码

> spice.html

```html
<div id="Sidenav" class="SidenavClosed" style="width: 0;">
    <p class="closebtn" onclick="close_nav()">&#10006;</p>
    <label for="host">Host:</label> <input type='text' id='host' value='localhost'> <!-- localhost --><br>
    <label for="port">Port:</label> <input type='text' id='port' value='5959'><br>
    <label for="password">Password:</label> <input type='password' id='password' value=''><br>
    <label for="if_websocket_secure">Protocol:</label> <select id="if_websocket_secure"> </select> <br>
    <label for="if_ntls">Safety Option:</label>
    <select id="if_ntls">
        <option value="ntls">NTLS</option>
        <option value="tls">TLS</option>
    </select> <br>
    <label for="if_sm2">Ticket Handler:</label>
    <select id="if_sm2">
        <option value="sm2">SM2</option>
        <option value="rsa">RSA</option>
    </select> <br>
    <button id="connectButton">Start Connection</button><br>
    <button id="sendCtrlAltDel">Send Ctrl-Alt-Delete</button>
    <button id="debugLogs">Toggle Debug Logs</button>
    <div id="message-div" class="spice-message" style="display: none;"></div>

    <div id="debug-div">
        <!-- If DUMPXXX is turned on, dumped images will go here -->
    </div>
</div>
```

```javascript
if (window.location.protocol == 'https:') {
    document.getElementById("if_websocket_secure").append(new Option("wss", "wss://"));
} else {
    document.getElementById("if_websocket_secure").append(new Option("ws", "ws://"));
}

function connect() {
    var host, port, password, scheme = "ws://", uri, tls_option, ticket_handler_option;

    host = document.getElementById("host").value;
    port = document.getElementById("port").value;
    password = document.getElementById("password").value;
    var select_websocket_scheme = document.getElementById("if_websocket_secure");
    scheme = select_websocket_scheme.options[select_websocket_scheme.selectedIndex].value;
    var select_websocket_tls_option = document.getElementById("if_ntls");
    tls_option = select_websocket_tls_option.options[select_websocket_tls_option.selectedIndex].value;
    var select_ticket_handler_option = document.getElementById("if_sm2");
    ticket_handler_option = select_ticket_handler_option.options[select_ticket_handler_option.selectedIndex].value;
    if ((!host) || (!port)) {
        console.log("must set host and port");
        return;
    }

    if (sc) {
        sc.stop();
    }

    uri = scheme + host + ":" + port + "/" + tls_option;

    document.getElementById('connectButton').innerHTML = "Stop Connection";
    document.getElementById('connectButton').onclick = disconnect;
    try {
        sc = new SpiceHtml5.SpiceMainConn({
            uri: uri, screen_id: "spice-screen", dump_id: "debug-div", ticket_handler: ticket_handler_option,
            message_id: "message-div", password: password, onerror: spice_error, onagent: agent_connected
        });
    }
    catch (e) {
        alert(e.toString());
        disconnect();
    }

}
```

> spiceconn.js SpiceConn.process_inbound

```javascript
process_inbound: async function (mb, saved_header) {
    ... ...
    case "link":
        this.reply_link = new SpiceLinkReply(mb, 0, { ticket_handler: this.ticket_handler });
        await this.reply_link.from_buffer(mb);
        if (this.reply_link.error) {
            this.state = "error";
            var e = new Error('Error: reply link error ' + this.reply_link.error);
            this.report_error(e);
        }
        else {
            if (this.ticket_handler == "rsa") {
                this.send_ticket(rsa_encrypt(this.reply_link.pub_key, this.password + String.fromCharCode(0)));
            } else {
                let that = this;
                let encryptedPassword = "";
                var url = 'https://xxx.xxx.xxx.xxx:18887/sm2-plugin/encrypt-password-pubKey'
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
                        console.log(dataArr);
                        encryptedPassword = dataArr;
                    }
                })
                this.send_ticket(encryptedPassword);
            }
            this.state = "ticket";
            this.wire_reader.request(SpiceLinkAuthReply.prototype.buffer_size());
        }
        break;
    ... ...
}
```

> spicemsg.js SpiceLinkReply.from_buffer

```javascript
from_buffer: async function (a, at) {
    at = at || 0;
    var i;
    var orig_at = at;
    var dv = new SpiceDataView(a);
    this.error = dv.getUint32(at, true); at += 4;
    if (this.options.ticket_handler == "rsa") {
        this.pub_key = create_rsa_from_mb(a, at);
    } else {
        var keyData = dv.u8.slice(4, 4 + 162);
        let that = this;
        var url = 'https://xxx.xxx.xxx.xxx:18887/sm2-plugin/repair-pubKey'
        await $.ajax({
            url: url,
            data: {
                keyData: keyData.toString(),
            },
            dataType: 'json',
            type: 'POST',
            success: function (data) {
                console.log(data);
                that.pub_key = data.pubKey;
            }
        })
        at = 4;
    }
    at += Constants.SPICE_TICKET_PUBKEY_BYTES;

    var num_common_caps = dv.getUint32(at, true); at += 4;
    var num_channel_caps = dv.getUint32(at, true); at += 4;
    var caps_offset = dv.getUint32(at, true); at += 4;

    at = orig_at + caps_offset;
    this.common_caps = [];
    for (i = 0; i < num_common_caps; i++) {
        this.common_caps.unshift(dv.getUint32(at, true)); at += 4;
    }

    this.channel_caps = [];
    for (i = 0; i < num_channel_caps; i++) {
        this.channel_caps.unshift(dv.getUint32(at, true)); at += 4;
    }
}
```

