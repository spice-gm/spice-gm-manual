### Handle Ticket With SM2

此小节主要包含改造 *spice-html5*，使其适配利用 SM2 与 *spice server* 进行 *Ticket* 口令验证的内容。

主要包括：

+ 利用 *sm2-EVP* 实现 *Ticket* 口令验证关键函数
  + 对 *Server* 传递的公钥进行还原
  + 将 *Password* 通过还原后公钥进行加密按格式传递
+ 利用 *Node.js Addons* 转为 *node* 插件
  + 处理 *Node*与*CPP* 之间参数传递
  + 利用 *express* 启动服务处理请求
+ spice-html5 接入该插件服务
