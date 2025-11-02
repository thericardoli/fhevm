# FHE 库

本文档对 **FHEVM 库**进行了高级概述，帮助您了解它如何融入更广泛的 Zama 协议。要了解如何在实践中使用它，请参阅[Solidity 指南](https://docs.zama.ai/protocol/solidity-guides)。

## 什么是 FHEVM 库？

FHEVM 库使开发人员能够构建对加密数据进行操作的智能合约——而无需任何密码学知识。

它通过以下方式扩展了标准的 Solidity 开发流程：

- 加密数据类型
- 对加密值的算术、逻辑和条件运算
- 精细的访问控制
- 安全的输入处理和证明支持

该库充当全同态加密（FHE）的**抽象层**，并与**协处理器**和**网关**等链下组件无缝交互。

## 主要功能

### 加密数据类型

该库引入了常见 Solidity 类型的加密变体，实现为用户定义的值类型。在内部，这些表示为指向链下存储的加密值的 `bytes32` 句柄。

| 类别 | 类型 |
| --- | --- |
| 布尔值 | `ebool` |
| 无符号整数 | `euint8`、`euint16`、...、`euint256` |
| 有符号整数 | `eint8`、`eint16`、...、`eint256` |
| 地址 | `eaddress` |

→ 查看[加密数据类型](https://docs.zama.ai/protocol/solidity-guides/smart-contract/types)的完整指南。

### FHE 操作

每个加密类型都支持与其明文对应物类似的操作：

- 算术：`add`、`sub`、`mul`、`div`、`rem`、`neg`
- 逻辑：`and`、`or`、`xor`、`not`
- 比较：`lt`、`gt`、`le`、`ge`、`eq`、`ne`、`min`、`max`
- 位操作：`shl`、`shr`、`rotl`、`rotr`

这些操作通过生成新的句柄并在链上发出事件来象征性地执行，以便协处理器在链下处理实际的 FHE 计算。

示例：

```solidity
function compute(euint64 x, euint64 y, euint64 z) public returns (euint64) {
  euint64 result = FHE.mul(FHE.add(x, y), z);
  return result;
}
```

→ 查看[对加密类型的操作](https://docs.zama.ai/protocol/solidity-guides/smart-contract/operations)的完整指南。

### 使用加密条件进行分支

直接的 if 或 require 语句与加密布尔值不兼容。相反，该库提供了一个 `select` 运算符来模拟条件逻辑，而不会泄露采取了哪个分支：

```solidity
ebool condition = FHE.lte(x, y);
euint64 result = FHE.select(condition, valueIfTrue, valueIfFalse);
```

即使在条件逻辑中，这也保留了机密性。

→ 查看[分支](https://docs.zama.ai/protocol/solidity-guides/smart-contract/logics/conditions)的完整指南。

### 处理外部加密输入

当用户想要传递加密输入（例如，他们在链下加密的值或从另一个链桥接的值）时，他们提供：

- 外部值
- 协处理器签名列表（证明）

`fromExternal` 函数用于验证证明并提取可用的加密句柄：

```solidity
function handleInput(externalEuint64 param1, externalEbool param2, bytes calldata attestation) public {
  euint64 val = FHE.fromExternal(param1, attestation);
  ebool flag = FHE.fromExternal(param2, attestation);
}
```

这确保只有授权的、格式正确的密文才能被智能合约接受。

→ 查看[加密输入](https://docs.zama.ai/protocol/solidity-guides/smart-contract/inputs)的完整指南。

### 访问控制

FHE 库还公开了使用主机合约维护的 ACL 管理对加密值的访问的方法：

- `allow(handle, address)`：授予永久访问权限
- `allowTransient(handle, address)`：仅授予当前交易的访问权限
- `allowForDecryption(handle)`：使句柄可公开解密
- `isAllowed(handle, address)`：检查地址是否具有访问权限
- `isSenderAllowed(handle)`：检查 msg.sender 权限的快捷方式

这些 `allow` 方法会发出由协处理器使用的事件，以在网关中复制 ACL 状态。

→ 查看[ACL](https://docs.zama.ai/protocol/solidity-guides/smart-contract/acl)的完整指南。

### 伪随机加密值

该库允许生成伪随机加密整数，可用于游戏、彩票或随机逻辑：

- `randEuintXX()`
- `randEuintXXBounded`(uint bound)

这些在协处理器之间是确定性的，并且对于外部观察者来说是无法区分的。

→ 查看[生成随机数](https://docs.zama.ai/protocol/solidity-guides/smart-contract/operations/random)的完整指南。
