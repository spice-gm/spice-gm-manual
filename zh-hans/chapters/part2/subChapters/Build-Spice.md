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
./autogen.sh
./configure --enable-gstreamer=yes --enable-lz4=yes
make; sudo make install
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
