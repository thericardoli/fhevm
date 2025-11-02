# 将其转换为 FHEVM

在本教程中，您将学习如何使用 Zama 的 FHEVM 库将一个基本的 Solidity 智能合约逐步升级以支持全同态加密。

从您在[“编写一个简单的合约”教程](write_a_simple_contract.md)中构建的普通 `Counter.sol` 合约开始，您将逐步学习如何：

- 用加密等效项替换标准类型
- 集成零知识证明验证
- 启用加密的链上计算
- 授予安全链下解密的权限

到最后，您将拥有一个功能齐全的智能合约，支持 FHE 计算。

## 初始化合约

{% stepper %} {% step %}

## 创建 `FHECounter.sol` 文件

导航到您项目的 `contracts` 目录：

```sh
cd <your-project-root-directory>/contracts
```

在那里，创建一个名为 `FHECounter.sol` 的新文件，并将以下 Solidity 代码复制到其中：

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

/// @title 一个简单的计数器合约
contract Counter {
  uint32 private _count;

  /// @notice 返回当前计数值
  function getCount() external view returns (uint32) {
    return _count;
  }

  /// @notice 将计数器增加一个指定值
  function increment(uint32 value) external {
    _count += value;
  }

  /// @notice 将计数器减去一个指定值
  function decrement(uint32 value) external {
    require(_count >= value, "Counter: cannot decrement below zero");
    _count -= value;
  }
}
```

这是一个普通的 `Counter` 合约，我们将用它作为添加 FHEVM 功能的起点。我们将逐步修改此合约以逐步集成 FHEVM 功能。 {% endstep %}

{% step %}

## 将 `Counter` 转换为 `FHECounter`

要开始将 FHEVM 功能集成到您的合约中，我们首先需要导入所需的 FHEVM 库。

#### 替换当前标题

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;
```

