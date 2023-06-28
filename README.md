# mip-nft-standard

The following proposal will describe a standard for NFTs on Mina. Core of the standard is the functionality for accounts to "hold" and transfer NFTs. Both externally owned accounts and smart contract accounts should be able to access the full functionality.

The following interface tries to stay close to the EVM ERC721 Standard, given its popularity and the large number of developers familiar with it. 

However, deviations from ERC721 are to be expected, given (among others) the primary off-chain state of mina zkApps.  

### Table of contents: 
- Interface
- Challenges
- Exemplary implementation

## Interface 

```
public balanceOf(address: Publickey) : UInt64;

```

