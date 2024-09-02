---
layout: post
title: substrate部署链笔记
date: 2024-6-19
tags: 区块链
---
[substrate.io文档](https://docs.substrate.io/)

## 一、安装环境(Linux)
1. 安装substrate
- [install_linux](https://docs.substrate.io/install/linux/) 按此页命令顺序安装。
- `clang curl git make` 报错不用管，后续命令会安装。
- 【Test】[quick_start_a_node](https://docs.substrate.io/quick-start/start-a-node/) 从`View information for the node`开始，可以验证节点模板。

2. 安装yarn
```BASH
# 安装node v14以上和npm
# 官方只推荐用corepack管理和安装(yarn) https://yarnpkg.com/getting-started/install
npm instal -g corepack
corepack enable
yarn init -2
```

3. 清理数据库、生成密钥命令

```bash
# remove database
./target/release/node-template purge-chain --base-path /tmp/node01 --chain customSpecRaw.json

# 随机生成key
# key类型： aura - Sr25519    
# key类型： GRANDPA - Ed25519
# key类型link https://docs.substrate.io/deploy/keys-and-network-operations/
./target/release/node-template key generate --scheme Sr25519 --password-interactive
# 通过助记词得到key
./target/release/node-template key inspect --scheme Ed25519 --password-interactive "code1 code2 ... coden"
```

## 集群启动 add trusted nodes
1. 生成密钥（见上）
2. 创建区块链定义文件

```bash
# 从预设的local(development/staging)环境导出成json规格文件
./target/release/node-template build-spec --disable-default-bootnode --chain local > customSpec.json

./target/release/node-template build-spec --disable-default-bootnode --chain dev > customSpec-dev.json

# 修改customSpec.json
# "aura" 加入生成的Sr25519密钥的SS58地址，此为创造区块职责
# "grandpa" 加入生成的Ed25519密钥的SS58地址，此为确认区块职责。数字1，表示投票时节点的权重是1。

# 通过json规格文件构造原码执行文件
./target/release/node-template build-spec --chain=customSpec-dev.json --raw --disable-default-bootnode > customSpecRaw-dev.json
```

3. 启动节点1
- `base-path`:  数据存储目录 - 删除数据库时用此目录
- `chain`: 原码文件Spec
- `port`: 通信端口
- `rpc-port`: json rpc端口
- 外网ws： `--unsafe-rpc-external`
- 本地不安全ws： `--rpc-methods Unsafe`
- 本地安全ws： `--rpc-methods Safe`
- ws最大连接数： `--ws-max-connections 5000`

```bash
./target/release/node-template \
  --base-path /tmp/node01 \
  --chain ./customSpecRaw.json \
  --port 30333 \
  --rpc-port 9945 \
  --telemetry-url "wss://telemetry.polkadot.io/submit/ 0" \
  --validator \
  --rpc-methods Unsafe \
  --unsafe-rpc-external \
  --name MyNode01 \
  --password-interactive
```

4. Add keys to the keystore
```bash
# aura密钥， 填充助记词
# <your-secret-seed> e.g. --suri "pig giraffe ceiling enter weird liar orange decline behind total despair fly"
./target/release/node-template key insert --base-path /tmp/node01 \
  --chain customSpecRaw.json \
  --scheme Sr25519 \
  --suri <your-secret-seed> \
  --password-interactive \
  --key-type aura

# grandfa密钥
./target/release/node-template key insert \
  --base-path /tmp/node01 \
  --chain customSpecRaw.json \
  --scheme Ed25519 \
  --suri <your-secret-key> \
  --password-interactive \
  --key-type gran

# 通过下面这个命令，应该能看到2个keys
# "/tmp/node01/" 是你设置的"--base-path", local_testnet是json规格文件里的id
ls /tmp/node01/chains/my_custom_testnet/keystore
```

5. 然后重启节点1 
- ctrl c中断，然后用相同的命令启动即可

6. add key script

```bash
# node 1
./target/release/node-template key insert \
  --base-path /tmp/node01 \
  --chain customSpecRaw.json \
  --scheme Sr25519 \
  --suri "effort prevent stone lion when jelly grace alter machine physical better execute" \
  --password-interactive \
  --key-type aura

  
./target/release/node-template key insert \
  --base-path /tmp/node01 \
  --chain customSpecRaw.json \
  --scheme Ed25519 \
  --suri "effort prevent stone lion when jelly grace alter machine physical better execute" \
  --password-interactive \
  --key-type gran

# node 2
./target/release/node-template key insert \
  --base-path /tmp/node02 \
  --chain customSpecRaw.json \
  --scheme Sr25519 \
  --suri "short enroll butter prize whisper soccer spatial come good core wrap sick" \
  --password-interactive \
  --key-type aura

  
./target/release/node-template key insert \
  --base-path /tmp/node02 \
  --chain customSpecRaw.json \
  --scheme Ed25519 \
  --suri "short enroll butter prize whisper soccer spatial come good core wrap sick" \
  --password-interactive \
  --key-type gran
```

## 启动节点2
```bash
# bootnodes 格式： /ip4/{node1-ip}/tcp/{node1-port}/p2p/{node1-identity-print-while-start}
# ./customSpecRaw.json 文件必须与node1的原码文件一模一样，不然创世块会不同，随后加入失败。
./target/release/node-template \
  --base-path /tmp/node02 \
  --chain ./customSpecRaw.json \
  --port 30334 \
  --rpc-port 9946 \
  --telemetry-url "wss://telemetry.polkadot.io/submit/ 0" \
  --validator \
  --rpc-methods Unsafe \
  --name MyNode02 \
  --bootnodes /ip4/100.25.117.101/tcp/30333/p2p/12D3KooWC5sqd2ZBVAJ1fTBmUcmQWhTRHrchiPARgpVhUZuvwtQt \
  --password-interactive
```