#### 使用此更新后的标题：

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, euint32, externalEuint32 } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";
```

这些导入：

- **FHE** — 使用 FHEVM 加密类型的核心库
- **euint32** 和 **externalEuint32** — FHEVM 中使用的加密 uint32 类型
- **SepoliaConfig** — 为 Sepolia 网络提供 FHEVM 配置。
  继承它使您的合约能够使用 FHE 库

#### 替换当前合约声明：

```solidity
/// @title 一个简单的计数器合约
contract Counter {
```

#### 使用更新后的声明：

```solidity
/// @title 一个简单的 FHE 计数器合约
contract FHECounter is SepoliaConfig {
```

此更改：

- 将合约重命名为 `FHECounter`
- 继承自 `SepoliaConfig` 以启用 FHEVM 支持

{% hint style="warning" %}
此合约必须继承自 `SepoliaConfig` 抽象合约；否则，它将无法在 Sepolia 或 Hardhat 上执行任何与 FHEVM 相关的功能。
{% endhint %}

从您项目的根目录运行：

```sh
npx hardhat compile
```

太棒了！您的智能合约现已编译并准备好使用 **FHEVM 功能。**

{% endstep %} {% endstepper %}

## 应用 FHE 函数和类型

{% stepper %} {% step %}

## 注释掉 `increment()` 和 `decrement()` 函数

在我们继续之前，让我们注释掉 `FHECounter` 中的 `increment()` 和 `decrement()` 函数。我们稍后会用支持 FHE 加密操作的更新版本替换它们。

```solidity
 /// @notice 将计数器增加一个指定值
// function increment(uint32 value) external {
//     _count += value;
// }

/// @notice 将计数器减去一个指定值
// function decrement(uint32 value) external {
//     require(_count >= value, "Counter: cannot decrement below zero");
//     _count -= value;
// }
```

{% endstep %}

{% step %}

## 将 `uint32` 替换为 FHEVM `euint32` 类型

我们现在将从标准的 Solidity `uint32` 类型切换到加密的 FHEVM 类型 `euint32`。

这使得可以在加密整数上进行私密的同态计算。

#### 替换

```solidity
uint32 _count;
```

和

```solidity
function getCount() external view returns (uint32) {
```

#### 使用：

```solidity
euint32 _count;
```

和

```solidity
function getCount() external view returns (euint32) {
```

{% endstep %}

{% step %}

## 将 `increment(uint32 value)` 替换为 FHEVM 版本 `increment(externalEuint32 value)`

为了支持加密输入，我们将更新 increment 函数以接受在链下加密的值。

新版本将接受 `externalEuint32`，而不是使用 `uint32`，这是一个在链下生成并发送到智能合约的加密整数。

为了确保此加密值的有效性，我们还包含第二个参数：`inputProof`，一个包含零知识知识证明 (ZKPoK) 的字节数组，该证明证明了两件事：

1. `externalEuint32` 是由函数调用者（`msg.sender`）在链下加密的
2. `externalEuint32` 绑定到合约（`address(this)`）并且只能由它处理。

#### 替换

```solidity
 /// @notice 将计数器增加一个指定值
// function increment(uint32 value) external {
//     _count += value;
// }
```

#### 使用：

```solidity
/// @notice 将计数器增加一个指定值
function increment(externalEuint32 inputEuint32, bytes calldata inputProof) external {
  //     _count += value;
}
```

{% endstep %}

{% step %}

## 将 `externalEuint32` 转换为 `euint32`

您不能直接在 FHE 操作中使用 `externalEuint32`。要使用 FHEVM 库对其进行操作，您首先需要将其转换为本机 FHE 类型 `euint32`。

此转换使用：

```solidity
FHE.fromExternal(inputEuint32, inputProof);
```

此方法验证零知识证明并在合约内返回一个可用的加密值。

#### 替换

```solidity
/// @notice 将计数器增加一个指定值
function increment(externalEuint32 inputEuint32, bytes calldata inputProof) external {
  //     _count += value;
}
```

#### 使用：

```solidity
/// @notice 将计数器增加一个指定值
function increment(externalEuint32 inputEuint32, bytes calldata inputProof) external {
  euint32 evalue = FHE.fromExternal(inputEuint32, inputProof);
  //     _count += value;
}
```

{% endstep %}

{% step %}

## 将 `_count += value` 转换为其 FHEVM 等效项

要以完全同态的方式执行更新 `_count += value`，我们使用 `FHE.add()` 运算符。此函数允许我们计算 2 个加密整数的 FHE 和。

#### 替换

```solidity
/// @notice 将计数器增加一个指定值
function increment(externalEuint32 inputEuint32, bytes calldata inputProof) external {
  euint32 evalue = FHE.fromExternal(inputEuint32, inputProof);
  //     _count += value;
}
```

#### 使用：

```solidity
/// @notice 将计数器增加一个指定值
function increment(externalEuint32 inputEuint32, bytes calldata inputProof) external {
  euint32 evalue = FHE.fromExternal(inputEuint32, inputProof);
  _count = FHE.add(_count, evalue);
}
```

{% hint style="info" %}
此 FHE 操作允许智能合约处理加密值而无需解密它们——这是 FHEVM 启用链上隐私的核心功能。
{% endhint %}

{% endstep %} {% endstepper %}

## 授予 FHE 权限

{% hint style="warning" %}
此步骤至关重要！您必须向合约和调用者授予 FHE 权限，以确保加密的 `_count` 值可以由调用者在链下解密。没有这两个权限，调用者将无法计算明文结果。
{% endhint %}

要授予 FHE 权限，我们将调用 `FHE.allow()` 函数。

#### 替换

```solidity
/// @notice 将计数器增加一个指定值
function increment(externalEuint32 inputEuint32, bytes calldata inputProof) external {
  euint32 evalue = FHE.fromExternal(inputEuint32, inputProof);
  _count = FHE.add(_count, evalue);
}
```

#### 使用：

```solidity
/// @notice 将计数器增加一个指定值
function increment(externalEuint32 inputEuint32, bytes calldata inputProof) external {
  euint32 evalue = FHE.fromExternal(inputEuint32, inputProof);
  _count = FHE.add(_count, evalue);

  FHE.allowThis(_count);
  FHE.allow(_count, msg.sender);
}
```

{% hint style="info" %}
我们在这里授予**两个** FHE 权限——而不仅仅是一个。在本教程的下一部分中，您将了解为什么**两者**都是必需的。
{% endhint %}

## 将 `decrement()` 转换为其 FHEVM 等效项

就像 `increment()` 迁移一样，我们现在将 `decrement()` 函数转换为其与 FHEVM 兼容的版本。

替换：

```solidity
/// @notice 将计数器减去一个指定值
function decrement(uint32 value) external {
  require(_count >= value, "Counter: cannot decrement below zero");
  _count -= value;
}
```

替换为以下内容：

```solidity
/// @notice 将计数器减去一个指定值
/// @dev 为简单起见和可读性，本示例省略了溢出/下溢检查。
/// 在生产合约中，应实现适当的范围检查。
function decrement(externalEuint32 inputEuint32, bytes calldata inputProof) external {
  euint32 encryptedEuint32 = FHE.fromExternal(inputEuint32, inputProof);

  _count = FHE.sub(_count, encryptedEuint32);

  FHE.allowThis(_count);
  FHE.allow(_count, msg.sender);
}
```

{% hint style="warning" %}
`increment()` 和 `decrement()` 函数不执行任何溢出或下溢检查。
{% endhint %}

## 编译 `FHECounter.sol`

从您项目的根目录运行：

```sh
npx hardhat compile
```

恭喜！您的智能合约现在完全**与 FHEVM 兼容**。

现在您的项目中应该有以下文件：

- [`contracts/FHECounter.sol`](https://docs.zama.ai/protocol/examples/basic/fhe-counter#fhecounter.sol) — 您的 Solidity 智能 FHEVM 合约
- [`test/FHECounter.ts`](https://docs.zama.ai/protocol/examples/basic/fhe-counter#fhecounter.ts) — 您用 TypeScript 编写的 FHEVM Hardhat 测试套件

在[下一个教程](test_fhevm_contract.md)中，我们将继续进行 **TypeScript 集成**，您将学习如何在测试套件中与新升级的 FHEVM 合约进行交互。
