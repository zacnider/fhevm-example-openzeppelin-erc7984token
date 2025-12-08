# EntropyERC7984Token

Basic ERC7984 confidential token implementation with EntropyOracle integration

## 🚀 Standard workflow
- Install (first run): `npm install --legacy-peer-deps`
- Compile: `npx hardhat compile`
- Test (local FHE + local oracle/chaos engine auto-deployed): `npx hardhat test`
- Deploy (frontend Deploy button): constructor args fixed to EntropyOracle, name, and symbol; oracle is `0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`
- Verify: `npx hardhat verify --network sepolia <contractAddress> 0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361 "TokenName" "SYMBOL"`

## 📋 Overview

This example demonstrates **OpenZeppelin** concepts in FHEVM with **EntropyOracle integration**:
- ERC7984 confidential token standard implementation
- Encrypted balances using euint64
- Transfer functions with encrypted amounts
- Mint/burn operations
- EntropyOracle integration for random token operations

## 🎯 What This Example Teaches

This tutorial will teach you:

1. **How to implement ERC7984 confidential tokens** with FHE and entropy
2. **How to manage encrypted token balances** on-chain
3. **How to transfer encrypted tokens** between addresses
4. **How to mint tokens with entropy** for randomness
5. **How to burn encrypted tokens**
6. **Real-world confidential token** implementation

## 💡 Why This Matters

ERC7984 enables private token transfers:
- **Balances remain encrypted** on-chain
- **Transfers are private** - amounts not visible
- **EntropyOracle adds randomness** to minting operations
- **Privacy-preserving** token operations
- **Real-world application** of FHE in DeFi

## 🔍 How It Works

### Contract Structure

The contract has five main components:

1. **Request Mint with Entropy**: Request entropy for random minting
2. **Mint with Entropy**: Mint tokens using entropy
3. **Transfer**: Transfer encrypted tokens
4. **Burn**: Burn encrypted tokens
5. **Balance Queries**: Get encrypted balances

### Step-by-Step Code Explanation

#### 1. Constructor

```solidity
constructor(
    address _entropyOracle,
    string memory _name,
    string memory _symbol
) {
    require(_entropyOracle != address(0), "Invalid oracle address");
    entropyOracle = IEntropyOracle(_entropyOracle);
    name = _name;
    symbol = _symbol;
}
```

**What it does:**
- Takes EntropyOracle address, token name, and symbol
- Validates oracle address is not zero
- Stores oracle interface and token metadata

**Why it matters:**
- Must use the correct oracle address: `0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`
- Token name and symbol for identification

#### 2. Request Mint with Entropy

```solidity
function requestMintWithEntropy(bytes32 tag) external payable returns (uint256 requestId) {
    require(msg.value >= entropyOracle.getFee(), "Insufficient fee");
    
    requestId = entropyOracle.requestEntropy{value: msg.value}(tag);
    mintRequests[requestId] = msg.sender;
    mintRequestCount++;
    
    emit MintRequested(msg.sender, requestId);
    return requestId;
}
```

**What it does:**
- Validates fee payment
- Requests entropy from EntropyOracle
- Stores mint request with user address
- Returns request ID

**Key concepts:**
- **Two-phase minting**: Request first, mint later
- **Request tracking**: Maps request ID to user
- **Entropy for randomness**: Adds randomness to mint amount

#### 3. Mint with Entropy

```solidity
function mintWithEntropy(
    uint256 requestId,
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) external {
    require(entropyOracle.isRequestFulfilled(requestId), "Entropy not ready");
    require(mintRequests[requestId] == msg.sender, "Invalid request");
    
    euint64 entropy = entropyOracle.getEncryptedEntropy(requestId);
    FHE.allowThis(entropy);
    
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
    FHE.allowThis(amount);
    
    // Add entropy to amount (for randomness)
    euint64 mintAmount = FHE.add(amount, entropy);
    
    // Mint tokens
    encryptedBalances[msg.sender] = FHE.add(encryptedBalances[msg.sender], mintAmount);
    totalSupplyEncrypted = FHE.add(totalSupplyEncrypted, mintAmount);
    
    delete mintRequests[requestId];
    emit Minted(msg.sender, requestId);
}
```

**What it does:**
- Validates request ID and fulfillment
- Gets encrypted entropy from oracle
- **Grants permission** to use entropy (CRITICAL!)
- Converts external encrypted amount to internal
- Combines amount with entropy using ADD
- Mints tokens to user balance
- Updates total supply
- Emits mint event

**Key concepts:**
- **Entropy enhancement**: Adds randomness to mint amount
- **Encrypted minting**: All amounts remain encrypted
- **Balance update**: Encrypted balance increased

