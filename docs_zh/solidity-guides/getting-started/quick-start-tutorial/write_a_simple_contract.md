# 编写一个简单的合约

在本教程中，您将在 FHEVM Hardhat 模板中编写并测试一个简单的常规 Solidity 智能合约，以熟悉 Hardhat 的工作流程。

在[下一个教程](turn_it_into_fhevm.md)中，您将学习如何将此合约转换为 FHEVM 合约。

## 先决条件

- [设置您的 Hardhat 环境](setup.md)。
- 确保您的 Hardhat 项目是干净的并且可以开始。请参阅[此处](setup.md#rest-set-the-hardhat-envrionment)的说明。

## 您将学到什么

在本教程结束时，您将学习如何：

- 使用 Hardhat 编写一个最小的 Solidity 合约。
- 使用 TypeScript 和 Hardhat 的测试框架测试合约。

## 编写一个简单的合约

{% stepper %} {% step %}

## 创建 `Counter.sol`

转到您项目的 `contracts` 目录：

```sh
cd <your-project-root-directory>/contracts
```

在那里，创建一个名为 `Counter.sol` 的新文件，并将以下 Solidity 代码复制并粘贴到其中。

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

{% endstep %}

{% step %}

## 编译 `Counter.sol`

从您项目的根目录运行：

```sh
npx hardhat compile
```

太棒了！您的智能合约现已编译。 {% endstep %} {% endstepper %}

## 设置测试环境

{% stepper %} {% step %}

## 创建测试脚本 `test/Counter.ts`

转到您项目的 `test` 目录

```sh
cd <your-project-root-directory>/test
```

在那里，创建一个名为 `Counter.ts` 的新文件，并将以下 Typescript 骨架代码复制并粘贴到其中。

```ts
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { ethers } from "hardhat";

describe("Counter", function () {
  it("empty test", async function () {
    console.log("Cool! The test basic skeleton is running!");
  });
});
```

该文件包含以下内容：

- 我们在各种测试中需要的所有必需的 `import` 语句
- 用于运行名为 `empty test` 的第一个空测试的 `chai` 基本语句 {% endstep %}

{% step %}

## 运行测试 `test/Counter.ts`

从您项目的根目录运行：

```sh
npx hardhat test
```

输出：

```sh
  Counter
Cool! The test basic skeleton is running!
    ✔ empty test


  1 passing (1ms)
```

太棒了！您的 Hardhat 测试环境已正确设置。

{% endstep %}

{% step %}

## 设置测试签名者

在 Hardhat 测试中与智能合约交互之前，我们需要初始化签名者。

{% hint style="info" %}
在以太坊开发中，签名者代表一个可以发送交易和签署消息的实体（通常是一个钱包）。在 Hardhat 中，`ethers.getSigners()` 返回一个预先注资的测试帐户列表。
{% endhint %}

为方便起见，我们将定义三个命名的签名者：

- `owner` — 合约的部署者
- `alice` 和 `bob` — 其他模拟用户

#### 将 `test/Counter.ts` 的内容替换为以下内容：

```ts
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { ethers } from "hardhat";

type Signers = {
  owner: HardhatEthersSigner;
  alice: HardhatEthersSigner;
  bob: HardhatEthersSigner;
};

describe("Counter", function () {
  let signers: Signers;

  before(async function () {
    const ethSigners: HardhatEthersSigner[] = await ethers.getSigners();
    signers = { owner: ethSigners[0], alice: ethSigners[1], bob: ethSigners[2] };
  });

  it("should work", async function () {
    console.log(`address of user owner is ${signers.owner.address}`);
    console.log(`address of user alice is ${signers.alice.address}`);
    console.log(`address of user bob is ${signers.bob.address}`);
  });
});
```

#### 运行测试

从您项目的根目录运行：

```sh
npx hardhat test
```

**预期输出**

```sh
  Counter
address of user owner is 0x37AC010c1c566696326813b840319B58Bb5840E4
address of user alice is 0xD9F9298BbcD72843586e7E08DAe577E3a0aC8866
address of user bob is 0x3f0CdAe6ebd93F9F776BCBB7da1D42180cC8fcC1
    ✔ should work


  1 passing (2ms)
```

{% endstep %}

{% step %}

## 设置测试实例

现在我们已经设置了签名者，我们可以部署智能合约了。

为了确保测试的隔离性和确定性，我们应该在每个测试之前部署一个新的 `Counter.sol` 实例。这可以避免先前测试的任何副作用。

标准方法是定义一个处理合约部署的 `deployFixture()` 函数。

```ts
async function deployFixture() {
  const factory = (await ethers.getContractFactory("Counter")) as Counter__factory;
  const counterContract = (await factory.deploy()) as Counter;
  const counterContractAddress = await counterContract.getAddress();

  return { counterContract, counterContractAddress };
}
```

要在每个测试用例之前运行此设置，请在 `beforeEach` 块中调用 `deployFixture()`：

```ts
beforeEach(async () => {
  ({ counterContract, counterContractAddress } = await deployFixture());
});
```

这确保每个测试都使用一个干净、独立的合约实例运行。

让我们把它放在一起。现在您的`test/Counter.ts` 应该如下所示：

```ts
import { Counter, Counter__factory } from "../types";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";

type Signers = {
  deployer: HardhatEthersSigner;
  alice: HardhatEthersSigner;
  bob: HardhatEthersSigner;
};

async function deployFixture() {
  const factory = (await ethers.getContractFactory("Counter")) as Counter__factory;
  const counterContract = (await factory.deploy()) as Counter;
  const counterContractAddress = await counterContract.getAddress();

  return { counterContract, counterContractAddress };
}

describe("Counter", function () {
  let signers: Signers;
  let counterContract: Counter;
  let counterContractAddress: Counter;

  before(async function () {
    const ethSigners: HardhatEthersSigner[] = await ethers.getSigners();
    signers = { deployer: ethSigners[0], alice: ethSigners[1], bob: ethSigners[2] };
  });

  beforeEach(async () => {
    // 在每个测试之前部署一个新的合约实例
    ({ counterContract, counterContractAddress } = await deployFixture());
  });

  it("should be deployed", async function () {
    console.log(`Counter has been deployed at address ${counterContractAddress}`);
    // 测试已部署的地址是否有效
    expect(ethers.isAddress(counterContractAddress)).to.eq(true);
  });
});
```

**运行测试：**

从您项目的根目录运行：

```sh
npx hardhat test
```

#### 预期输出：

```sh
  Counter
Counter has been deployed at address 0x7553CB9124f974Ee475E5cE45482F90d5B6076BC
    ✔ should be deployed


  1 passing (7ms)
```

{% endstep %} {% endstepper %}

## 测试函数

现在一切都已启动并运行，您可以开始测试您的合约函数了。

{% stepper %} {% step %}

## 调用合约 `getCount()` 视图函数

一切都已启动并运行，我们现在可以调用 `Counter.sol` 视图函数 `getCount()`！

就在测试块 `it("should be deployed", async function () {...}` 下面，

添加以下单元测试：

```ts
it("count should be zero after deployment", async function () {
  const count = await counterContract.getCount();
  console.log(`Counter.getCount() === ${count}`);
  // 期望部署后初始计数为 0
  expect(count).to.eq(0);
});
```

#### 运行测试

从您项目的根目录运行：

```sh
npx hardhat test
```

#### 预期输出

```sh
  Counter
Counter has been deployed at address 0x7553CB9124f974Ee475E5cE45482F90d5B6076BC
    ✔ should be deployed
Counter.getCount() === 0
    ✔ count should be zero after deployment


  1 passing (7ms)
```

{% endstep %}

{% step %}

## 调用合约 `increment()` 交易函数

就在测试块 `it("count should be zero after deployment", async function () {...}` 下面，添加以下测试块：

```ts
it("increment the counter by 1", async function () {
  const countBeforeInc = await counterContract.getCount();
  const tx = await counterContract.connect(signers.alice).increment(1);
  await tx.wait();
  const countAfterInc = await counterContract.getCount();
  expect(countAfterInc).to.eq(countBeforeInc + 1n);
});
```

#### 备注：

- `increment()` 是一个修改区块链状态的交易函数。
- 它必须由用户签名——这里我们使用 `alice`。
- `await wait()` 等待交易被挖掘。
- 测试在交易前后比较计数器以确保其按预期递增。

#### 运行测试

从您项目的根目录运行：

```sh
npx hardhat test
```

#### 预期输出

```sh
  Counter
Counter has been deployed at address 0x7553CB9124f974Ee475E5cE45482F90d5B6076BC
    ✔ should be deployed
Counter.getCount() === 0
    ✔ count should be zero after deployment
    ✔ increment the counter by 1


  2 passing (12ms)
```

{% endstep %}

{% step %}

## 调用合约 `decrement()` 交易函数

就在测试块 `it("increment the counter by 1", async function () {...}` 下面，

添加以下测试块：

```ts
it("decrement the counter by 1", async function () {
  // 首先递增，计数变为 1
  let tx = await counterContract.connect(signers.alice).increment(1);
  await tx.wait();
  // 然后递减，计数回到 0
  tx = await counterContract.connect(signers.alice).decrement(1);
  await tx.wait();
  const count = await counterContract.getCount();
  expect(count).to.eq(0);
});
```

---

#### 运行测试

从您项目的根目录运行：

```sh
npx hardhat test
```

#### 预期输出

```sh
  Counter
Counter has been deployed at address 0x7553CB9124f974Ee475E5cE45482F90d5B6076BC
    ✔ should be deployed
Counter.getCount() === 0
    ✔ count should be zero after deployment
    ✔ increment the counter by 1
    ✔ decrement the counter by 1


  2 passing (12ms)
```

{% endstep %} {% endstepper %}

现在您已成功编写并测试了您的计数器合约。您的项目中应该有以下文件：

- [`contracts/Counter.sol`](https://docs.zama.ai/protocol/examples/basic/fhe-counter#counter.sol) — 您的 Solidity 智能合约
- [`test/Counter.ts`](https://docs.zama.ai/protocol/examples/basic/fhe-counter#counter.ts) — 您用 TypeScript 编写的 Hardhat 测试套件

这些文件构成了基于 Hardhat 的基本智能合约项目的基础。

## 下一步

现在您已经编写并测试了一个基本的 Solidity 智能合约，您已准备好迈出下一步。

在[下一个教程](turn_it_into_fhevm.md)中，我们将把这个标准的 `Counter.sol` 合约转换为 `FHECounter.sol`，一个简单的 FHEVM 兼容版本——允许使用简单的全同态加密来存储和更新计数器值。
