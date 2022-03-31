### QEMU v6.1.0

```sh
wget https://download.qemu.org/qemu-6.1.0.tar.xz
tar -xf qemu-6.1.0.tar.xz
cd qemu-6.1.0
apt install libaio-dev libiscsi-dev libnfs-dev libvirglrenderer-dev librados-dev librbd-dev libbz2-dev liblzo2-dev libusbredirparser-dev libusb-1.0-0-dev libcap-ng-dev libattr1-dev

./configure --target-list=x86_64-softmmu,x86_64-linux-user --enable-kvm --enable-libusb --enable-usb-redir --enable-lzo --enable-bzip2 --enable-libiscsi --enable-libnfs --enable-spice --enable-linux-aio --enable-virtfs --enable-gtk --enable-rbd --enable-virglrenderer --enable-tools

make; sudo make install
```

### QEMU 镜像创建

```shell
mkdir img-instance && cd img-instance
qemu-img create -f qcow2 fedora.img 10G

wget https://archives.fedoraproject.org/pub/archive/fedora/linux/releases/20/Live/x86_64/Fedora-Live-Desktop-x86_64-20-1.iso
# 安装 
qemu-system-x86_64 -m 2048 -enable-kvm fedora.img -cdrom ./Fedora-Live-Desktop-x86_64-20-1.iso
# 启动
qemu-system-x86_64 -m 2048 -enable-kvm fedora.img
```
