---
layout: post
title: polygon-cdk部署笔记
date: 2024-6-13
tags: 区块链
---
[polygon-cdk Deploy文档](https://docs.polygon.technology/cdk/build/quickstart/deploy-stack/#prerequisites)
[Kurtosis 安装文档](https://docs.kurtosis.com/install/)
[Foundry 安装文档](https://book.getfoundry.sh/getting-started/installation)

## 一、安装环境与运行项目

```bash
# 1. 安装环境
## 安装docker(此命令版本不匹配，查看附录)
sudo apt install docker-compose

## 安装kurtosis-cli latest(此命令版本不匹配，查看附录)
echo "deb [trusted=yes] https://apt.fury.io/kurtosis-tech/ /" | sudo tee /etc/apt/sources.list.d/kurtosis.list
sudo apt update
sudo apt install kurtosis-cli


# 2. pull git项目
git clone https://github.com/0xPolygon/kurtosis-cdk.git
cd kurtosis-cdk

# 3. 检查工具包是否齐全
#### a. 远程检查
curl -s https://raw.githubusercontent.com/0xPolygon/kurtosis-cdk/main/scripts/tool_check.sh | bash
#### b. kurtosis-cdk项目根目录检查
./scripts/tool_check.sh

## 如果检查版本有问题，参考下一章《软件工具特定版本》

# 4. 运行 Kurtosis enclave.(项目根目录)
sudo su root ##需要root用户，sudo不行
kurtosis run --enclave cdk-v1 --args-file params.yml --image-download always .

# 5. 一些其他可能会用到的命令
## 安装foundry_setup
curl -L https://foundry.paradigm.xyz | bash
source /root/.bashrc
foundryup

## 清除kurtosis环境
kurtosis clean --all

## 关闭docker所有容器
 sudo docker stop $(sudo docker ps -q)
## 删除docker所有容器
 sudo docker rm $(sudo docker ps -aq)

```

## 测试
```BASH
# 修改端口
export ETH_RPC_URL="$(sudo kurtosis port print cdk-v1 zkevm-node-rpc-001 http-rpc)"
## 永久修改  nano /root/.bashrc
## export ETH_RPC_URL= ...
# 查询
cast block-number

```

## 附录1、部署在指定l1
1. 修改`params.yml`
```bash
# Deploy local L1.
deploy_l1: false

l1_chain_id: 11155111
# The first account for the mnemonic should have enough funds to cover the deployment costs and the l1_funding_amount transfers to L1 addresses.
l1_preallocated_mnemonic: <mnemonic_for_a_funded_L1_account>
# The amount of initial funding to L1 addresses performing services like the Sequencer, Aggregator, Admin, Agglayer, Claimtxmanager.
# make sure your addresses have enough ether, then you can set 0ether. 
l1_funding_amount: 0ether
l1_rpc_url: <Sepolia_RPC_URL>
l1_ws_url: <Sepolia_WS_URL>
```
2. 修改 cdk_bridge_infra.star
```bash
 sudo nano cdk_bridge_infra.star
...
# l1rpc_service = plan.get_service("el-1-geth-lighthouse")
...
# "l1rpc_ip": l1rpc_service.ip_address,
# "l1rpc_port": l1rpc_service.ports["rpc"].number,
```

3. 修改 templates/bridge-infra/haproxy.cfg
```bash
sudo nano templates/bridge-infra/haproxy.cfg

...
    # acl url_l1rpc path_beg /l1rpc
...
    # use_backend backend_l1rpc if url_l1rpc
...   
# backend backend_l1rpc
#     http-request set-path /
#     server server1 {{.l1rpc_ip}}:{{.l1rpc_port}}
```


4. 修改 deploy_parameters.json
```bash
sudo nano templates/contract-deploy/deploy_parameters.json
# 修改salt, 每个salt用一次
"salt": ...
# test-mode
"test": false
```

5. 自定义zkevm_contracts docker
```bash
cd docker
sudo nano zkevm-contracts.Dockerfile
# change the official git repository to your own repository
# RUN git clone --branch ${ZKEVM_CONTRACTS_BRANCH} https://github.com/0xPolygonHermez/zkevm-contracts . \
RUN git clone --branch ${ZKEVM_CONTRACTS_BRANCH} https://github.com/{your_fork_repository}/zkevm-contracts . \

# build local docker
sudo docker build . \
  --tag local/zkevm-contracts:fork9 \
  --build-arg ZKEVM_CONTRACTS_BRANCH=main \
  --build-arg POLYCLI_VERSION=main \
  --file zkevm-contracts.Dockerfile

# 修改params.yml中的docker镜像
# zkevm_contracts_image: leovct/zkevm-contracts 
  zkevm_contracts_image: local/zkevm-contracts # the tag is automatically replaced by the value of /zkevm_rollup_fork_id/

```

## 附录2、软件工具特定版本

### docker最新版本
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 检查docker版本
sudo docker version
```

### kurtosis-cli 历史版本（0.89）
```bash
## 安装kurtosis-cli 历史版本-git-release（0.89）
wget https://github.com/kurtosis-tech/kurtosis-cli-release-artifacts/releases/download/0.89.18/kurtosis-cli_0.89.18_linux_amd64.tar.gz

sudo tar -zxvf kurtosis-cli_0.89.18_linux_amd64.tar.gz  -C /usr/bin
```

















## log
```js

========================================== User Services ==========================================
UUID           Name                                             Ports                                                     Status
1ce5fab58ed3   cl-1-lighthouse-geth                             http: 4000/tcp -> http://127.0.0.1:32776                  RUNNING
                                                                metrics: 5054/tcp -> http://127.0.0.1:32775
                                                                tcp-discovery: 9000/tcp -> 127.0.0.1:32774
                                                                udp-discovery: 9000/udp -> 127.0.0.1:32769
7c87a7626a3b   contracts-001                                    <none>                                                    RUNNING
cfc64edcb27a   el-1-geth-lighthouse                             engine-rpc: 8551/tcp -> 127.0.0.1:32771                   RUNNING
                                                                metrics: 9001/tcp -> 127.0.0.1:32770
                                                                rpc: 8545/tcp -> http://127.0.0.1:32773
                                                                tcp-discovery: 30303/tcp -> 127.0.0.1:32769
                                                                udp-discovery: 30303/udp -> 127.0.0.1:32768
                                                                ws: 8546/tcp -> 127.0.0.1:32772
570240e0efb6   grafana-001                                      dashboards: 3000/tcp -> http://127.0.0.1:32809            RUNNING
0e8099438616   panoptichain-001                                 prometheus: 9090/tcp -> http://127.0.0.1:32807            RUNNING
43199f2876dc   postgres-001                                     postgres: 5432/tcp -> postgresql://127.0.0.1:32778        RUNNING
054b50f6970d   prometheus-001                                   http: 9090/tcp -> http://127.0.0.1:32808                  RUNNING
16096bf4a778   validator-key-generation-cl-validator-keystore   <none>                                                    RUNNING
fb0dffa65446   vc-1-lighthouse-geth                             metrics: 8080/tcp -> http://127.0.0.1:32777               RUNNING
914daa8d821e   zkevm-agglayer-001                               agglayer: 4444/tcp -> http://127.0.0.1:32804              RUNNING
                                                                prometheus: 9091/tcp -> http://127.0.0.1:32802
b8d436e885af   zkevm-bridge-proxy-001                           web-ui: 80/tcp -> http://127.0.0.1:32806                  RUNNING
ca3e39d3ecdc   zkevm-bridge-service-001                         grpc: 9090/tcp -> grpc://127.0.0.1:32801                  RUNNING
                                                                rpc: 8080/tcp -> http://127.0.0.1:32803
f03e71363708   zkevm-bridge-ui-001                              web-ui: 80/tcp -> http://127.0.0.1:32805                  RUNNING
36e52ca004cb   zkevm-dac-001                                    dac: 8484/tcp -> http://127.0.0.1:32800                   RUNNING
ab498b1080ef   zkevm-node-aggregator-001                        aggregator: 50081/tcp -> grpc://127.0.0.1:32797           RUNNING
                                                                pprof: 6060/tcp -> http://127.0.0.1:32799
                                                                prometheus: 9091/tcp -> http://127.0.0.1:32798
787812a1aa57   zkevm-node-eth-tx-manager-001                    pprof: 6060/tcp -> http://127.0.0.1:32791                 RUNNING
                                                                prometheus: 9091/tcp -> http://127.0.0.1:32787
0f48b2dfff99   zkevm-node-l2-gas-pricer-001                     pprof: 6060/tcp -> http://127.0.0.1:32788                 RUNNING
                                                                prometheus: 9091/tcp -> http://127.0.0.1:32784
7b330b0fb439   zkevm-node-rpc-001                               http-rpc: 8123/tcp -> http://127.0.0.1:32793              RUNNING
                                                                pprof: 6060/tcp -> http://127.0.0.1:32794
                                                                prometheus: 9091/tcp -> http://127.0.0.1:32786
                                                                ws-rpc: 8133/tcp -> ws://127.0.0.1:32790
fa78b02faf8a   zkevm-node-sequence-sender-001                   pprof: 6060/tcp -> http://127.0.0.1:32796                 RUNNING
                                                                prometheus: 9091/tcp -> http://127.0.0.1:32795
be1d93d1a80a   zkevm-node-sequencer-001                         data-streamer: 6900/tcp -> datastream://127.0.0.1:32789   RUNNING
                                                                pprof: 6060/tcp -> http://127.0.0.1:32792
                                                                prometheus: 9091/tcp -> http://127.0.0.1:32783
                                                                rpc: 8123/tcp -> http://127.0.0.1:32785
f10a87f2eddf   zkevm-node-synchronizer-001                      pprof: 6060/tcp -> http://127.0.0.1:32782                 RUNNING
                                                                prometheus: 9091/tcp -> http://127.0.0.1:32781
5b7652d4cb8f   zkevm-prover-001                                 executor: 50071/tcp -> grpc://127.0.0.1:32779             RUNNING
                                                                hash-db: 50061/tcp -> grpc://127.0.0.1:32780

```
