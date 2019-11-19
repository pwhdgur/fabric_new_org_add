# fabric_new_org_add

■ 참조 사이트 : https://medium.com/@kctheservant/add-a-new-organization-on-existing-hyperledger-fabric-network-2c9e303955b2

< Add a New Organization on Existing Hyperledger Fabric Network >
주의사항 : 각 org별  환경변수 셋팅 후 명령어 수행함. Fabric 2.0알파버전에서는 peer chaincode upgrade => peer lifecyce 명령어로 변경됨.

1. Bring Up First Network with byfn.sh
$ cd fabric-samples/first-network
$ ./byfn.sh up

1.1 check the block height
$ docker exec -it peer0.org1.example.com peer channel getinfo -c mychannel
- Result : Blockchain info: {"height":5,"currentBlockHash":"Iv2qSTXRBbWGW5AXXjZf+JVkGvUZRTFRERIyVG6TaTQ=","previousBlockHash":"kLucQkv4uqLHi9pd+oGbEHF8Iv6XJRhgjjJlCHLxI/4="}

2. Prepare Org3 artifacts
- The configuration files to be used : org3-crypto.yaml, configtx.yaml

2.1 org3-crypto.yaml
- two peer nodes are created in Org3
- an Admin and an additional User

2.1.1 the crypto artifacts 생성
// in first-network/org3-artifacts directory
$ ../../bin/cryptogen generate --config=./org3-crypto.yaml

- crypto-config 신규 디렉토리 생성 (구조는 그림참조)
- Orderer를 First Network에서 crypto-config 로 복사해서 완전한 디렉토리구조를 만듬.(그림참조)
// in first-network/org3-artifacts directory
$ cp -r ../crypto-config/ordererOrganizations crypto-config/

2.2 configtx.yaml
- 이 파일에는 Org3 정보만 포함되어있음
- org3.json 생성
// in first-network/org3-artifacts directory
$ export FABRIC_CFG_PATH=$PWD && ../../bin/configtxgen -printOrg Org3MSP > ../channel-artifacts/org3.json

3. Update the network to include Org3 configuration

3.1 기본적인 개념
- create a new block containing the new organization configuration
- the peers of new organization can join the channel
- the whole network knows this organization is part of the network.

3.1.1 Tool: configtxlator
- The tool deals with a format called protobuf

3.1.1.1 configtxlator의 options 내용
- proto_encode: encodes JSON format to protobuf format
- proto_decode: decodes protobuf format to JSON format
- compute_update: computes the delta of two protobuf format in configuration and produces the configuration update from the first protobuf to the second

3.2 원장에서 최신 구성 블록 가져 오기

3.2.1 블록 체인 (ledger)의 블록 내부에 무엇이 있는지 검사.
- byfn.sh 실행 후 원장에는 5개의 블록이 존재함.
- block #0: Genesis block (mychannel.block)
- block #1: Anchor peer for Org1 update
- block #2: Anchor peer for Org2 update
- block #3: chaincode mycc instantiation (a, 100, b, 200)
- block #4: chaincode mycc invoke (a, b, 10)

- 처음 세 개는 구성을 위한 블록, 마지막 두 개는 체인 코드 작업을 위한 블럭임.
- 우리가 필요한 the latest configuration block, that is, #2.