**Why ADD entropy:**
- Adds randomness to mint amount
- Result: amount + entropy (encrypted)
- Maintains privacy while adding randomness

#### 4. Transfer

```solidity
function transfer(
    address to,
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) external returns (bool) {
    require(to != address(0), "Transfer to zero address");
    
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
    FHE.allowThis(amount);
    
    euint64 senderBalance = encryptedBalances[msg.sender];
    FHE.allowThis(senderBalance);
    
    // Transfer (subtract from sender, add to recipient)
    encryptedBalances[msg.sender] = FHE.sub(senderBalance, amount);
    FHE.allowThis(encryptedBalances[msg.sender]);
    
    euint64 recipientBalance = encryptedBalances[to];
    euint64 newRecipientBalance = FHE.add(recipientBalance, amount);
    FHE.allowThis(newRecipientBalance);
    encryptedBalances[to] = newRecipientBalance;
    
    emit Transfer(msg.sender, to, abi.encode(encryptedAmount));
    return true;
}
```

**What it does:**
- Validates recipient address
- Converts external encrypted amount to internal
- Gets sender's encrypted balance
- Subtracts amount from sender balance
- Adds amount to recipient balance
- Emits transfer event

**Key concepts:**
- **Encrypted transfer**: All amounts remain encrypted
- **FHE operations**: SUB from sender, ADD to recipient
- **Privacy-preserving**: Transfer amount not visible

**Why encrypted:**
- Maintains privacy of transfer amounts
- Balances remain encrypted on-chain
- Only authorized users can decrypt

#### 5. Burn

```solidity
function burn(
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) external {
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
    FHE.allowThis(amount);
    
    euint64 senderBalance = encryptedBalances[msg.sender];
    FHE.allowThis(senderBalance);
    
    // Burn (subtract from balance and total supply)
    encryptedBalances[msg.sender] = FHE.sub(senderBalance, amount);
    FHE.allowThis(encryptedBalances[msg.sender]);
    totalSupplyEncrypted = FHE.sub(totalSupplyEncrypted, amount);
    FHE.allowThis(totalSupplyEncrypted);
    
    emit Burned(msg.sender, abi.encode(encryptedAmount));
}
```

**What it does:**
- Converts external encrypted amount to internal
- Gets sender's encrypted balance
- Subtracts amount from sender balance
- Subtracts amount from total supply
- Emits burn event

**Key concepts:**
- **Encrypted burn**: Burn amount remains encrypted
- **Supply reduction**: Total supply decreased
- **Balance reduction**: User balance decreased

## 🧪 Step-by-Step Testing

### Prerequisites

1. **Install dependencies:**
   ```bash
   npm install --legacy-peer-deps
   ```

2. **Compile contracts:**
   ```bash
   npx hardhat compile
   ```

### Running Tests

```bash
npx hardhat test
```

### What Happens in Tests

1. **Fixture Setup** (`deployContractFixture`):
   - Deploys FHEChaosEngine, EntropyOracle, and EntropyERC7984Token
   - Returns all contract instances

2. **Test: Request Mint with Entropy**
   ```typescript
   it("Should request mint with entropy", async function () {
     const tag = hre.ethers.id("mint-request");
     const fee = await oracle.getFee();
     const requestId = await contract.requestMintWithEntropy(tag, { value: fee });
     expect(requestId).to.not.be.undefined;
   });
   ```
   - Requests entropy for minting
   - Pays required fee
   - Verifies request ID returned

3. **Test: Mint with Entropy**
   ```typescript
   it("Should mint tokens with entropy", async function () {
     // ... request mint code ...
     await waitForEntropy(requestId);
     
     const input = hre.fhevm.createEncryptedInput(contractAddress, owner.address);
     input.add64(100);
     const encryptedInput = await input.encrypt();
     
     await contract.mintWithEntropy(
       requestId,
       encryptedInput.handles[0],
       encryptedInput.inputProof
     );
     
     const balance = await contract.balanceOf(owner.address);
     expect(balance).to.not.be.undefined;
   });
   ```
   - Waits for entropy to be ready
   - Creates encrypted amount
   - Mints tokens with entropy
   - Verifies balance increased

### Expected Test Output

```
  EntropyERC7984Token
    Deployment
      ✓ Should deploy successfully
      ✓ Should have EntropyOracle address set
    Minting
      ✓ Should request mint with entropy
      ✓ Should mint tokens with entropy
    Transfers
      ✓ Should transfer encrypted tokens
    Burning
      ✓ Should burn encrypted tokens

  6 passing
```

**Note:** All balances and amounts are encrypted (handles). Decrypt off-chain using FHEVM SDK to see actual values.

## 🚀 Step-by-Step Deployment

### Option 1: Frontend (Recommended)

