# 超级账本 Hyperledger Fabric 多机部署教程

本文将介绍如何通过使用超级账本Hyperledger Fabric的可执行程序，在多机环境下部署fabric区块链网络及链码。此外，对于Fabric整体的网络架构组成不作过多描述。

1. 环境介绍
2. 环境准备
3. 下载fabric可执行程序
4. 生成组织证书文件
5. 生成创世块、通道交易文件
6. 部署orderer
7. 部署peer
8. 部署peer-cli
9. 创建通道
10. 测试链码

## 1.环境介绍

本文以Hyperledger Fabric 2.2.0 版本为例进行多机部署

需要使用到的Fabric可执行程序：
- cryptogen 用于生成区块链网络中相应用户的相关证书文件

- configtxgen 用于生成区块链系统链码的创世区块、新建通道的配置文件、以及组织中锚节点的配置文件

- orderer  共识节点。为交易排序，并生成区块

- peer 共识节点。为交易背书，并记录区块信息 

网络环境如下：

- 7台Centos 7 系统的虚拟机，硬件配置为：1核，1G内存

- 3个 orderer节点，4个 peer节点，使用etcdRaft共识算法

      192.168.134.145 ~ 147 部署 orgnetorderer组织的orderer0 ~ orderer2 节点
    
      192.168.134.148 ~ 149 部署 org1组织 的 peer0、peer1节点

      192.168.134.150 ~ 151 部署 org2组织 的 peer0、peer1节点

## 2.环境准备
### 安装Docker、Docker-compose
执行如下shell命令：

    curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun

    systemctl start docker

    curl -L https://get.daocloud.io/docker/compose/releases/download/1.27.3/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

    chmod +x /usr/local/bin/docker-compose

### 配置虚拟机Hosts
由于 fabric 网络启动相关的配置文件中，与网络地址相关的信息都是填写的域名，所以需要在 hosts 文件中配置域名与 ip 的对应关系。
    
    192.168.134.145 orderer0.orgnetorderer
    192.168.134.146 orderer1.orgnetorderer
    192.168.134.147 orderer2.orgnetorderer
    192.168.134.148 peer0.org1
    192.168.134.149 peer1.org1
    192.168.134.150 peer0.org2
    192.168.134.151 peer1.org2

## 3.下载fabric可执行程序
### 获取Hyperledger Fabric 2.2.0执行程序
获取Hyperledger Fabric 2.2.0相关执行程序，通过github下载：
https://github.com/hyperledger/fabric/releases/download/v2.2.0/hyperledger-fabric-linux-amd64-2.2.0.tar.gz

对压缩包进行解压后可以得到以下内容

    --bin
      configtxgen
      configtxlator
      cryptogen
      discover
      idemixgen
      orderer
      peer
    --config
      configtx.yaml
      core.yaml (peer程序启动配置文件)
      orderer.yaml (orderer程序启动配置文件)

为了便于使用，我们将bin/目录下的相关执行程序放置到各个虚拟机的环境变量中

- 192.168.134.145 ~ 147 orderer 节点 /usr/local/bin 目录下放置 orderer 可执行程序

- 192.168.134.148 ~ 151 peer 节点 /usr/local/bin 目录下放置 peer 可执行程序

由于生成组织证书文件、创世块及通道相关配置文件需要用到 cryptogen 和 configtxgen 可执行程序，相关的操作本文选择直接在 orderer0 节点操作，所以在 orderer0 节点的 /usr/local/bin 目录下额外放置 cryptogen 及 configtxgen程序，如已有相关内容，可忽略。

在7台虚拟机中创建目录/etc/hyperledger/fabric，以下的命令全部在该目录下执行，并且需要设置 fabric 网络执行的环境变量: 
$FABRIC_CFG_PATH=/etc/hyperledger/fabric

## 4.生成组织证书文件
进入到orderer0节点的虚拟机中

创建组织证书文件的配置crypto-config.yaml。文件内容如下：

