# Hyperledger Fabric network configuration example
It is a configuration example of the network, and all the files of this repository has been modified.

If you want to have a clean repository, in the following repository there are basic skeletons for the network creation:

https://github.com/hlf-go/fabrics

## Prerequisites
Go to the following link to install the prerequisites

https://hyperledger-fabric.readthedocs.io/en/release/prereqs.html

## Network topology for this example

![Network Topology](/images/topology.png)

## Configuration files

### crypto-config.yaml
In this file we can find two sections:
- OrdererOrgs - Definition of organizations managing orderer nodes.
- PeerOrgs - Definition of organizations managing peer nodes.

#### OrdererOrgs
In this section we must define the configuration of the orderer.
```
OrdererOrgs:
  
  - Name: Orderer
    Domain: example.com
    Specs:
      - Hostname: orderer
```

#### PeerOrgs
In this section we must define the organizations and peers for organization.
```
PeerOrgs:

  - Name: Org1
    Domain: org1.example.com
    Template:
      Count: 3
    Users:
      Count: 1

  - Name: Org2
    Domain: org2.example.com
    Template:
      Count: 3
    Users:
      Count: 1
    
  - Name: Org3
    Domain: org3.example.com
    Template:
      Count: 3
    Users:
      Count: 1      
```
NOTE: For more information about the configuration, please consult the content of this file. In it you can find explained each field.

### configtx.yaml
In this file we have four sections:
- Profiles - Different configuration profiles may be encoded here to be specified as parameters to the configtxgen tool

    In our example we can see the following configuration:
    ```
    Profiles:

    ThreeOrgsOrdererGenesis:
        Orderer:
            <<: *OrdererDefaults
            Organizations:
                - *OrdererOrg
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *Org1
                    - *Org2
                    - *Org3
    Org1Org3Channel:
        Consortium: SampleConsortium
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - *Org1
                - *Org3
    Org2Org3Channel:
        Consortium: SampleConsortium
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - *Org2
                - *Org3   
    ``` 
    As you can see, we have defined three organizations disstributed in two different channels and a orderer.

- Organizations - This section defines the different organizational identities which will be referenced later in the configuration.

    In our example we can see the following configuration:
    ```
    - &OrdererOrg
        Name: OrdererOrg
        ID: OrdererMSP
        MSPDir: crypto-config/ordererOrganizations/example.com/msp

    - &Org1
        Name: Org1MSP
        ID: Org1MSP
        MSPDir: crypto-config/peerOrganizations/org1.example.com/msp
        AnchorPeers:
            - Host: peer0.org1.example.com
              Port: 7051

    - &Org2
        Name: Org2MSP
        ID: Org2MSP
        MSPDir: crypto-config/peerOrganizations/org2.example.com/msp
        AnchorPeers:
            - Host: peer0.org2.example.com
              Port: 7051

    - &Org3
        Name: Org3MSP
        ID: Org3MSP
        MSPDir: crypto-config/peerOrganizations/org3.example.com/msp
        AnchorPeers:
            - Host: peer0.org3.example.com
              Port: 7051
            - Host: peer1.org3.example.com
              Port: 7051 
    ```
    As you can see, we have defined one organization for the orderer and three organizations with their corresponding anchor peers.

- Orderer - This section defines the values to encode into a config transaction or genesis block for orderer related parameters.

    In our example we have changed the following configuration for the orderer:
    ```
    OrdererType: solo
    Addresses:
        - orderer.example.com:7050
    ```
    Only we have defined a orderer, as you can see in OrdererType field (solo), with the address previously defined.

-  Application - This section defines the values to encode into a config transaction or genesis block for application related parameters

NOTE: For more information about the configuration, please consult the content of this file. In it you can find explained each field.

### base/docker-compose-base.yaml

As you can see in this file, you must to define a service for each orderer and peers which has been defined for each organization.

It is necessary to change the hosts and the paths of the crypto materials according the configuration of the 2 files previous.

### docker-compose-template.yaml

In this file you must to define a service for each CA, for each orderer and peer and we define a service for the client (cli).

In this case the client is deployed only in Organization 1 in the Peer 0.

It is also necessary necessary to change the hosts and the paths of the crypto materials.

