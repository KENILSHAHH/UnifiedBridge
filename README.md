# Unified Bridge

This is simple tutorial to explain the lifecycle of bridging assets to and fro from chains connected to AggLayer using [Unified Bridge Contract](https://github.com/0xPolygonHermez/zkevm-contracts/blob/main/contracts/v2/PolygonZkEVMBridgeV2.sol) | [Contract Address](https://sepolia.etherscan.io/address/0x528e26b25a34a4A5d0dbDa1d57D318153d2ED582) 


**Basic architecture of the Unified Bridge**
--- 
<img width="617" alt="Screenshot 2024-07-31 at 3 39 35 PM" src="https://github.com/user-attachments/assets/90b496fc-4cee-4d40-84b4-89bba762accd">

**Get Started**
---

To get started with the project, clone the project and go to the root directory

```
git clone https://github.com/KENILSHAHH/AggLayerUnifiedBridge
cd AggLayerUnifiedBridge
```

Create a .env file in the root directory and copy paste the content of *.env.example* file into your .env file. 

Enter your wallet's private key. Its recommended to use rpc endpoints from Alchemy or Infura

Once done, install the dependencies using yarn

```
yarn install
```

The `bridgeETH.js` file shows an example of bridging 0.01 ETH from Polygon zkEVM Cardona testnet to Sepolia testnet. To bridge a native token, the ETH token contract address is nothing but a Zero Address (0x0000000000000000000000000000000000000000) .To bridge a custom ERC20 between AggLayer Chains, just change the token contract address in `constants.js` file 

**Note : If you are bridging custom ERC20 token, you need to approve contract to spend the amount of tokens on your behalf**

**Lifecycle of a bridge transaction**
---
Any ERC20 asset can be bridged with the help of Unified Bridge across AggLayer Chains

This can be done by hitting the [bridgeAsset](https://github.com/0xPolygonHermez/zkevm-contracts/blob/a5eacc6e51d7456c12efcabdfc1c37457f2219b2/contracts/v2/PolygonZkEVMBridgeV2.sol#L204C5-L211C7) function in [Unified Bridge Contract](https://github.com/0xPolygonHermez/zkevm-contracts/blob/main/contracts/v2/PolygonZkEVMBridgeV2.sol)
 with the given params.An example can be found in the `bridgeETH.js` file in the repo

```solidity 
function bridgeAsset(
        uint32 destinationNetwork,      // networkd id which is either 0,1,2 for sepolia, polygonZkEVMCardona, astar zKyoto respectively
        address destinationAddress,     // destination Ethereum wallet address  
        uint256 amount,                 // Amount of tokens to be bridged (e.g for 1 Ether :- amount = 1000000000000000000n)
        address token,                  // erc20 token address (ZERO ADDRESS in case of ETHER)
        bool forceUpdateGlobalExitRoot, // true
        bytes calldata permitData       // "0x"
    )
```

Once the transaction is successful on the source chain, an [event](https://github.com/0xPolygonHermez/zkevm-contracts/blob/a5eacc6e51d7456c12efcabdfc1c37457f2219b2/contracts/v2/PolygonZkEVMBridgeV2.sol#L97C3-L106C7) `BridgeEvent` is emitted which looks like below 

```solidity
  event BridgeEvent(
        uint8 leafType,
        uint32 originNetwork,
        address originAddress,
        uint32 destinationNetwork,
        address destinationAddress,
        uint256 amount,
        bytes metadata,
        uint32 depositCount
    );
```
It is vital to listen to this event to get all the variables and store them, which are required for generation of merkle proof once the asset is ready to be claimed on the destination chain (which takes around ~1 hour for bridging from Polygon Cardona zkEVM -> Sepolia and about 15 mins for Sepolia -> Polygon Cardona zkEVM)

This merkle proof can be easily generated by sending a GET request to the below api which is for testnets

```javascript
`https://api-gateway.polygon.technology/api/v3/proof/testnet/merkle-proof?networkId=${sourceNetworkId}&depositCount=${depositCount}`
```
where `sourceNetworkId` is your source chain's network id. For e.g for Sepolia `sourceNetworkId = 0`

and `depositCount` can be extracted by listening to the `BridgeEvent` 


Once the Merkle Proof is generated, it can be used to pass into the claimEvent Function which then processes the claim. The claim function should be ran by destinationAddress on the destinationChain. It looks like below 

```solidity 
function claimAsset(
        bytes32[_DEPOSIT_CONTRACT_TREE_DEPTH] calldata smtProofLocalExitRoot,
        bytes32[_DEPOSIT_CONTRACT_TREE_DEPTH] calldata smtProofRollupExitRoot,
        uint256 globalIndex,
        bytes32 mainnetExitRoot,
        bytes32 rollupExitRoot,
        uint32 originNetwork,
        address originTokenAddress,
        uint32 destinationNetwork,
        address destinationAddress,
        uint256 amount,
        bytes calldata metadata
    )
```