3.2.2 cli termial(peer0.org1.example.com) 통해 the latest configuration block(#2) 가져오기 (그림참조) 
$ docker exec -it cli bash
$# export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem && export CHANNEL_NAME=mychannel
$# peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
- Result : 블록 # 2 수신 후 프로토 타입 바이너리 형식 인 config_block.pb 파일에 저장됨

3.3 Org3 구성 업데이트 준비작업
- config_block.pb 파일로 시작, 이 파일은 블록 안에 모든 정보를 가지고 있습니다.
- 전체 블록이 아닌 구성 업데이트와 관련된 정보만 필요.

3.3.1 config_block.pb을 JSON format 변환작업 (cli terminal : peer0.org1.example.com, 그림참조)
- configtxlator proto_decode 이용
$ docker exec -it cli bash (신규 진입시)
$# configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json

3.3.2 Org1MSP and Org2MSP 구성파일에 Org3MSP 정보를 Merge (그림참조 modified_config.json 내용)
- config.json(Org1MSP and Org2MSP) + org3.json(Org3MSP) => modified_config.json(Org1MSP, Org2MSP and Org3MSP) 생성
$ docker exec -it cli bash (신규 진입시)
$# jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ./channel-artifacts/org3.json > modified_config.json

- config.json (which contains org1 and org2)
- modified_config.json (in which org3 is included)

3.3.3 protobuf binary format 형태의 두 파일의 차이점을 찾아 업테이트하는 과정
- configtxlator 사용
- The result file : org3_update.pb.
$ docker exec -it cli bash (신규 진입시)
$# configtxlator proto_encode --input config.json --type common.Config --output config.pb
$# configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
$# configtxlator compute_update --channel_id $CHANNEL_NAME --original config.pb --updated modified_config.pb --output org3_update.pb
- $CHANNEL_NAME => 명시함, 아래참조
$# configtxlator compute_update --channel_id mychannel --original config.pb --updated modified_config.pb --output org3_update.pb

- 위 파일은 envelope 되어야 사용 가능합니다.

3.3.4 envelope 작업 (그림참조 : The org3_updated.json is wrapped into org3_updated_in_envelope.json, 전체과정 그림 : From config_block.pb (latest configuration block) to the org3_update_in_envelope.pb)
- org3_update.pb을 decode
- org3_update_in_envelope.pb로 wrapper
- org3_update_in_envelope.pb로 encode
$ docker exec -it cli bash (신규 진입시)
$# configtxlator proto_decode --input org3_update.pb --type common.ConfigUpdate | jq . > org3_update.json
$# echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat org3_update.json)'}}}' | jq . > org3_update_in_envelope.json
$# configtxlator proto_encode --input org3_update_in_envelope.json --type common.Envelope --output org3_update_in_envelope.pb

3.4 sends update to Orderer and a new block containing Org3 configuration is committed back to the ledger
- Open a new terminal (peer0.org2.example.com)

// Open a new terminal for peer0.org2
$ docker exec -e CORE_PEER_LOCALMSPID="Org2MSP" -e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp -e CORE_PEER_ADDRESS=peer0.org2.example.com:9051 -it cli bash

// perform export environment as well
$# export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem && export CHANNEL_NAME=mychannel
$# echo $ORDERER_CA && echo $CHANNEL_NAME

- peer0.org1에서 업데이트에 서명 
// cli for peer0.org1.example.com
$# peer channel signconfigtx -f org3_update_in_envelope.pb

- peer0.org2로 업데이트를 보내면 Org2의 서명도 포함
// cli for peer0.org2.example.com
$# peer channel update -f org3_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050 --tls --cafile $ORDERER_CA
- $CHANNEL_NAME => 명시함, 아래참조
$# peer channel update -f org3_update_in_envelope.pb -c mychannel -o orderer.example.com:7050 --tls --cafile $ORDERER_CA

- 업데이트가 Orderer로 전송 된 후 Orderer는 업데이트를 처리하고 새 블록을 패브릭 네트워크로 보냅니다
- org1과 org2 cli에서 블록 높이를 확인

// cli for both peer0.org1 and peer0.org2
$# peer channel getinfo -c mychannel
- Result : height=6

4. Bring up Peers of Org3 and Join mychannel
- bring up the peers of Org3 and a cli by default acting on peer0.org3.example.com.
- docker-compose-org3.yaml : two peers and one cli (Org3cli) 구성

4.1 new terminal open
// a new terminal for Org3cli
~/fabric-samples/first-network $ docker-compose -f docker-compose-org3.yaml up -d

4.2 환경 변수를 설정
// Org3cli
$# docker exec -it Org3cli bash
$# export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem && export CHANNEL_NAME=mychannel
$# echo $ORDERER_CA && echo $CHANNEL_NAME

4.3 Org3의 peers가 mychannel에 합류
4.3.1 genesis block (block #0) 필요함
// Org3cli for peer0.org1.example.com
$# peer channel fetch 0 mychannel.block -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
- Result : readBlock -> INFO 002 Received block: 0

4.3.2 mychannel.block 을 Org3cli에 복사
// my localhost (특정 디렉토리 생성 후 작업 진행이 좋음)
$ docker cp cli:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block .
$ docker cp mychannel.block Org3cli:/opt/gopath/src/github.com/hyperledger/fabric/peer/

4.3.3 join the peers of Org3
// Org3cli
$ peer channel join -b mychannel.block
- Result : Successfully submitted proposal to join channel

4.3.3.1 Optionally we can use the same Org3cli to join peer1.org3 (by specifying the environment variables.)
// Org3cli
$# CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer1.org3.example.com/tls/ca.crt CORE_PEER_ADDRESS=peer1.org3.example.com:12051 peer channel join -b mychannel.block

4.3.4 peers of Org3 동기화 확인
// Org3cli
$# peer channel getinfo -c mychannel
- height = 6 이면 peer0.org3 원장이 업데이트 된것을 의미함.

5. Chaincode operations after Org3 is part of the network
- Org1, Org2, Org3 : install a new version of chaincode (2.0) 
- focus on peer0 of each organization.

5.1 chaincode (2.0) install
// cli for peer0.org1.example.com
$# peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go/

// cli for peer0.org2.example.com
$# peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go/

// Org3cli
$# peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go/

5.2 chaincode (2.0) upgrade
- 시간이 좀 걸림
// org1 cli for peer0.org1.example.com
$# peer chaincode upgrade -o orderer.example.com:7050 —-tls $CORE_PEER_TLS_ENABLED —-cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -v 2.0 -c '{"Args":["init","a","90","b","210"]}' -P "OR ('Org1MSP.peer','Org2MSP.peer','Org3MSP.peer')"

5.3 query 확인
// Org3cli
$# peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
- the result : 90

// Org3cli
$# peer chaincode invoke -o orderer.example.com:7050 --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -c '{"Args":["invoke","a","b","10"]}'

// Org1 cli for peer0.org2.example.com
# peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
- the result : 80

6. 요약
- Hyperledger Fabric과 같은 권한이 부여 된 블록 체인의 권한 특성으로 인해 하나 이상의 조직을 추가하는 것은 결코 쉬운 일이 아닙니다. 주의를 요하는 작업입니다.

7. Clean Up
$ cd first-network
$ ./byfn.sh down
$ docker rm $(docker ps -aq)
$ docker rmi $(docker images dev-* -q)
$ docker network prune
