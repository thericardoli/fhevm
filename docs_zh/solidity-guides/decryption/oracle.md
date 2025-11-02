# 预言机

本节介绍了如何在 fhevm 中处理解密。解密允许在合约逻辑或用户表示需要时访问明文数据，确保在整个过程中保持机密性。

解密在两种主要情况下至关重要：

1. **智能合约逻辑**：合约需要明文值进行计算或决策。
2. **用户交互**：需要向所有用户显示明文数据，例如显示投票决定。

## 概述

FHEVM 中的解密是一个异步过程，涉及中继器和密钥管理系统 (KMS)。以下是如何在合约中安全地请求解密的示例。

### 示例：合约中的异步解密

```solidity
pragma solidity ^0.8.24;

import "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

contract TestAsyncDecrypt is SepoliaConfig {
  ebool xBool;
  bool public yBool;
  bool isDecryptionPending;
  uint256 latestRequestId;

  constructor() {
    xBool = FHE.asEbool(true);
    FHE.allowThis(xBool);
  }

  function requestBool() public {
    require(!isDecryptionPending, "Decryption is in progress");
    bytes32[] memory cts = new bytes32[](1);
    cts[0] = FHE.toBytes32(xBool);
    uint256 latestRequestId = FHE.requestDecryption(cts, this.myCustomCallback.selector);

    /// @dev This prevents sending multiple requests before the first callback was sent.
    isDecryptionPending = true;
  }

  function myCustomCallback(uint256 requestId, bytes memory cleartexts, bytes memory decryptionProof) public returns (bool) {
    /// @dev This check is used to verify that the request id is the expected one.
    require(requestId == latestRequestId, "Invalid requestId");
    FHE.checkSignatures(requestId, cleartexts, decryptionProof);

    (bool decryptedInput) = abi.decode(cleartexts, (bool));
    yBool = decryptedInput;
    isDecryptionPending = false;
    return yBool;
  }
}
```

## 深入解密

本文档提供了有关使用 fhevm 中的 `DecryptionOracle` 在您的智能合约中实现解密的详细指南。它涵盖了设置、`FHE.requestDecryption` 函数的使用以及使用 Hardhat 进行测试。

## `DecryptionOracle` 设置

`DecryptionOracle` 已预先部署在 FHEVM 测试网上。它使用 `.env` 文件中指定的默认中继器帐户。

任何人都可以满足解密请求，但必须添加签名验证（并包含一个使解密请求的重放无效的逻辑）。`DecryptionOracle` 合约的作用是在执行期间独立验证 KMS 签名。这确保了即使中继器遭到破坏，也无法操纵或发送欺诈性的解密结果。

有两个函数需要考虑：`requestDecryption` 和 `checkSignatures`。

### `FHE.requestDecryption` 函数

您可以这样调用 `FHE.requestDecryption` 函数：

```solidity
function requestDecryption(
  bytes32[] calldata ctsHandles,
  bytes4 callbackSelector
) external payable returns (uint256 requestId);
```

#### 函数参数

第一个参数 `ctsHandles` 应该是一个密文句柄数组，可以是不同类型的，即来自解包类型为 `ebool`、`euint8`、`euint16`、`euint32`、`euint64` 或 `eaddress` 的句柄的 `uint256` 值。

`ctsHandles` 是请求解密的密文数组。中继器将在满足请求之前将相应的密文发送到 KMS 进行解密。

`callbackSelector` 是回调函数的函数选择器，一旦中继器满足解密请求，就会调用该函数。

```solidity
function [callbackName](uint256 requestID, bytes memory cleartexts, bytes memory decryptionProof) external;
```

`cleartexts` 是对应于所有请求的解密值的 ABI 编码的字节数组。这些解密值的每个类型都应该是与原始密文类型对应的本机 Solidity 类型，遵循此约定表：

| 密文类型 | 解密类型 |
| --- | --- |
| ebool | bool |
| euint8 | uint8 |
| euint16 | uint16 |
| euint32 | uint32 |
| euint64 | uint64 |
| euint128 | uint128 |
| euint256 | uint256 |
| eaddress | address |

这里的 `callbackName` 是开发人员为回调函数指定的自定义名称，`requestID` 将是解密的请求 ID（如果在逻辑中不需要，可以注释掉，但必须存在），`cleartexts` 是 `ct` 数组值解密结果的 ABI 编码字节数组，即它们的数量应为 `ct` 数组的大小。`decryptionProof` 是包含 KMS 签名和额外数据的字节数组。

`msgValue` 是在满足期间发送到调用合约的本机代币值，即当使用解密结果调用回调时。

{% hint style="warning" %}
请注意，回调应始终验证签名并实现重放保护机制（见下文）。
{% endhint %}

### `FHE.checkSignatures` 函数

您可以这样调用 `FHE.checkSignatures` 函数：

```solidity
function checkSignatures(uint256 requestId, bytes memory cleartexts, bytes[] memory signatures);
```

#### 函数参数

- `requestID`，是在 `requestDecryption` 函数中返回的值。
- `cleartexts`，是与句柄关联的解密值的 ABI 编码（使用 `abi.encode`）。这可以包含一个或多个值，具体取决于在 `requestDecryption` 函数中请求的句柄数量。这些值的每个类型都必须与相应句柄的类型匹配。
- `decryptionProof`，是包含 KMS 签名和额外数据的字节数组。

如果签名无效，此函数将回滚。
