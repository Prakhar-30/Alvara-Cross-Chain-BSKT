# CREATE?

**CREATE2** is an Ethereum opcode that lets you deploy smart contracts to **predictable addresses** - you know the address BEFORE deploying.

## Normal Contract Deployment (CREATE)

```solidity
// Address depends on deployer's nonce (changes each time)
address unpredictable = address(new MyContract());
// Address = hash(deployer_address, nonce)
// Problem: Can't predict address in advance
```

## CREATE2 Deployment

```solidity
// Address depends on: deployer + salt + bytecode (all known beforehand)
address predictable = address(new MyContract{salt: bytes32("unique-id")}());
// Address = hash(0xFF, deployer_address, salt, bytecode_hash)
// Benefit: Address is deterministic and predictable!
```

## Why We Use It for Receipt Tokens

### In Approach 2:

```solidity
// Deploy receipt token for USDe on Base (chain 8453)
bytes32 salt = keccak256(abi.encodePacked(uint256(8453), address(USDe)));
address receiptToken = address(new ReceiptToken{salt: salt}());

// Result: rUSDe-Base always deploys to SAME address
// Even if we deploy it 100 times across different baskets!
```

### The Use:

1. **First basket** creates rUSDe-Base → deploys to `0xABC...123`
2. **Second basket** needs rUSDe-Base → CREATE2 sees it exists at `0xABC...123`, skips deployment
3. **Third basket** needs rUSDe-Base → Reuses same `0xABC...123` address


### Example:

```
Chain: Base (8453)
Token: USDe (0x123...abc)
Salt: keccak256(8453, 0x123...abc) = 0xdef...456

Receipt Address = CREATE2(Factory, 0xdef...456, ReceiptTokenBytecode)
                = 0x789...xyz (same every time!)

Basket #1 deploys rUSDe-Base → 0x789...xyz
Basket #2 needs rUSDe-Base → Already exists at 0x789...xyz ✅
Basket #3 needs rUSDe-Base → Already exists at 0x789...xyz ✅
```

## TL;DR

**CREATE2 = Deploy contracts to the same address every time** by using a deterministic formula instead of a nonce. Perfect for reusable receipt tokens!
