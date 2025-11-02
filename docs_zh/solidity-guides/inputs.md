# 加密输入

本文档介绍了 FHEVM 中加密输入的概念，解释了它们的作用、结构、验证过程以及开发人员如何将它们集成到智能合约和应用程序中。

加密输入是 FHEVM 的核心功能，使用户能够将加密数据推送到区块链上，同时确保数据的机密性和完整性。

## 什么是加密输入？

加密输入是用户以密文形式提交的数据值。这些输入允许敏感信息在被智能合约处理的同时保持机密。它们附有**零知识知识证明 (ZKPoK)**，以确保加密数据的有效性，而不会泄露明文。

### 加密输入的主要特点：

1. **机密性**：数据使用公共 FHE 密钥加密，确保只有授权方才能解密或处理这些值。
2. **通过 ZKPoK 进行验证**：每个加密输入都附有一个证明，验证用户知道密文的明文值，从而防止重放攻击或滥用。
3. **高效打包**：交易的所有输入都按用户定义的顺序打包成单个密文，从而优化了零知识证明的大小和生成。

## 加密函数中的参数

当调用智能合约中的函数时，它可能会接受两种类型的加密输入参数：

1. **`externalEbool`、`externalEaddress`、`externalEuintXX`**：指证明中加密参数的索引，表示一个特定的加密输入句柄。
2. **`bytes`**：包含用于验证的密文和相关的零知识证明。

以下是一个接受多个加密参数的 Solidity 函数示例：

```solidity
function exampleFunction(
  externalEbool param1,
  externalEuint64 param2,
  externalEuint8 param3,
  bytes calldata inputProof
) public {
  // 此处为函数逻辑
}
```

在此示例中，`param1`、`param2` 和 `param3` 是 `ebool`、`euint64` 和 `euint8` 的加密输入，而 `inputProof` 包含相应的 ZKPoK 以验证其真实性。

### 使用 Hardhat 生成输入

在下面的示例中，我们使用 Alice 的地址创建加密输入并提交交易。

```typescript
import { fhevm } from "hardhat";

const input = fhevm.createEncryptedInput(contract.address, signers.alice.address);
input.addBool(canTransfer); // 索引为 0
input.add64(transferAmount); // 索引为 1
input.add8(transferType); // 索引为 2
const encryptedInput = await input.encrypt();

const externalEboolParam1 = encryptedInput.handles[0];
const externalEuint64Param2 = encryptedInput.handles[1];
const externalEuint8Param3 = encryptedInput.handles[2];
const inputProof = encryptedInput.inputProof;

tx = await myContract
  .connect(signers.alice)
  [
    "exampleFunction(bytes32,bytes32,bytes32,bytes)"
  ](signers.bob.address, externalEboolParam1, externalEuint64Param2, externalEuint8Param3, inputProof);

await tx.wait();
```

### 输入顺序

开发人员可以按任何顺序设计函数参数。在 TypeScript 中构造加密输入的顺序与 Solidity 函数中参数的顺序之间没有必需的对应关系。

## 验证加密输入

智能合约通过对照相关的零知识证明验证加密输入来处理它们。这是使用 `FHE.asEuintXX`、`FHE.asEbool` 或 `FHE.asEaddress` 函数完成的，这些函数会验证输入并将其转换为适当的加密类型。

### 验证示例

此示例演示了一个执行多个加密操作的函数，例如更新用户的加密余额和切换加密的布尔标志：

```solidity
function myExample(externalEuint64 encryptedAmount, externalEbool encryptedToggle, bytes calldata inputProof) public {
  // 验证并转换加密的输入
  euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
  ebool toggleFlag = FHE.fromExternal(encryptedToggle, inputProof);

  // 更新用户的加密余额
  balances[msg.sender] = FHE.add(balances[msg.sender], amount);

  // 切换用户的加密标志
  userFlags[msg.sender] = FHE.not(toggleFlag);

  // 此处为 FHE 权限和函数逻辑
  ...
}

// 用于检索用户加密余额的函数
function getEncryptedBalance() public view returns (euint64) {
  return balances[msg.sender];
}

// 用于检索用户加密标志的函数
function getEncryptedFlag() public view returns (ebool) {
  return userFlags[msg.sender];
}
```

### `ConfidentialERC20.sol` 智能合约中的验证示例

以下是一个智能合约函数的示例，该函数在继续之前会验证加密输入：

```solidity
function transfer(
  address to,
  externalEuint64 encryptedAmount,
  bytes calldata inputProof
) public {
  // 验证提供的加密金额并将其转换为加密的 uint64
  euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);

  // 此处为函数逻辑，例如转移资金
  ...
}
```

### 验证的工作原理

1. **输入验证**：
   `FHE.fromExternal` 函数确保输入是具有相应 ZKPoK 的有效密文。
2. **类型转换**：
   该函数将 `externalEbool`、`externalEaddress`、`externalEuintXX` 转换为适当的加密类型（`ebool`、`eaddress`、`euintXX`），以便在合约内进行进一步操作。

## 最佳实践

- **输入打包**：通过将所有加密输入打包到单个密文中，最大限度地减小零知识证明的大小和复杂性。
- **前端加密**：始终在客户端使用 FHE 公钥加密输入，以确保数据机密性。
- **证明管理**：确保将正确的零知识证明与每个加密输入相关联，以避免验证错误。

加密输入及其验证构成了 FHEVM 中安全和私密交互的基础。通过利用这些工具，开发人员可以创建强大、保护隐私的智能合约，而不会影响功能或可伸缩性。