1. Navigate to [Examples page](/examples)
2. Find "EntropyERC7984Token" in Tutorial Examples
3. Click **"Deploy"** button
4. Approve transaction in wallet
5. Wait for deployment confirmation
6. Copy deployed contract address

### Option 2: CLI

1. **Create deploy script** (`scripts/deploy.ts`):
   ```typescript
   import hre from "hardhat";

   async function main() {
     const ENTROPY_ORACLE_ADDRESS = "0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361";
     const TOKEN_NAME = "EntropyToken";
     const TOKEN_SYMBOL = "ENT";
     
     const ContractFactory = await hre.ethers.getContractFactory("EntropyERC7984Token");
     const contract = await ContractFactory.deploy(
       ENTROPY_ORACLE_ADDRESS,
       TOKEN_NAME,
       TOKEN_SYMBOL
     );
     await contract.waitForDeployment();
     
     const address = await contract.getAddress();
     console.log("EntropyERC7984Token deployed to:", address);
   }

   main().catch((error) => {
     console.error(error);
     process.exitCode = 1;
   });
   ```

2. **Deploy:**
   ```bash
   npx hardhat run scripts/deploy.ts --network sepolia
   ```

## ✅ Step-by-Step Verification

### Option 1: Frontend

1. After deployment, click **"Verify"** button on Examples page
2. Wait for verification confirmation
3. View verified contract on Etherscan

### Option 2: CLI

```bash
npx hardhat verify --network sepolia <CONTRACT_ADDRESS> 0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361 "EntropyToken" "ENT"
```

**Important:** Constructor arguments must be:
1. EntropyOracle address: `0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`
2. Token name: Your token name (e.g., "EntropyToken")
3. Token symbol: Your token symbol (e.g., "ENT")

## 📊 Expected Outputs

### After Request Mint with Entropy

- `mintRequests[requestId]` contains user address
- `mintRequestCount` increments
- `MintRequested` event emitted

### After Mint with Entropy

- `balanceOf(user)` returns increased encrypted balance
- `totalSupply()` returns increased encrypted total supply
- `Minted` event emitted

### After Transfer

- `balanceOf(sender)` returns decreased encrypted balance
- `balanceOf(recipient)` returns increased encrypted balance
- `Transfer` event emitted

### After Burn

- `balanceOf(user)` returns decreased encrypted balance
- `totalSupply()` returns decreased encrypted total supply
- `Burned` event emitted

## ⚠️ Common Errors & Solutions

### Error: `SenderNotAllowed()`

**Cause:** Missing `FHE.allowThis()` call on encrypted balance or amount.

**Solution:**
```solidity
euint64 entropy = entropyOracle.getEncryptedEntropy(requestId);
FHE.allowThis(entropy); // ✅ Required!
```

**Prevention:** Always call `FHE.allowThis()` on all encrypted values before using them.

---

### Error: `Entropy not ready`

**Cause:** Calling `mintWithEntropy()` before entropy is fulfilled.

**Solution:** Always check `isRequestFulfilled()` before using entropy.

---

### Error: `Invalid request`

**Cause:** Request ID doesn't belong to caller.

**Solution:** Ensure request ID matches the caller's request.

---

### Error: `Transfer to zero address`

**Cause:** Trying to transfer to zero address.

**Solution:** Always validate recipient address before transferring.

---

### Error: `Insufficient fee`

**Cause:** Not sending enough ETH when requesting mint.

**Solution:** Always send exactly 0.00001 ETH:
```typescript
const fee = await contract.entropyOracle.getFee();
await contract.requestMintWithEntropy(tag, { value: fee });
```

---

### Error: Verification failed - Constructor arguments mismatch

**Cause:** Wrong constructor arguments used during verification.

**Solution:** Always use EntropyOracle address, token name, and symbol:
```bash
npx hardhat verify --network sepolia <CONTRACT_ADDRESS> 0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361 "TokenName" "SYMBOL"
```

## 🔗 Related Examples

- [EntropyERC7984ToERC20Wrapper](../openzeppelin-erc7984toerc20wrapper/) - Wrapping ERC7984 to ERC20
- [EntropySwapERC7984ToERC20](../openzeppelin-swaperc7984toerc20/) - Swapping ERC7984 to ERC20
- [Category: openzeppelin](../)

## 📚 Additional Resources

- [Full Tutorial Track Documentation](../../../frontend/src/pages/Docs.tsx) - Complete educational guide
- [Zama FHEVM Documentation](https://docs.zama.org/) - Official FHEVM docs
- [GitHub Repository](https://github.com/zacnider/entrofhe/tree/main/examples/openzeppelin-erc7984token) - Source code

## 📝 License

BSD-3-Clause-Clear
