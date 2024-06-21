---
layout: post
title: foundry安装笔记
date: 2024-1-23
tags: 区块链
---
[Foundry快速安装（Windows版）](https://blog.csdn.net/weixin_51306597/article/details/130399689)

```JS
## 设置科学网络代理，windows安装rust
set all_proxy=http://10.9.1.???:???
set http_proxy=http://10.9.1.???:???
set https_proxy=http://10.9.1.???:???
## 启动安装程序后, 第一项参数设置成x86-xxxxx-gnu
run rustup-init.exe
## 如果超时了，用命令安装。rustc -v会有hint
rustup default stable 

## https://book.getfoundry.sh/getting-started/installation 安装foundry-build from source
# clone the repository
git clone https://github.com/foundry-rs/foundry.git
cd foundry
# install Forge
cargo install --path ./crates/forge --profile local --force --locked
# install Cast
cargo install --path ./crates/cast --profile local --force --locked
# install Anvil
cargo install --path ./crates/anvil --profile local --force --locked
# install Chisel
cargo install --path ./crates/chisel --profile local --force --locked
```