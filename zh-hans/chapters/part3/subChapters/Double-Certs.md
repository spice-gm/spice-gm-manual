##### 双证书配置

> nginx.conf 443端口配置双证书-RSA/SM2(双证书)

```nginx
server {
    listen       443 ssl;
    server_name  localhost;
 enable_ntls  on;
 
 # rsa
 ssl_certificate      /home/sovea/dev/spice/rsa_cert_files/server-cert.pem;
 ssl_certificate_key  /home/sovea/dev/spice/rsa_cert_files/server-key.pem;
 
 # sm2
 ssl_sign_certificate /home/sovea/dev/BabaSSL/test_certs/double_cert/SS.cert.pem;
 ssl_sign_certificate_key /home/sovea/dev/BabaSSL/test_certs/double_cert/SS.key.pem;
 ssl_enc_certificate /home/sovea/dev/BabaSSL/test_certs/double_cert/SE.cert.pem;
 ssl_enc_certificate_key /home/sovea/dev/BabaSSL/test_certs/double_cert/SE.key.pem;
 ssl_session_cache    shared:SSL:1m;
 ssl_session_timeout  5m;
 
 # ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

 location / {
     root   /var/www;
     index  index.html index.htm;
 }
}
```

> conf.d/spice-web-wss.conf   8888端口配置双证书监听转发给5930(spice-server)

```nginx
map $http_upgrade $connection_upgrade {  
    default upgrade;  
    '' close;  
}

server {
  listen 8888 ssl;
  server_name localhost;
 
  #charset koi8-r;
 
  #access_log  logs/host.access.log  main;
  enable_ntls  on;
  ssl_certificate      /home/sovea/dev/spice/rsa_cert_files/server-cert.pem;
  ssl_certificate_key  /home/sovea/dev/spice/rsa_cert_files/server-key.pem;
  ssl_sign_certificate /home/sovea/dev/BabaSSL/test_certs/double_cert/SS.cert.pem;
  ssl_sign_certificate_key /home/sovea/dev/BabaSSL/test_certs/double_cert/SS.key.pem;
  ssl_enc_certificate /home/sovea/dev/BabaSSL/test_certs/double_cert/SE.cert.pem;
  ssl_enc_certificate_key /home/sovea/dev/BabaSSL/test_certs/double_cert/SE.key.pem;
  ssl_session_timeout 5m;

  ssl_prefer_server_ciphers on;
  ssl_verify_client off;
  location / {
    proxy_http_version 1.1;
    proxy_pass http://127.0.0.1:5930;
    proxy_set_header Upgrade $http_upgrade;  
    proxy_set_header Connection "Upgrade";  
  }
}
```
