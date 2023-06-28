# mip-nft-standard

The following proposal will describe a standard for NFTs on Mina. Core of the standard is the functionality for accounts to "hold" and transfer NFTs. Both externally owned accounts and smart contract accounts should be able to access the full functionality.

The following interface tries to stay close to the EVM ERC721 Standard, given its popularity and the large number of developers familiar with it. 

State will be handled preferably onchain. 

### Table of contents: 
- Interface
- Challenges 
    - Account Creation Fee 
- Design 
- Exemplary implementation

## Interface 

As described the interface is based on the ERC721 standard as it is a proven concept. 

Given the rather off-chain nature of Mina state there could also be a NFT standard that e.g. tries to provide functionality in the privacy-area. However, this does not (yet) seem to be a sought-after feature in the existing NFT ecosystem. 

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

As in the ERC721 standard, the `mint` and `burn` functions are not part of the standard. Looking at the Ethereum NFT ecosystem there are a multitude of different implementations for these functions and they play no role in the functionality of the applications most reliant on a standardized interface (marketplaces and indexing solutions). 

They are, however, part of the exemplary implementation.

Besides the function described above the efficient integration of NFTs in (offchain-) applications relies on the ability to index the state of the nft contract. The ERC721 standard relies on the following events, which also seem appropriate for this standard. 

Note: depending on the ability to map on-chain from tokenId to NftAccount and from/to to OwnerInfoAccount (see Design) we might be reliable to add more arguments to the above described functions. These would be stored offchain and e.g. emitted as event.

Note: `setApprovalForAll` and `isApprovedForAll` rely on off-chain state and a merkle root (simplified 'merkleData'). Therefore transferFrom is reliant on off-chain state and a merkle root as well, IF it is called by an approvedForAll operator.

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

## Challenges

The most obvious challenge is the limited on-chain state per zkApp account. Saving the full state (e.g. ownerships and approvals) of an NFT contract in a single zkApp is not possible without utilisation of Merkle Maps or Trees. These, however, would lead to race conditions where transfers (which include mints and burns) rely on the most-up-to-date merkle root, which is again influenced by other transfers. A multitude of invalid proofs and therefore failed transactions in times of high collection-throughput would be the result. 

An supposedly obvious approach to solve that challenge might be a design that relies on the creation of one custom token per NFT. This would, however, lead to a account creation fee of 1 Mina on nearly every transfer. The only expection to this would be the transfer of an NFT to an account that already held that NFT in the past, since no new account would be created. 

Another challenge is the efficient tracking of balances. 

## Design 

The following design describes an implemenation of the standard that relies on separation of (on-chain) state across three different types of zkApps. 


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
    @state(UInt64) totalSupply = State<UInt64>();
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
    @state(PublicKey) owner = State<PublicKey>();
    @state(UInt64) tokenId = State<UInt64>();
    @state(Field) approval = State<PublicKey>();
```


# Open 
- is there a way to deterministically create zkApp accounts from the collection zkApp account. That would allow for a very efficient onchain mapping from tokenId to NftAccount. Otherwise this has to happen offchain. The ` getAccountOf(address: PublicKey)` method of the fungible token standard might lead into the right direction here.