```yaml
OrdererOrgs:
  - Name: orgnetorderer
    Domain: orgnetorderer
    EnableNodeOUs: true
    Specs:
      - Hostname: orderer0
      - Hostname: orderer1
      - Hostname: orderer2
PeerOrgs:
  - Name: org1
    Domain: org1
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
  - Name: org2
    Domain: org2
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
```
使用 cryptogen 工具，生成orgnetorderer组织，org1组织，org2组织的密钥和证书文件
    
    cryptogen generate –config=./crypto-config.yaml –output ./crypto-config
生成的文件目录结构如下：

    crypto-config/  
    ├── ordererOrganizations  
    │   └── orgnetorderer  
    │       ├── ca  
    │       │   ├── ca.orgnetorderer-cert.pem  
    │       │   └── priv_sk  
    │       ├── msp  
    │       │   ├── admincerts  
    │       │   ├── cacerts  
    │       │   │   └── ca.orgnetorderer-cert.pem  
    │       │   ├── config.yaml  
    │       │   └── tlscacerts  
    │       │       └── tlsca.orgnetorderer-cert.pem  
    │       ├── orderers  
    │       │   ├── orderer0.orgnetorderer  
    │       │   │   ├── msp  
    │       │   │   │   ├── admincerts  
    │       │   │   │   ├── cacerts  
    │       │   │   │   │   └── ca.orgnetorderer-cert.pem  
    │       │   │   │   ├── config.yaml  
    │       │   │   │   ├── keystore  
    │       │   │   │   │   └── priv_sk  
    │       │   │   │   ├── signcerts  
    │       │   │   │   │   └── orderer0.orgnetorderer-cert.pem  
    │       │   │   │   └── tlscacerts  
    │       │   │   │       └── tlsca.orgnetorderer-cert.pem  
    │       │   │   └── tls  
    │       │   │       ├── ca.crt  
    │       │   │       ├── server.crt  
    │       │   │       └── server.key  
    │       │   ├── orderer1.orgnetorderer  
    │       │   │   ├── msp  
    │       │   │   │   ├── admincerts  
    │       │   │   │   ├── cacerts  
    │       │   │   │   │   └── ca.orgnetorderer-cert.pem  
    │       │   │   │   ├── config.yaml  
    │       │   │   │   ├── keystore  
    │       │   │   │   │   └── priv_sk  
    │       │   │   │   ├── signcerts  
    │       │   │   │   │   └── orderer1.orgnetorderer-cert.pem  
    │       │   │   │   └── tlscacerts  
    │       │   │   │       └── tlsca.orgnetorderer-cert.pem  
    │       │   │   └── tls  
    │       │   │       ├── ca.crt  
    │       │   │       ├── server.crt  
    │       │   │       └── server.key  
    │       │   └── orderer2.orgnetorderer  
    │       │       ├── msp  
    │       │       │   ├── admincerts  
    │       │       │   ├── cacerts  
    │       │       │   │   └── ca.orgnetorderer-cert.pem  
    │       │       │   ├── config.yaml  
    │       │       │   ├── keystore  
    │       │       │   │   └── priv_sk  
    │       │       │   ├── signcerts  
    │       │       │   │   └── orderer2.orgnetorderer-cert.pem  
    │       │       │   └── tlscacerts  
    │       │       │       └── tlsca.orgnetorderer-cert.pem  
    │       │       └── tls  
    │       │           ├── ca.crt  
    │       │           ├── server.crt  
    │       │           └── server.key  
    │       ├── tlsca  
    │       │   ├── priv_sk  
    │       │   └── tlsca.orgnetorderer-cert.pem  
    │       └── users  
    │           └── Admin@orgnetorderer  
    │               ├── msp  
    │               │   ├── admincerts  
    │               │   ├── cacerts  
    │               │   │   └── ca.orgnetorderer-cert.pem  
    │               │   ├── config.yaml  
    │               │   ├── keystore  
    │               │   │   └── priv_sk  
    │               │   ├── signcerts  
    │               │   │   └── Admin@orgnetorderer-cert.pem  
    │               │   └── tlscacerts  
    │               │       └── tlsca.orgnetorderer-cert.pem  
    │               └── tls  
    │                   ├── ca.crt  
    │                   ├── client.crt  
    │                   └── client.key  
    └── peerOrganizations  
        ├── org1  
        │   ├── ca  
        │   │   ├── ca.org1-cert.pem  
        │   │   └── priv_sk  
        │   ├── msp  
        │   │   ├── admincerts  
        │   │   ├── cacerts  
        │   │   │   └── ca.org1-cert.pem  
        │   │   ├── config.yaml  
        │   │   └── tlscacerts  
        │   │       └── tlsca.org1-cert.pem  
        │   ├── peers  
        │   │   ├── peer0.org1  
        │   │   │   ├── msp  
        │   │   │   │   ├── admincerts  
        │   │   │   │   ├── cacerts  
        │   │   │   │   │   └── ca.org1-cert.pem  
        │   │   │   │   ├── config.yaml  
        │   │   │   │   ├── keystore  
        │   │   │   │   │   └── priv_sk  
        │   │   │   │   ├── signcerts  
        │   │   │   │   │   └── peer0.org1-cert.pem  
        │   │   │   │   └── tlscacerts  
        │   │   │   │       └── tlsca.org1-cert.pem  
        │   │   │   └── tls  
        │   │   │       ├── ca.crt  
        │   │   │       ├── server.crt  
        │   │   │       └── server.key  
        │   │   └── peer1.org1  
        │   │       ├── msp  
        │   │       │   ├── admincerts  
        │   │       │   ├── cacerts  
        │   │       │   │   └── ca.org1-cert.pem  
        │   │       │   ├── config.yaml  
        │   │       │   ├── keystore  
        │   │       │   │   └── priv_sk  
        │   │       │   ├── signcerts  
        │   │       │   │   └── peer1.org1-cert.pem  
        │   │       │   └── tlscacerts  
        │   │       │       └── tlsca.org1-cert.pem  
        │   │       └── tls  
        │   │           ├── ca.crt  
        │   │           ├── server.crt  
        │   │           └── server.key  
        │   ├── tlsca  
        │   │   ├── priv_sk  
        │   │   └── tlsca.org1-cert.pem  
        │   └── users  
        │       ├── Admin@org1  
        │       │   ├── msp  
        │       │   │   ├── admincerts  
        │       │   │   ├── cacerts  
        │       │   │   │   └── ca.org1-cert.pem  
        │       │   │   ├── config.yaml  
        │       │   │   ├── keystore  
        │       │   │   │   └── priv_sk  
        │       │   │   ├── signcerts  
        │       │   │   │   └── Admin@org1-cert.pem  
        │       │   │   └── tlscacerts  
        │       │   │       └── tlsca.org1-cert.pem  
        │       │   └── tls  
        │       │       ├── ca.crt  
        │       │       ├── client.crt  
        │       │       └── client.key  
        │       └── User1@org1  
        │           ├── msp  
        │           │   ├── admincerts  
        │           │   ├── cacerts  
        │           │   │   └── ca.org1-cert.pem  
        │           │   ├── config.yaml  
        │           │   ├── keystore  
        │           │   │   └── priv_sk  
        │           │   ├── signcerts  
        │           │   │   └── User1@org1-cert.pem  
        │           │   └── tlscacerts  
        │           │       └── tlsca.org1-cert.pem  
        │           └── tls  
        │               ├── ca.crt  
        │               ├── client.crt  
        │               └── client.key  
        └── org2  
            ├── ca  
            │   ├── ca.org2-cert.pem  
            │   └── priv_sk  
            ├── msp  
            │   ├── admincerts  
            │   ├── cacerts  
            │   │   └── ca.org2-cert.pem  
            │   ├── config.yaml  
            │   └── tlscacerts  
            │       └── tlsca.org2-cert.pem  
            ├── peers  
            │   ├── peer0.org2  
            │   │   ├── msp  
            │   │   │   ├── admincerts  
            │   │   │   ├── cacerts  
            │   │   │   │   └── ca.org2-cert.pem  
            │   │   │   ├── config.yaml  
            │   │   │   ├── keystore  
            │   │   │   │   └── priv_sk  
            │   │   │   ├── signcerts  
            │   │   │   │   └── peer0.org2-cert.pem  
            │   │   │   └── tlscacerts  
            │   │   │       └── tlsca.org2-cert.pem  
            │   │   └── tls  
            │   │       ├── ca.crt  
            │   │       ├── server.crt  
            │   │       └── server.key  
            │   └── peer1.org2  
            │       ├── msp  
            │       │   ├── admincerts  
            │       │   ├── cacerts  
            │       │   │   └── ca.org2-cert.pem  
            │       │   ├── config.yaml  
            │       │   ├── keystore  
            │       │   │   └── priv_sk  
            │       │   ├── signcerts  
            │       │   │   └── peer1.org2-cert.pem  
            │       │   └── tlscacerts  
            │       │       └── tlsca.org2-cert.pem  
            │       └── tls  
            │           ├── ca.crt  
            │           ├── server.crt  
            │           └── server.key  
            ├── tlsca  
            │   ├── priv_sk  
            │   └── tlsca.org2-cert.pem  
            └── users  
                ├── Admin@org2  
                │   ├── msp  
                │   │   ├── admincerts  
                │   │   ├── cacerts  
                │   │   │   └── ca.org2-cert.pem  
                │   │   ├── config.yaml  
                │   │   ├── keystore  
                │   │   │   └── priv_sk  
                │   │   ├── signcerts  
                │   │   │   └── Admin@org2-cert.pem  
                │   │   └── tlscacerts  
                │   │       └── tlsca.org2-cert.pem  
                │   └── tls  
                │       ├── ca.crt  
                │       ├── client.crt  
                │       └── client.key  
                └── User1@org2  
                    ├── msp  
                    │   ├── admincerts  
                    │   ├── cacerts  
                    │   │   └── ca.org2-cert.pem  
                    │   ├── config.yaml  
                    │   ├── keystore  
                    │   │   └── priv_sk  
                    │   ├── signcerts  
                    │   │   └── User1@org2-cert.pem  
                    │   └── tlscacerts  
                    │       └── tlsca.org2-cert.pem  
                    └── tls  
                        ├── ca.crt  
                        ├── client.crt  
                        └── client.key  