### [base/peer-base.yaml](#base-peer-base)
We must pay attention to this file. Make sure that this property has the correct network name:

```
- CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=hyperledgertest_default
```
You can find the network name with the following command:
```
docker nertwork ls
```
Note that you only can obtain the network name only if the network has been started.

### fabricOps.sh 
In this file you must configure the following methods:
- replacePrivateKey: Change the paths of the cryptomaterials. It is necessary to add all the certificates generated previously. 

    In our example we can see the following configuration for this method:
    ```
    CURRENT_DIR=$PWD
    cd crypto-config/peerOrganizations/org1.example.com/ca/
    PRIV_KEY=$(ls *_sk)
    cd $CURRENT_DIR
    sed $OPTS "s/CA1_PRIVATE_KEY/${PRIV_KEY}/g" docker-compose.yaml
    
    cd crypto-config/peerOrganizations/org2.example.com/ca/
    PRIV_KEY=$(ls *_sk)
    cd $CURRENT_DIR
    sed $OPTS "s/CA2_PRIVATE_KEY/${PRIV_KEY}/g" docker-compose.yaml

    cd crypto-config/peerOrganizations/org3.example.com/ca/
    PRIV_KEY=$(ls *_sk)
    cd $CURRENT_DIR
    sed $OPTS "s/CA3_PRIVATE_KEY/${PRIV_KEY}/g" docker-compose.yaml
    ```
- generateChannelArtifacts: It is necessary to configure all the channels defined previously.

    In our case these lines are:
    ```
    $GOPATH/bin/configtxgen -profile ThreeOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block

    $GOPATH/bin/configtxgen -profile Org1Org3Channel -outputCreateChannelTx ./channel-artifacts/channel13.tx -channelID "mychannel13"

    $GOPATH/bin/configtxgen -profile Org2Org3Channel -outputCreateChannelTx ./channel-artifacts/channel23.tx -channelID "mychannel23"
    ```
    The first line define de orderer that can communicate with the three orgs, and the other lines define the channels established between organizations 1 and 2, and between organizations 2 and 3.

    It is necessary to be equal to the variables defined in the configtx.yaml file. Be careful with the channelID, uppercases are considered like ilegal characters.

## Executing the network

### Run the network
```
./fabricOps.sh start
```
Now, you have an environment to deploy and to interact chaincodes. It's very important that the chaincode is deployed in the same docker network than the peers. To check this execute the following command:

```
docker inspect "name_of_a_peer_container
```
At this point you must check the name of the network configured in [base/peer-base.yaml](#base-peer-base).

### Delete and modify the network
With every change in the configuration it is necessary to execute the following command:
```
./fabricOps.sh clean
```
### Interact with the network
To interact with the network creating channels, joining peers and interacting with the chaincode, execute the following command:
```
./fabricOps.sh cli
```
Once we are in the docker container of the hyperledger cli tool we can execute the following commands:
```
011-createchannel-1-3.sh //Channel between orgs 1 and 3 is created
012-createchannel-2-3.sh //Channel between orgs 2 and 3 is created
021-joinOrg1Peer0Channel13.sh //Organization 1 Peer 0 joins in to the channel 13
022-joinOrg3Peer0Channel13.sh //Organization 3 Peer 0 joins in to the channel 13
023-joinOrg2Peer0Channel23.sh //Organization 2 Peer 0 joins in to the channel 23
024-joinOrg3Peer1Channel23.sh //Organization 3 Peer 1 joins in to the channel 23
031-installCCOrg1Peer0.sh //The chaincode is installed in Organization 1 Peer 0
032-installCCOrg2Peer0.sh //The chaincode is installed in Organization 2 Peer 0
033-installCCOrg3Peer0.sh //The chaincode is installed in Organization 3 Peer 0
034-installCCOrg3Peer1.sh //The chaincode is installed in Organization 3 Peer 1
041-instanciateCCOrg1Peer0Ch13.sh //The chaincode is instanciated in the network at Organization 1 Peer 0 Channel 13 
                                  //A docker-container of the chaincode is created
050-invokeCCOrg1Peer0Ch13.sh //The chaincode is invoked at Organization 1 Peer 0 Channel 13
051-queryCCOrg3Peer0Ch13.sh  //The chaincode is consulted from Organization 3 Peer 0 Channel 13
```
