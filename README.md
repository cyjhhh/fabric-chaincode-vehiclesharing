# fabric-chaincode-vehiclesharing
Follow the instructions in https://www.ibm.com/developerworks/cn/cloud/library/cl-lo-hyperledger-fabric-practice-analysis2/index.html

此文主要是跟随IBM的教程实现vehiclesharing教程，并解决教程中的一些bug
下载与部署部分跳过，直接下载faric-v1.4即自带vehiclesharing示例(上传的是1.4版本的vehiclesharing的整个文件夹)

## 启动并运行网络
~~~
cd ~/fabric/fabric-samples/first-network
/byfn.sh generate
/byfn.sh up -c mychannel -s couchdb
~~~

## 查看docker
此时已经启动了4(peer)+4(couchdb)+1(cli)+1(orderer)=10个容器
docker-compose-couch.yaml 中定义了 4 个 CouchDB Container 并分别与 4 个 Peer 对应
~~~
d35fa6a91062        hyperledger/fabric-tools:latest                                                                        "/bin/bash"              2 minutes ago        Up 2 minutes                                                     cli
aebff672f1fb        hyperledger/fabric-peer:latest                                                                         "peer node start"        2 minutes ago        Up 2 minutes        0.0.0.0:10051->10051/tcp                     peer1.org2.example.com
19b43e706ec5        hyperledger/fabric-peer:latest                                                                         "peer node start"        2 minutes ago        Up 2 minutes        0.0.0.0:9051->9051/tcp                       peer0.org2.example.com
0c2f341f876a        hyperledger/fabric-peer:latest                                                                         "peer node start"        2 minutes ago        Up 2 minutes        0.0.0.0:7051->7051/tcp                       peer0.org1.example.com
7e2082e705a7        hyperledger/fabric-peer:latest                                                                         "peer node start"        2 minutes ago        Up 2 minutes        0.0.0.0:8051->8051/tcp                       peer1.org1.example.com
6cbbfe06253c        hyperledger/fabric-couchdb                                                                             "tini -- /docker-ent…"   2 minutes ago        Up 2 minutes        4369/tcp, 9100/tcp, 0.0.0.0:8984->5984/tcp   couchdb3
3e15689d61ee        hyperledger/fabric-couchdb                                                                             "tini -- /docker-ent…"   2 minutes ago        Up 2 minutes        4369/tcp, 9100/tcp, 0.0.0.0:7984->5984/tcp   couchdb2
5bfdc8a7d159        hyperledger/fabric-orderer:latest                                                                      "orderer"                2 minutes ago        Up 2 minutes        0.0.0.0:7050->7050/tcp                       orderer.example.com
96e5241c1bee        hyperledger/fabric-couchdb                                                                             "tini -- /docker-ent…"   2 minutes ago        Up 2 minutes        4369/tcp, 9100/tcp, 0.0.0.0:6984->5984/tcp   couchdb1
6dfb51d667fa        hyperledger/fabric-couchdb                                                                             "tini -- /docker-ent…"   2 minutes ago        Up 2 minutes        4369/tcp, 9100/tcp, 0.0.0.0:5984->5984/tcp   couchdb0
~~~

## 登录cli容器
个人理解cli容器应该是一个管理工具之类的，进入cli的bash可以执行对其他peer的部署
~~~
docker exec -it cli bash
~~~

## 设置cli的环境变量
~~~
export CC_SRC_PATH="github.com/chaincode/vehiclesharing"   # chaincode的路径
export VERSION="1.0"                                       # chiancode的版本，后续如果在host(非cli)中更新了chaincode，需要修改VERSION
export CC_NAME="vehiclesharing"                            # chaincode的名字
source ./scripts/setparas.sh                               # 执行一些其他的环境变量设置，包括LANGUAGE=golang
~~~

## 分别对每个节点安装chaincode
~~~
source ./scripts/setparas.sh peerenv 0 1  # peerenv对应在setparas.sh表示设置对应节点peer0.org1的环境变量
peer chaincode install -n $CC_NAME -v $VERSION -l $LANGUAGE -p $CC_SRC_PATH  # chaincode是peer的字命令，install是chaincode的子命令
~~~
原教程中，下一步执行的是对peer0.org2的环境变量的设置，对应的命令是source ./scripts/setparas.sh peerenv 0 2；但是由于所有节点的容器都开在一个虚拟机中，所有不同peer开启的端口是不同的，而在setparas.sh全部对应的是7051，如果按官方教程执行，则会出现以下报错
~~~
Error: error getting endorser client for install: endorser client failed to connect to peer0.org2.example.com:7051: failed to create new connection: connection error: desc = "transport: error while dialing: dial tcp 172.18.0.9:7051: connect: connection refused"
~~~
所以手动修改setparas.sh中要设置的环境变量
~~~
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051  # 注意这里的端口9051是在host端bash使用docker ps查看的对应的peer0.org2的端口
peer chaincode install -n $CC_NAME -v $VERSION -l $LANGUAGE -p $CC_SRC_PATH
~~~
剩下的两个peer1.org1和peer1.org2的操作同上
~~~
export CORE_PEER_ADDRESS=peer1.org1.example.com:8051
peer chaincode install -n $CC_NAME -v $VERSION -l $LANGUAGE -p $CC_SRC_PATH
export CORE_PEER_ADDRESS=peer1.org2.example.com:10051
peer chaincode install -n $CC_NAME -v $VERSION -l $LANGUAGE -p $CC_SRC_PATH
~~~

## 初始化chaincode
~~~
peer chaincode instantiate -o $ORDERER_ADDRESS --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n $CC_NAME -l $LANGUAGE -v $VERSION -c '{"Args":[""]}' -P "$DEFAULT_POLICY"
~~~

## 增改查
进行invoke操作时命令 "peer chaincode invoke"会生成并背书 Transaction，并提交给 Channel Network，所以需要连接两个节点进行通信，在原教程中使用命令为
~~~
source ./scripts/setparas.sh peerconn 0 1 0 2
~~~
查看对应的脚本，发现想要连接的两个节点peer1.org1和peer1.org2在脚本中的地址都为7051，与虚拟机中开启的docker容器的实际地址不符，于是把命令提出来单独执行
~~~
export PEER_CONN_PARMS="--peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt"
~~~

添加vehicle
~~~
peer chaincode invoke -o $ORDERER_ADDRESS --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n $CC_NAME $PEER_CONN_PARMS -c '{"Args":["createVehicle", "C123", "FBC"]}'
~~~
查询vehicle
~~~
peer chaincode query -C $CHANNEL_NAME -n $CC_NAME -c '{"Args":["findVehicle","C123"]}'
~~~
更新vehicle价格
~~~
peer chaincode invoke -o $ORDERER_ADDRESS --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n $CC_NAME $PEER_CONN_PARMS -c '{"Args":["updateVehiclePrice", "C123", "123.55"]}'
~~~

## 在CouchDB中查看数据
~~~
 http://<HOST>:5984/_utils/
~~~
其中HOST为虚拟机的ip地址，5984为couchdb0的端口，更改5984为其他couchdb的端口即可访问其他couchdb的数据
