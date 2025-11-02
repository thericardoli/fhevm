# 迁移

本文档提供了从 FHEVM v0.6 迁移到 v0.7 的说明。

## 从 0.6.x 开始

### 包和库

包现在是 `@fhevm/solidity` 而不是 `FHEVM`，库名称已从 `TFHE` 更改为 `FHE`

```solidity
import { FHE } from "@fhevm/solidity";
```

### 配置

配置已从 `SepoliaZamaConfig` 重命名为 `SepoliaConfig`。

```solidity
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";
```

此外，手动定义协处理器的函数已从 `setFHEVM` 重命名为 `setCoprocessor`，定义预言机的函数现在已集成到 `setCoprocessor` 中。

```solidity
import { ZamaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";
constructor () {
  FHE.setCoprocessor(ZamaConfig.getSepoliaConfig());
}
```

您可以在[专门的配置页面](configure.md)上阅读有关配置的更多信息。

### 解密预言机

以前，使用抽象合约 `GatewayCaller` 来请求解密。它已被 `FHE.requestDecryption` 取代：

```solidity
function requestBoolInfinite() public {
  bytes32[] memory cts = new bytes32[](1);
  cts[0] = FHE.toBytes32(myEncryptedValue);
  FHE.requestDecryption(cts, this.myCallback.selector);
}
```

您可以在[专门的解密预言机页面](decryption/oracle.md)上阅读有关解密预言机的更多信息。

### ebytes 的弃用

`ebytes` 已被弃用并从 FHEVM 中删除。

### 区块 gas 限制

区块 gas 限制已被 HCU（同态复杂度单位）限制取代。FHEVM 0.7.0 包括两个限制：

- **每个交易的顺序同态操作深度限制**：控制必须按顺序处理的操作的 HCU 使用情况。此限制设置为 **5,000,000** HCU。
- **每个交易的全局同态操作复杂度**：控制可以并行处理的操作的 HCU 使用情况。此限制设置为 **20,000,000** HCU。

您可以在[专门的 HCU 页面](hcu.md)上阅读有关 HCU 的更多信息。
