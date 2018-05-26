## 준비
### Clone repo
  ```sh
    git clone https://github.com/9bow/fabric-dev-net.git
    cd fabric-dev-net
  ```

### 환경 변수 설정
  ```sh
    export FABRIC_CFG_PATH=$PWD
    export CHANNEL_NAME=mychan
    export IMAGE_TAG=latest
  ```


## 설정 및 필요 파일 생성
### 인증서 생성 (`./bin/cryptogen`)
* `crypto-config.yaml` 설정 확인 및 수정
* 인증서 및 키 파일 생성
  ```sh
    ./bin/cryptogen generate --config=./crypto-config.yaml
    # 실행 결과
    # myorg.mycompany.com
  ```

### Genesis Block 등 생성 (`./bin/configtxgen`)
* `configtx.yaml` 설정 확인 및 수정
*  Genesis Block 생성
  ```sh
    mkdir channel-artifacts
    ./bin/configtxgen -profile MyCompanyOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
    # 실행 결과
    # 2018-05-26 13:00:33.236 KST [common/tools/configtxgen] main -> INFO 001 Loading configuration
    # 2018-05-26 13:00:33.243 KST [msp] getMspConfig -> INFO 002 Loading NodeOUs
    # 2018-05-26 13:00:33.244 KST [common/tools/configtxgen] doOutputBlock -> INFO 003 Generating genesis block
    # 2018-05-26 13:00:33.244 KST [common/tools/configtxgen] doOutputBlock -> INFO 004 Writing genesis block
  ```
* `channel.tx` 생성
  ```sh
    ./bin/configtxgen -profile MyOrgChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME
    # 실행 결과
    # 2018-05-26 13:01:43.223 KST [common/tools/configtxgen] main -> INFO 001 Loading configuration
    # 2018-05-26 13:01:43.229 KST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
    # 2018-05-26 13:01:43.229 KST [msp] getMspConfig -> INFO 003 Loading NodeOUs
    # 2018-05-26 13:01:43.246 KST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 004 Writing new channel tx
  ```
* AnchorPeer 정의 (Org별로 하나(또는 그 이상))
  ```sh
    ./bin/configtxgen -profile MyOrgChannel -outputAnchorPeersUpdate ./channel-artifacts/MyOrgMSPanchors.tx -channelID $CHANNEL_NAME -asOrg MyOrgMSP
    # 실행 결과
    # 2018-05-26 13:02:17.514 KST [common/tools/configtxgen] main -> INFO 001 Loading configuration
    # 2018-05-26 13:02:17.521 KST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 002 Generating anchor peer update
    # 2018-05-26 13:02:17.521 KST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 003 Writing anchor peer update
  ```

## Fabric Network 실행
### docker base 설정 파일 수정 및 실행
* `base/docker-compose-base.yaml` 설정 확인 및 수정
* `base/peer-base.yaml` 설정 확인 및 수정
* `docker-compose-cli.yaml` 설정 확인 및 수정
* Docker Container 실행 및 cli Container에 진입
  ```sh
    CHANNEL_NAME=$CHANNEL_NAME docker-compose -f docker-compose-cli.yaml up -d
    # 실행 결과
    # Creating network "mycompany_mynet" with the default driver
    # Creating volume "mycompany_orderer.mycompany.com" with default driver
    # Creating volume "mycompany_peer0.myorg.mycompany.com" with default driver
    # Creating volume "mycompany_peer1.myorg.mycompany.com" with default driver
    # Pulling orderer.mycompany.com (hyperledger/fabric-orderer:latest)...
    # ...
    # Creating peer1.myorg.mycompany.com ... done
    # Creating peer0.myorg.mycompany.com ... done
    # Creating orderer.mycompany.com     ... done
    # Creating cli                       ... done
    docker exec -it cli bash
  ```

