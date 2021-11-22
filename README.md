# Add a new Org to Existing Application Channel

### Starting network

1. Start the network

```bash
./network.sh up createChannel -ca -c mychannel -s couchdb
```

2. Deploy Chaincode

```
./network.sh deployCC -ccn basic -ccp ../asset-transfer-basic/chaincode-go -ccl go
```

```bash
export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=${PWD}/../config
export CORE_PEER_TLS_ENABLED=true

export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp

export CORE_PEER_ADDRESS=localhost:7051

export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

export CORE_PEER_LOCALMSPID=Org1MSP
```

### Invoke and Query

1. Invoke

```
peer chaincode invoke -n basic -C mychannel -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com  --tls --cafile "$ORDERER_CA" --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt  -c '{"Args":["CreateAsset", "100","red", "20","aditya","100"]}'
```

2. Query

```
 peer chaincode query -C mychannel -n basic -c '{"Args":["GetAllAssets"]}'
```

### Org3 Setup

1. Generate Certs

```bash
cd addOrg3
./addOrg3.sh generate -ca
```

1. Generate org config

```bash
export FABRIC_CFG_PATH=$PWD
../../bin/configtxgen -printOrg Org3MSP > ../organizations/peerOrganizations/org3.example.com/org3.json
```

1. Starting docker container

   ```bash
   docker-compose -f docker/docker-compose-org3.yaml up -d
   ```

2. setup ENV

```bash
cd ..

export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=${PWD}/../config/
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051
```

1. Fetch config

```bash
peer channel fetch config channel-artifacts/config_block.pb -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com -c mychannel --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

1. Decoding config block and trimming it

```bash
configtxlator proto_decode --input ./channel-artifacts/config_block.pb --type common.Block --output ./channel-artifacts/config_block.json
jq ".data.data[0].payload.data.config" ./channel-artifacts/config_block.json > ./channel-artifacts/config.json
```

1. Adding config

```bash
jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' ./channel-artifacts/config.json ./organizations/peerOrganizations/org3.example.com/org3.json > ./channel-artifacts/modified_config.json
```

1. translate `config.json` back into a protobuf called `config.pb`

```bash
configtxlator proto_encode --input ./channel-artifacts/config.json --type common.Config --output ./channel-artifacts/config.pb
```

1. encode `modified_config.json` to `modified_config.pb`

```bash
configtxlator proto_encode --input ./channel-artifacts/modified_config.json --type common.Config --output ./channel-artifacts/modified_config.pb
```

1. calculate the delta between these two config protobufs

```bash
configtxlator compute_update --channel_id mychannel --original ./channel-artifacts/config.pb --updated ./channel-artifacts/modified_config.pb --output ./channel-artifacts/org3_update.pb
```

1. decode this object into editable JSON format and call it `org3_update.json`

```bash
configtxlator proto_decode --input ./channel-artifacts/org3_update.pb --type common.ConfigUpdate --output ./channel-artifacts/org3_update.json
```

1. Adding headers back

```bash
echo '{"payload":{"header":{"channel_header":{"channel_id":"'mychannel'", "type":2}},"data":{"config_update":'$(cat ./channel-artifacts/org3_update.json)'}}}' | jq . > ./channel-artifacts/org3_update_in_envelope.json
```

1. convert it into the fully fledged protobuf format

```bash
configtxlator proto_encode --input ./channel-artifacts/org3_update_in_envelope.json --type common.Envelope --output ./channel-artifacts/org3_update_in_envelope.pb
```

### Sign the tx

1. Sign from Org1

```bash
peer channel signconfigtx -f channel-artifacts/org3_update_in_envelope.pb
```

1. Env for org2

```bash
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=localhost:9051
```

1. Sign from Org2

```bash
peer channel update -f channel-artifacts/org3_update_in_envelope.pb -c mychannel -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

### **Join Org3 to the Channel**

1. Env

```bash
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org3MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
export CORE_PEER_ADDRESS=localhost:11051
```

1. Fetch block 0

```bash
peer channel fetch 0 channel-artifacts/mychannel.block -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com -c mychannel --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

1. Join channel

```bash
peer channel join -b channel-artifacts/mychannel.block

peer channel getinfo -c mychannel
```

### Chaincode Setup

1. Install CC

```bash
peer lifecycle chaincode install basic.tar.gz
peer lifecycle chaincode queryinstalled
```

1. Approve CC

```bash
export CC_PACKAGE_ID=

peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name basic --version 1.0 --package-id $CC_PACKAGE_ID --sequence 1
```

1. Query Commited

```bash
peer lifecycle chaincode querycommitted --channelID mychannel --name basic --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

1. Invoke CC

```bash
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n basic --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --peerAddresses localhost:11051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt -c '{"function":"InitLedger","Args":[]}'
```

1. Query CC

```bash
peer chaincode query -C mychannel -n basic -c '{"Args":["GetAllAssets"]}'
```