## 5.生成创世块、通道交易文件
  配置configtx.yaml文件，通过使用configtxgen工具根据配置生成创世块文件、新建通道所需的交易信息。configtx.yaml如下所示：

```yaml
Organizations:
  - &org1
    Name: org1
    ID: org1MSP
    MSPDir: ../../org1/peerOrganizations/org1/msp

    Policies:
      Readers:
        Type: Signature
        Rule: "OR('org1MSP.admin','org1MSP.peer','org1MSP.client')"
      Writers:
        Type: Signature
        Rule: "OR('org1MSP.admin','org1MSP.client')"
      Admins:
        Type: Signature
        Rule: "OR('org1MSP.admin')"
      Endorsement:
        Type: Signature
        Rule: "OR('org1MSP.peer')"

    AnchorPeers:
      - Host: peer0.org1
        Port: 7051

  - &org2
    Name: org2
    ID: org2MSP
    MSPDir: ../../org2/peerOrganizations/org2/msp

    Policies:
      Readers:
        Type: Signature
        Rule: "OR('org2MSP.admin','org2MSP.peer','org2MSP.client')"
      Writers:
        Type: Signature
        Rule: "OR('org2MSP.admin','org2MSP.client')"
      Admins:
        Type: Signature
        Rule: "OR('org2MSP.admin')"
      Endorsement:
        Type: Signature
        Rule: "OR('org2MSP.peer')"

    AnchorPeers:
      - Host: peer0.org2
        Port: 7051

Capabilities:
  Channel: &ChannelCapabilities
    V2_0: true

  Orderer: &OrdererCapabilities
    V2_0: true

  Application: &ApplicationCapabilities
    V2_0: true

Application: &ApplicationDefaults
  ACLs: &ACLsDefault
    _lifecycle/CheckCommitReadiness: /Channel/Application/Writers
    _lifecycle/CommitChaincodeDefinition: /Channel/Application/Writers
    _lifecycle/QueryChaincodeDefinition: /Channel/Application/Readers
    _lifecycle/QueryChaincodeDefinitions: /Channel/Application/Readers
    lscc/ChaincodeExists: /Channel/Application/Readers
    lscc/GetDeploymentSpec: /Channel/Application/Readers
    lscc/GetChaincodeData: /Channel/Application/Readers
    lscc/GetInstantiatedChaincodes: /Channel/Application/Readers
    qscc/GetChainInfo: /Channel/Application/Readers
    qscc/GetBlockByNumber: /Channel/Application/Readers
    qscc/GetBlockByHash: /Channel/Application/Readers
    qscc/GetTransactionByID: /Channel/Application/Readers
    qscc/GetBlockByTxID: /Channel/Application/Readers
    cscc/GetConfigBlock: /Channel/Application/Readers
    cscc/GetConfigTree: /Channel/Application/Readers
    cscc/SimulateConfigTreeUpdate: /Channel/Application/Readers
    peer/Propose: /Channel/Application/Writers
    peer/ChaincodeToChaincode: /Channel/Application/Readers
    event/Block: /Channel/Application/Readers
    event/FilteredBlock: /Channel/Application/Readers

  Organizations:

  Policies: &ApplicationDefaultPolicies
    Readers:
      Type: ImplicitMeta
      Rule: "ANY Readers"
    Writers:
      Type: ImplicitMeta
      Rule: "ANY Writers"
    Admins:
      Type: ImplicitMeta
      Rule: "MAJORITY Admins"
    LifecycleEndorsement:
      Type: ImplicitMeta
      Rule: "MAJORITY Endorsement"
    Endorsement:
      Type: ImplicitMeta
      Rule: "MAJORITY Endorsement"

  Capabilities:
    <<: *ApplicationCapabilities

Orderer: &OrdererDefaults

  OrdererType: etcdraft

  BatchTimeout: 2s

  BatchSize:
    MaxMessageCount: 10
    AbsoluteMaxBytes: 99 MB
    PreferredMaxBytes: 512 KB

  MaxChannels: 0

  Organizations:

  Policies:
    Readers:
      Type: ImplicitMeta
      Rule: "ANY Readers"
    Writers:
      Type: ImplicitMeta
      Rule: "ANY Writers"
    Admins:
      Type: ImplicitMeta
      Rule: "MAJORITY Admins"
    BlockValidation:
      Type: ImplicitMeta
      Rule: "ANY Writers"
  Capabilities:
    <<: *OrdererCapabilities

Channel: &ChannelDefaults
  Policies:
    Readers:
      Type: ImplicitMeta
      Rule: "ANY Readers"
    Writers:
      Type: ImplicitMeta
      Rule: "ANY Writers"
    Admins:
      Type: ImplicitMeta
      Rule: "MAJORITY Admins"

  Capabilities:
    <<: *ChannelCapabilities

Profiles:

  Channel:
    Consortium: SampleConsortium
    <<: *ChannelDefaults
    Application:
      <<: *ApplicationDefaults
      Organizations:
        - *org1
        - *org2
      Capabilities:
        <<: *ApplicationCapabilities

  Genesis:
    <<: *ChannelDefaults
    Capabilities:
      <<: *ChannelCapabilities
    Orderer:
      <<: *OrdererDefaults
      OrdererType: etcdraft
      Addresses:
      EtcdRaft:
        Consenters:
        Options:
          TickInterval: 500ms
          ElectionTick: 10
          HeartbeatTick: 1
          MaxInflightBlocks: 5
          SnapshotIntervalSize: 20 MB
      Organizations:
      Capabilities:
        <<: *OrdererCapabilities
    Application:
      <<: *ApplicationDefaults
      Organizations:
    Consortiums:
      SampleConsortium:
        Organizations:
          - *org1
          - *org2

```

    //生成创世块文件
    configtxgen -profile Genesis -outputBlock genesis.block 
    //生成新建通道所需的交易信息
    configtxgen -profile Channel -outputCreateChannelTx testchannel.tx -channelID testchannel
    //生成org1组织锚节点更新信息
    configtxgen –profile Channel –outputAnchorPeersUpdate ./org1MSPanchors.tx –channelID channel –asOrg org1
    //生成org2组织锚节点更新信息
    configtxgen –profile Channel –outputAnchorPeersUpdate ./org2MSPanchors.tx –channelID channel –asOrg org2

