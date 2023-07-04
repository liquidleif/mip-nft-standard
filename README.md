# MIP-721: Non-Fungible Token Standard 

\---  
mip: 721  
title: Non-Fungible Token Standard  
description: A standardized interface for the functionality for accounts to "hold" and transfer NFTs.   
author: Leif L. [(@etherleif)](https://github.com/etherleif), Sebastian G. [(@onchainguy-eth)](https://github.com/onchainguy-eth)  
discussions-to: <new topic on https://forums.minaprotocol.com/ is awaiting approval>  
status: Draft  
type: tbd  
created: 2023-28-06  
\---   

## Abstract
This MIP describes a standard for NFTs based on smart contracts on Mina. It provides a standardized interface and defines the basic mechanisms regarding holding and transferring NFTs. 

Looking mainly (but not exclusively) at the Ethereum and Solana NFT ecosystem, the most popular use cases for NFTs so far are 
- digital collectibles & art (e.g. CryptoPunks, DeGods, Autoglyphs) 
- decentralized finance (e.g. Uniswap V3 LP NFTs)
- gaming (e.g. Axie Infinity, Sorare)

The intended users of the NFTs as of today therefore would primarily be 
- collectors and traders of digital art and collectibles
- decentralized finance users that profit from the functionality that NFTs can offer for defi protocols 
- user that participate in games with onchain functionality 

The typical individuals representing these users are regular web3 users equipped with a wallet who are familiar with the concept of NFTs and have a basic understanding of the underlying technology. 

## Motivation 
Without a dedicated standard for NFTs on Mina it is very unlikely that the NFT ecosystem will be able to grow and thrive. 

The history of ERC721, the most-adopted NFT standard on Ethereum has shown how a successful standard enables 
- the onboarding of 100s of thousands of users into NFTs via integration into wallets and marketplaces (e.g. Metamask, Trust Wallet, OpenSea)
- a multitude of applications to be built on top of it (e.g. indexing solutions, fractionalization, nftFi solutions like lending)
- artists, communities and game developers to experiment with new forms of art and collectibles (e.g. Autoglyphs, DeGods, Sorare).

It furthermore has shown the importance to define the standard in a way that enables innovation and experimentation. Its flexibility regarding implementation enabled for example the creation of 
- Autoglyphs (generative art) 
- ERC721a (gas optimization) 
- Checks.art (composable art).

Given that success of the ERC721 standard and also its popularity and the large number of developers familiar with it, it seems reasonable to base the Mina NFT standard on it.

## Specification 

As described in the Motivation the interface is based on the ERC721 standard.

Given the rather off-chain nature of Mina state there could also be a NFT standard that e.g. tries to provide more functionality in the privacy-area. However, this does not (yet) seem to be a sought-after feature in the existing NFT ecosystem. 

The following interface 
- is meant to be as close as possible to the ERC721 standard 
- focusses on the functionality that is most relevant for the most popular use cases of NFTs and their integration into wallets and marketplaces
- is explicitely extendable to enable innovation and experimentation. 

```javascript
public balanceOf(address: Publickey) : UInt64;
public ownerOf(tokenId: UInt64): PublicKey; 
public transferFrom(from: PublicKey, to: PublicKey, tokenId: UInt64): void;
public transferFrom(from: PublicKey, to: PublicKey, tokenId: UInt64, ...merkleData): void;
public approve(to: PublicKey, tokenId: UInt64): void;
public setApprovalForAll(operator: PublicKey, approved: Bool, ...merkleData): void;
public getApproved(tokenId: UInt64): PublicKey;
public isApprovedForAll(owner: PublicKey, operator: PublicKey, ...merkleData): Bool;
public name(): string;
public symbol(): string;
public tokenURI (tokenId: UInt64): string;
```

As in the ERC721 standard, the `mint` and `burn` functions are not part of the interface standard. Looking at the Ethereum NFT ecosystem there are a multitude of different implementations for these functions and they play no role in the functionality of the applications most reliant on a standardized interface (marketplaces and indexing solutions). 

They are, however, part of the exemplary implementation.

Besides the function described above the efficient integration of NFTs in (offchain-) applications relies on the ability to index the state of the nft contract. The ERC721 standard relies on the following events, which also seem appropriate for this standard. 


```javascript


class Transfer extends Struct({ from: PublicKey, to: Publickey, tokenId: UInt64 }) {}
class Approval extends Struct({ owner: PublicKey, approved: Publickey, tokenId: UInt64 }) {}
class ApprovalForAll extends Struct({ owner: PublicKey, operator: Publickey, approved: Bool }) {}


 events = {
    'transfer': Transfer,
    'approval': Approval,
    'approvalForAll': ApprovalForAll,
  };
```

Note: depending on the ability to map on-chain from tokenId to NftAccount and from/to to OwnerInfoAccount (see Design) we might be reliable to add more arguments to the above described functions. These would be stored offchain and e.g. emitted as event in the case of NftAccount or OwnerInfoAccount creation.

Note: `setApprovalForAll` and `isApprovedForAll` rely on off-chain state and a merkle root (simplified 'merkleData'). Therefore transferFrom is reliant on off-chain state and a merkle root as well, IF it is called by an approvedForAll operator.

## Rationale [wip]

The most obvious challenge is the limited on-chain state per zkApp account. Saving the full state (e.g. ownerships and approvals) of an NFT contract in a single zkApp is not possible without utilisation of Merkle Maps or Trees. These, however, would lead to race conditions where transfers (which include mints and burns) rely on the most-up-to-date merkle root, which is again influenced by other transfers. A multitude of invalid proofs and therefore failed transactions in times of high collection-throughput would be the result. 

One supposedly obvious approach to solve that challenge might be a design that relies on the creation of one custom token per NFT. This would, however, lead to a account creation fee of 1 Mina on nearly every transfer. The only exception to this would be the transfer of an NFT to an account that already held that NFT in the past, since no new account would be created. 

Another challenge is the efficient tracking of balances. 

## Test Cases 
[wip] Will be part of the example implementation

## Reference Implementation 

The following design describes an implementation of the standard that relies on separation of (on-chain) state across three different smart contracts. 

This partioning of state across these three contracts is very close to popular ERC721 implementations on Ethereum, like the one from [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol), whose state (besides name and symbol) essentialy consists of four mappings and two string variables: 
- from tokenId to owner (moved to NftAccount)
- from tokenId to approved address (moved to NftAccount)
- from owner to number of owned tokens (moved to OwnerInfoAccount)
- from owner to approved operators (moved to OwnerInfoAccount)
- name (moved to CollectionAccount)
- symbol (moved to CollectionAccount)

The advantage of an alignment with this proven approach is that, again, many developers from the Ethereum and EVM ecosystem will be easily able to understand the implementation. 

A further advantage is that other smart contracts on the Mina blockchain can easily interact with the NFT contract since the state is mainly onchain. This also makes data-availability to an non-issue for the involved users and 3rd parties. 

The approvalForAllRoot of the OwnerInfoAccount is the only exception to this. This might seem as a problem at first, but the most popular use case (so far) of the approvalForAll functionality is the integration of NFTs in marketplaces, which usually rely on off-chain state (provided by the fulfiller of an order) anyway for gasless listings. 

### CollectionAccount: 
- exists once per collection
- holds name, symbol, totalSupply of the collection 
- hosts the above described functions and proxies them to the nftAccounts (if necessary)
- creates/updates nftAccounts (on transfers & approvals)
- creates/updates ownerInfoAccounts (on transfers & approvals)
- emits the above described events

State:
```javascript
    @state(Field) name = State<Field>();
    @state(Field) symbol = State<Field>();
    @state(UInt64) totalSupply = State<UInt64>(); // optional
```

### OwnerInfoAccount
- exists once per owner or past owner of an NFT
- holds owner, owner balance and approvalForAllRoot
- is updated on transfers and approvalForAlls via the Collection

State: 
```javascript
    @state(PublicKey) ownerAddress = State<PublicKey>();
    @state(UInt64) balance = State<UInt64>();
    @state(Field) approvalForAllRoot = State<Field>();
```

### NftAccount
- exists once per NFT
- holds tokenId, owner, approval
- is updated on transfers and approvals via the Collection

State:
```javascript
    @state(PublicKey) ownerAddress = State<PublicKey>();
    @state(UInt64) tokenId = State<UInt64>();
    @state(PublicKey) approval = State<PublicKey>();
```

A proof of concept implementation is currently under development.

## Security Considerations
[wip] 

## Copyright
Copyright and related rights waived via CC0.





