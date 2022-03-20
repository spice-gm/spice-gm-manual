### 连接测试

```sh
# spice no tls
qemu-system-x86_64 -spice port=5930,disable-ticketing=on -drive file=/home/user/dev/spice/img-instance/fedora.img -enable-kvm -m 2048

# spice with tls
qemu-system-x86_64 -spice port=5930,tls-port=47001,disable-ticketing=on,x509-dir=/home/user/dev/spice/sm2_cert_files,tls-channel=main,tls-channel=inputs -drive file=/home/user/dev/spice/img-instance/fedora.img -enable-kvm -m 2048
```

### 连接示例

`Chrome`

![image-20220221221958925](../images/image-20220226232805378.png)
![image-20220221221958925](../images/image-20220226232829017.png)

`可信浏览器`

![image-20220221221958925](../images/image-20220226232854244.png)
