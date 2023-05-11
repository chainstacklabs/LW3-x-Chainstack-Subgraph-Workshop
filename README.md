# LW3-x-Chainstack Subgraph Workshop

This repository serves as a codebase for the LW3 community to use for coding along with the Subgraph workshop held on 11/5/2023.

## Getting started

Before anything else, make sure you have the graph-cli installed in your system. You can do that with:

```bash
npm install -g @graphprotocol/graph-cli
```
OR

```bash
yarn global add @graphprotocol/graph-cli
```
To check if the CLI was installed correctly, run this command in your terminal:

```shell
graph
```

If the graph-cli is installed correctly, open up a new directory in any code editor.

## Initializing a new Subgraph project

To quickly bootstrap a new subgraph project, use:

```shell
graph init
```
This command will open up an interactive UI in your terminal. Create a new subgraph project with the following parameters:

```yaml
Protocol: Ethereum
Product for which to initialize: subgraph-studio
Subgraph Slug: BAYC
Directory to create the subgraph in: ChainstackSubgraph
Ethereum network: mainnet
Contract address: 0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D
Contract Name: BoredApeYachtClub
Index contract events as entities: true
```

## schema.graphql

Think of the schema file as a sort of blueprint for your subgraph. The schema file defines what data your subgraph will store, and how exactly will it be stored.

Paste the following code inside the `schema.graphql` file:

```
type Transfer @entity(immutable: true) {
  id: Bytes!
  from: Bytes!
  to: Bytes!
  tokenId: BigInt! 
  blockNumber: BigInt!
  transactionHash: Bytes!
}

type BoredApe @entity {
  id: ID!
  creator: Bytes!
  newOwner: Bytes!
  tokenURI: String!
  blockNumber: BigInt!
}

type Property @entity {
  id: ID!
  image: String
  background: String
  clothes: String
  earring: String
  eyes: String
  fur: String
  hat: String
  mouth: String
}
```
We define 3 entities in the schema file:

1. The Transfer entity keeps a record of all the transactions that involved the transfer of a bored ape NFT.
2. The BoredApe entity keeps a record of all the bored ape NFTs themselves. We record the original owner and the current owner amongst other data points. Please note that some fields like the current owner will keep on changing everytime the NFT is transferred.
3. The Property entity stores the metatadata for each Ape.

## subgraph.yaml

You should update your YAML file everytime you make changes to the schema file. Paste the following code inside the YAML file:

```yaml
specVersion: 0.0.5
description: A subgraph to index data on the Bored Apes contract
features:
  - ipfsOnEthereumContracts
schema:
  file: ./schema.graphql
dataSources:
  - kind: ethereum
    name: BoredApeYachtClub
    network: mainnet
    source:
      address: "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D"
      abi: BoredApeYachtClub
      startBlock: 12287507
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Transfer
        - BoredApe
        - Property
      abis:
        - name: BoredApeYachtClub
          file: ./abis/BoredApeYachtClub.json
      eventHandlers:
        - event: Transfer(indexed address,indexed address,indexed uint256)
          handler: handleTransfer
      file: ./src/bored-ape-yacht-club.ts
```
The YAML file is a collection of nested key value pairs that keeps a store of some crucial metadata about our subgraph. It is important to keep it updated.

## Mapping functions to schemas

Make sure to save all your changes in the schema and YAML files.
Now run this command in your terminal:

```shell
graph codegen
```

Go to `src/bored-ape-yacht-club.ts` and delete everything. Paste the following code at the top of the file to import all the AssemblyScript types we need from the generated folder:

```javascript
import {
  Transfer as TransferEvent,
  BoredApeYachtClub as BoredApeYachtClubContract,
} from "../generated/BoredApeYachtClub/BoredApeYachtClub"
import {
  BoredApe,
  Transfer,
  Property
} from "../generated/schema"

import { ipfs, json, JSONValue, log } from '@graphprotocol/graph-ts'
```
Now Create a new function named `handleTransfer` as follows:

```javascript
export function handleTransfer(event: TransferEvent): void {

}
```

Let us define the logic to handle our Transfer entity every time this function runs. Paste the following code inside the function:

```javascript
  let transfer = new Transfer(event.transaction.hash.concatI32(event.logIndex.toI32()))
  transfer.from = event.params.from
  transfer.to = event.params.to
  transfer.tokenId = event.params.tokenId
  transfer.blockNumber = event.block.number
  transfer.transactionHash = event.transaction.hash
  transfer.save()
```
Next, paste this code right below the previous snippet:

```javascript
  let contractAddress = BoredApeYachtClubContract.bind(event.address);
  let boredApe = BoredApe.load(event.params.tokenId.toString());

  if(boredApe==null){
  boredApe = new BoredApe(event.params.tokenId.toString());
  boredApe.creator=event.params.to;
  boredApe.tokenURI=contractAddress.tokenURI(event.params.tokenId);
  }

  boredApe.newOwner=event.params.to;
  boredApe.blockNumber=event.block.number;
  boredApe.save();
```

Lastly, paste the following code into the mappings file:

```javascript
const ipfshash = "QmeSjSinHpPnmXmspMjwiXyN6zS4E9zccariGR3jxcaWtq"
  let tokenURI = "/" + event.params.tokenId.toString();
  log.debug('The tokenURI is: {} ', [tokenURI]);

  let property = Property.load(event.params.tokenId.toString());

  if (property == null) {
    property = new Property(event.params.tokenId.toString());

    let fullURI = ipfshash + tokenURI;
    log.debug('The fullURI is: {} ', [fullURI]);

    let ipfsData = ipfs.cat(fullURI);

    if (ipfsData) {
      let ipfsValues = json.fromBytes(ipfsData);
      let ipfsValuesObject = ipfsValues.toObject();

      if (ipfsValuesObject) {
        let image = ipfsValuesObject.get('image');
        let attributes = ipfsValuesObject.get('attributes');

        let attributeArray: JSONValue[];
        if (image) {
          property.image = image.toString();
        }
        if (attributes) {

          attributeArray = attributes.toArray();

          for (let i = 0; i < attributeArray.length; i++) {

            let attributeObject = attributeArray[i].toObject();

            let trait_type = attributeObject.get('trait_type');
            let value_type = attributeObject.get('value');

            let trait: string;
            let value: string;

            if (trait_type && value_type) {

              trait = trait_type.toString();
              value = value_type.toString();

              if (trait && value) {

                if (trait == "Background") {
                  property.background = value;
                }

                if (trait == "Clothes") {
                  property.clothes = value;
                }

                if (trait == "Earring") {
                  property.earring = value;
                }

                if (trait == "Eyes") {
                  property.eyes = value;
                }

                if (trait == "Fur") {
                  property.fur = value;
                }

                if (trait == "Hat") {
                  property.hat = value;
                }

                if (trait == "Mouth") {
                  property.mouth = value;
                }

              }

            }

          }

        }

      }
    }
    
  }

  property.save();
```
