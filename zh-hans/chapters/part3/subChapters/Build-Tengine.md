### Tengine

> Tengine+SM2/RSA双证书360浏览器支持国密，Chrome等主流浏览器使用RSA证书相关套件

```sh
git clone https://github.com/alibaba/tengine.git
cd tengine
./configure --add-module=modules/ngx_openssl_ntls \
--with-openssl=../BabaSSL \
--with-openssl-opt="--strict-warnings enable-ntls" \
--with-http_ssl_module --with-stream \
--with-stream_ssl_module --with-stream_sni

make; sudo make install

/usr/local/nginx/sbin/nginx -v

/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```