### 기본 Channel 설정
* 기본 Channel 생성
  ```sh
    export CHANNEL_NAME=mychan
    peer channel create -o orderer.mycompany.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/mycompany.com/orderers/orderer.mycompany.com/msp/tlscacerts/tlsca.mycompany.com-cert.pem
    # 실행 결과
    # 2018-05-26 04:10:06.487 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
    # 2018-05-26 04:10:06.515 UTC [channelCmd] InitCmdFactory -> INFO 002 Endorser and orderer connections initialized
    # 2018-05-26 04:10:06.719 UTC [main] main -> INFO 003 Exiting.....
  ```
* 생성 Channel 조인
  ```sh
    peer channel join -b $CHANNEL_NAME.block
    # 실행 결과
    # 2018-05-26 04:10:32.681 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
    # 2018-05-26 04:10:32.719 UTC [channelCmd] executeJoin -> INFO 002 Successfully submitted proposal to join channel
    # 2018-05-26 04:10:32.719 UTC [main] main -> INFO 003 Exiting.....
  ```
* Anchor Peer 갱신
  ```sh
    peer channel update -o orderer.mycompany.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/MyOrgMSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/mycompany.com/orderers/orderer.mycompany.com/msp/tlscacerts/tlsca.mycompany.com-cert.pem
    # 실행 결과
    # 2018-05-26 04:11:01.550 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
    # 2018-05-26 04:11:01.571 UTC [channelCmd] update -> INFO 002 Successfully submitted channel update
    # 2018-05-26 04:11:01.571 UTC [main] main -> INFO 003 Exiting.....
  ```

### Chaincode 설치 및 실행
* Install Chaincode
  ```sh
    export CHANNEL_NAME=mychan
    peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/
    # 실행 결과
    # 2018-05-26 04:11:48.341 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
    # 2018-05-26 04:11:48.341 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
    # 2018-05-26 04:11:48.646 UTC [main] main -> INFO 003 Exiting.....
  ```
* Initiate Chaincode
   ```sh
    peer chaincode instantiate -o orderer.mycompany.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/mycompany.com/orderers/orderer.mycompany.com/msp/tlscacerts/tlsca.mycompany.com-cert.pem -C $CHANNEL_NAME -n mycc -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "AND ('MyOrgMSP.member')"
    # 실행 결과
    # 2018-05-26 04:12:09.067 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
    # 2018-05-26 04:12:09.067 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
    # 2018-05-26 04:12:27.822 UTC [main] main -> INFO 003 Exiting.....
  ```
* Query
  ```sh
    peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
    # 실행 결과
    # 2018-05-26 04:12:55.603 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
    # 2018-05-26 04:12:55.603 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
    # Query Result: 100
    # 2018-05-26 04:12:55.607 UTC [main] main -> INFO 003 Exiting.....
  ```
* Invoke
  ```sh
    peer chaincode invoke -o orderer.mycompany.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/mycompany.com/orderers/orderer.mycompany.com/msp/tlscacerts/tlsca.mycompany.com-cert.pem -C $CHANNEL_NAME -n mycc -v 1.0 -c '{"Args":["invoke","a", "b", "200"]}'
    # 실행 결과
    # 2018-05-26 04:13:19.555 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
    # 2018-05-26 04:13:19.555 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
    # 2018-05-26 04:13:19.561 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 003 Chaincode invoke successful. result: status:200
    # 2018-05-26 04:13:19.562 UTC [main] main -> INFO 004 Exiting.....
  ```
* Query
  ```sh
    peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
    # 실행 결과
    # 2018-05-26 04:13:46.718 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
    # 2018-05-26 04:13:46.718 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
    # Query Result: -100
    # 2018-05-26 04:13:46.723 UTC [main] main -> INFO 003 Exiting.....
  ```

### 기타 명령어
* 현재 실행 중인 Docker Container들의 상태 확인
  ```sh
    docker ps
  ```
* Container의 Log 확인
  ```sh
    # docker logs [Container Name]
    docker logs orderer.mycompany.com
  ```
* Fabric Network 종료
  ```sh
    # docker-compose-cli.yaml 파일이 존재하는 곳에서 실행
    docker-compose -f docker-compose-cli.yaml down
  ```