通过执行上述命令，最终可以得到如下文件：

    genesis.block、 testchannel.tx、 org1MSPanchors.tx、 org2MSPanchors.tx

## 6.部署orderer
每一个orderer节点启动前需要配置自己对应的MSP证书以及TLS证书，此处我们以orderer0节点为例：

    cp -r ./crypto-config/ordererOrganizations/orgnetorderer/orderers/orderer0.orgnetorderer/msp ./
    cp -r ./crypto-config/ordererOrganizations/orgnetorderer/orderers/orderer0.orgnetorderer/tls ./
另外将之前通过github下载获得的orderer.yaml启动配置项，及生成的证书、创世块文件一同放置到目录当中：

    ./crypto-config
    ./msp
    ./tls
    ./orderer.yaml
    ./genesis.block

此外，还需要对启动配置项orderer.yaml进行调整，内容主要包括：
    
    General.ListenAddress: 0.0.0.0
    General.TLS.Enabled: true
    General.Cluster.ClientCertificate: tls/server.crt
    General.Cluster.ClientPrivateKey: tls/server.key
    General.LocalMSPID: orgnetordererMSP
其他配置可根据需要进行相关调整

启动orderer节点：

    orderer start

## 7.部署peer
每一个peer节点启动前需要配置自己对应的MSP证书以及TLS证书，此处我们以org1组织的peer0节点为例：

    cp -r ./crypto-config/peerOrganizations/org1/peers/peer0.org1/msp ./
    cp -r ./crypto-config/peerOrganizations/org1/peers/peer0.org1/tls ./
