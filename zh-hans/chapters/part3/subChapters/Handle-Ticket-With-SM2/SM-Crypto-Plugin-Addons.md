### sm-crypto-plugin-spice-html5

*<https://github.com/Sovea/sm-crypto-plugin-spice-html5>*

此项目是一个简单的 *sm-crypto* 插件，使用 *Node.js C++ Addons & [sm2-EVP](https://github.com/Sovea/sm2-EVP.git)  & express* 编写，为 *spice NTLS* 改造项目创建，尤其是 *[spice-html5-gm](https://github.com/Sovea/spice-html5.git)*

该项目的主要特点是辅助 *spice-html5* 在连接 *spice server* 时处理 *Ticket* 口令验证。 包括从服务器发送的信息中还原 *pubKey*，以及使用 *SM2* 加密密码并以要求格式发送给 *Server*

该项目给出了利用 *express* 接入该插件来处理 *spice-html5* 请求，从而完成以上所述功能的示例。
