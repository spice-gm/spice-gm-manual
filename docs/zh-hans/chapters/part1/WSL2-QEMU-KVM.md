# WSL2 QEMU-KVM 环境搭建

##### 操作系统基础环境

+ 本机 Windows 11 + WSL + LxRunOffline
+ WSL2 Ubuntu 18.04

```sh
LxRunOffline install -n <WSL名称> -d <安装系统的路径> -f <镜像文件路径>\xxx.tar.gz -s

LxRunOffline install -n ubuntu18-spice -d D:\WSL\Instance\ubuntu18-spice\ -f D:\WSL\Ubuntu18\install.tar.gz -s

wsl --set-version ubuntu18-spice 2
```

建议利用*PowerShell*(管理员权限)设置允许*WSL*网卡通过防火墙

```powershell
New-NetFirewallRule -DisplayName "WSL" -Direction Inbound  -InterfaceAlias "vEthernet (WSL)"  -Action Allow
```

**重新编译Linux内核，优化 WSL 上 KVM 的内核配置**

```sh
git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
cd WSL2-Linux-Kernel
git checkout linux-msft-wsl-5.10.y

zcat /proc/config.gz > .config
sudo apt install ncurses-dev flex bison cpu-checker
make menuconfig
```

在 `Virtualization` 中确认 Intel 支持已被选中，`virtio-net` 也被选中 ( 如果有 )

再导航至 `Processor type and features -> Linux guest support`，确定 `KVM Guest support` 已被选中

以上提到的选项建议选中为*号，直接编译到内核中

```sh
apt install libelf-dev dwarves
make -j 8
sudo make modules_install

cp arch/x86/boot/bzImage /mnt/c/Users/<username>/bzImage
# bzImage已存在时会报错cannot create regular file
vim /mnt/c/Users/<username>/.wslconfig

# 写入
[wsl2]
nestedVirtualization=true
kernel=C:\\Users\\a1063\\bzImage
kernelCommandLine=intel_iommu=on iommu=pt kvm.ignore_msrs=1 kvm-intel.nested=1 kvm-intel.ept=1 kvm-intel.emulate_invalid_guest_state=0 kvm-intel.enable_shadow_vmcs=1 kvm-intel.enable_apicv=1
```

**关闭实例，重启WSL**

```sh
wsl --shutdown

#停止LxssManager服务
net stop LxssManager  
 
#启动LxssManager服务
net start LxssManager 
```

**配置 kvm-intel**

```sh
vim /etc/modprobe.d/kvm-nested.conf
```

写入

```sh
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1
```

**加载内核模块**

```sh
sudo modprobe kvm_intel
```

使用以下命令测试 KVM：

```sh
kvm-ok
```

检查嵌套的 KVM：

```sh
cat /sys/module/kvm_intel/parameters/nested # 输出Y
```
