# 配置

本文档介绍了如何通过设置 `fhevm` 环境在您的智能合约中启用加密计算。了解如何集成必要的库、配置加密以及向您的合约添加安全计算逻辑。

## 核心配置设置

要在 Solidity 合约中使用加密计算，您必须配置 **FHE 库**和**预言机地址**。`fhevm` 包通过预构建的配置合约简化了此过程，让您可以专注于开发合约的逻辑，而无需处理底层的加密设置。

该库及其相关的合约提供了一种标准化的方式来配置和与 Zama 的 FHEVM（全同态加密虚拟机）基础设施在不同的以太坊网络上进行交互。它为 Zama 的 FHEVM 组件（`ACL`、`FHEVMExecutor`、`KMSVerifier`、`InputVerifier`）和解密预言机提供了必要的合约地址，为需要 FHEVM 支持的 Solidity 合约实现了无缝集成。

## 自动配置的关键组件

1. **FHE 库**：设置加密参数和加密密钥。
2. **预言机**：管理安全的加密操作，例如公共解密。
3. **特定于网络的设置**：适应本地测试、测试网（例如 Sepolia）或主网部署。

通过继承这些配置合约，您可以确保跨环境的无缝初始化和功能。

## ZamaConfig.sol

`ZamaConfig` 库公开了用于检索受支持网络（目前仅 Sepolia 测试网）的 FHEVM 配置结构和预言机地址的函数。

在底层，该库将 Zama FHEVM 基础设施的特定于网络的地址封装到一个结构（`FHEVMConfigStruct`）中。

## SepoliaConfig

`SepoliaConfig` 合约旨在由用户合约继承。构造函数使用库为相应网络提供的配置自动设置 FHEVM 协处理器和解密预言机。当一个合约继承自 `SepoliaConfig` 时，构造函数会使用适当的地址调用 `FHE.setCoprocessor` 和 `FHE.setDecryptionOracle`。这确保了继承的合约自动连接到目标网络的正确 FHEVM 合约和预言机，从而抽象出手动地址管理并降低了配置错误的风险。

**示例：使用 Sepolia 配置**

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

contract MyERC20 is SepoliaConfig {
  constructor() {
    // 如果需要，可添加额外的初始化逻辑
  }
}
```

## 使用 `isInitialized`

`isInitialized` 实用函数检查加密变量是否已正确初始化，从而防止由于未初始化的值而导致的意外行为。

**函数签名**

```solidity
function isInitialized(T v) internal pure returns (bool)
```

**目的**

- 确保加密变量在使用前已初始化。
- 防止合约执行中潜在的逻辑错误。

**示例：加密计数器的初始化检查**

```solidity
require(FHE.isInitialized(counter), "Counter not initialized!");
```

## 总结

通过利用 `ZamaConfig.sol` 中的预构建配置合约（如 `SepoliaConfig`），您可以高效地为您的智能合约设置加密计算。这些工具抽象了加密初始化的复杂性，让您可以专注于构建安全、机密的智能合约。
