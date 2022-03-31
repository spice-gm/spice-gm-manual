### BaBaSSL编译

```sh
git clone https://github.com/BabaSSL/BabaSSL.git
cd BabaSSL
apt install make gcc

./config enable-ntls
make; sudo make install

cp libssl.so.1.1 /usr/lib/x86_64-linux-gnu/libssl.so.1.1
cp libcrypto.so.1.1 /usr/lib/x86_64-linux-gnu/libcrypto.so.1.1
```
