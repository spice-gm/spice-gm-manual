# Introduction

The main content of this manual is to reform the Spice project (Protocol, Server, GTK Client, Web Client) to support TLS encrypted transmission based on the SM algorithm(Based on RFC 8998 & GB/T 38636-2020).

>Notice:
This project was done in a WSL2 environment, and the manual contains a chapter on configuring the virtualized environment in WSL.

本手册的主要内容是通过改造Spice项目(Protocol, Server, GTK Client, Web Client)，使之支持基于SM算法的TLS加密传输(参考 RFC 8998 & GB/T 38636-2020)。
> 注意：
本项目在WSL2环境下完成，手册中包含有关WSL虚拟化环境配置的章节。


### This repository is a gitbook, How to use it:

GitBook can be installed from **NPM** using:

```sh
npm install gitbook-cli -g
```

You can serve this repository as a book using:

```sh
gitbook serve
```
Or simply build the static website using:

```sh
gitbook build
```