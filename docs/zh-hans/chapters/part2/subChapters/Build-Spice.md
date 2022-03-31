### Spice-Protocol

```sh
git clone https://gitlab.freedesktop.org/spice/spice-protocol.git
cd spice-protocol
mkdir build && cd build

meson ../
ninja install
export PKG_CONFIG_PATH=/usr/local/share/pkgconfig/:/usr/local/lib/pkgconfig
```

### Spice

```sh
git clone https://gitlab.freedesktop.org/spice/spice.git
cd spice
git submodule update --init --recursive
apt install pkg-config m4 libtool automake autoconf autoconf-archive libglib2.0-dev
apt install libpixman-1-dev libcairo2-dev libpango1.0-dev libjpeg8-dev libgif-dev
apt install libopus-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev liblz4-dev
pip3 install pyparsing
pip install pyparsing
```

make 编译

```sh
./autogen.sh
./configure --enable-gstreamer=yes --enable-lz4=yes
make; sudo make install
```

也可使用 *Meson* 和 *Ninja* 编译 (改造时推荐)

```shell
mkdir build && cd build
meson ../
ninja install

# 偶发so动态链接库不更新
cp server/libspice-server.so.1 /usr/local/lib/libspice-server.so.1
cp server/libspice-server.so.1.14.1 /usr/local/lib/libspice-server.so.1.14.1
```
> 编译时偶发 *pyparsing* 缺失的报错，如已安装，应是编译时检测命令有问题。可修改使跳过pyparsing检测，不影响项目后续编译运行

*spice/spice/subprojects/spice-common/meson.build*
```makefile
foreach module : ['six', 'pyparsing'] # 删除 pyparsing
```

### Spice-GTK

```sh
git clone https://gitlab.freedesktop.org/spice/spice-gtk.git
cd spice-gtk
git submodule update --init --recursive
mkdir build && cd build

apt install libjson-glib-dev libgtk-3-dev
meson ../
ninja install
```