另外将之前通过github下载获得的core.yaml启动配置项，及生成的证书、通道交易信息和锚节点更新信息一同放置到目录当中：

    ./crypto-config
    ./msp
    ./tls
    ./core.yaml
    ./testchannel.tx
    ./org1MSPanchors.tx

此外，还需要对启动配置项core.yaml进行调整，内容主要包括：
    
    peer.localMspId: org1MSP
其他配置可根据需要进行相关调整

启动peer节点：

    peer node start

## 8.部署peer-cli
peer-cli是我们为了方便操作peer点抽象出的一个组织中的概念。它本质上也是使用peer可执行程序对Fabric网络进行操作。只不过与peer节点有所不同的是，它使用的msp证书是组织下用户的证书。因此，对于peer-cli的部署我们可以直接在peer节点的虚拟机中打开新的会话，进行操作。此处仍以org1组织的peer0节点为例：

    cd /etc/hyperledger/fabric
    mkdir peer-cli
    cp ./core.yaml peer-cli/
    cp -r ./crypto-config/peerOrganizations/org1/users/Admin@org1/msp ./
    cp -r ./crypto-config/peerOrganizations/org1/users/Admin@org1/tls ./
    cd peer-cli

## 9.创建通道
此处以org1组织为例，peer0节点的peer-cli中执行如下命令：

    //进入到peer-cli操作目录
    cd /etc/hyperledger/fabric/peer-cli

    //根据通道交易文件testchannel.tx创建通道文件testchannel.block
    peer channel create -o orderer0.orgnetorderer:7050 --tls true --cafile /etc/hyperledger/fabric/crypto-config/ordererOrganizations/orgnetorderer/tlsca/tlsca.orgnetorderer-cert.pem -c testchannel -f /etc/hyperledger/fabric/testchannel.tx 

    //加入通道
    peer channel join -b testchannel.tx

    //查看通道
    peer channel list
    2020-09-27 10:46:50.185 CST [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
    Channels peers has joined: 
    testchannel

    //更新锚节点配置信息
    peer channel update -o orderer0.orgnetorderer:7050 --tls true --cafile /etc/hyperledger/fabric/crypto-config/ordererOrganizations/orgnetorderer/tlsca/tlsca.orgnetorderer-cert.pem -c testchannel -f /etc/hyperledger/fabric/org1MSPanchors.tx

上述命令同样也在其他组织中peer0节点执行一遍即可。

# 总结
至此，我们已经成功的部署Fabric以及通道，链码的安装调用以及API、SDK的调用可参考官方相关文档
https://hyperledger-fabric.readthedocs.io/en/v2.2.0/whatis.html#

SDK可参考包括github上hyperledger/fabric-sdk-java以及hyperledger/fabric-gateway-java
https://github.com/hyperledger/fabric-sdk-java

https://github.com/hyperledger/fabric-gateway-java

以上就是如何在多机环境下部署Hyperledger Fabric 2.2.0 的所有内容了。