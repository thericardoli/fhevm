本示例演示了如何使用 FHEVM 构建一个机密计数器，并与一个简单计数器进行比较。

{% hint style="info" %}
为正确运行此示例，请确保将文件放置在以下目录中：

- `.sol` 文件 → `<your-project-root-dir>/contracts/`
- `.ts` 文件 → `<your-project-root-dir>/test/`

这能确保 Hardhat 能够按预期编译和测试您的合约。
{% endhint %}

## 一个简单计数器

{% tabs %}

{% tab title="counter.sol" %}

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

  /// @notice 将计数器增加 1
  function increment(uint32 value) external {
    _count += value;
  }

  /// @notice 将计数器减去 1
  function decrement(uint32 value) external {
    require(_count > value, "Counter: cannot decrement below zero");
    _count -= value;
  }
}
```

{% endtab %}

{% tab title="counter.ts" %}

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

  before(async function () {
    const ethSigners: HardhatEthersSigner[] = await ethers.getSigners();
    signers = { deployer: ethSigners[0], alice: ethSigners[1], bob: ethSigners[2] };
  });

  beforeEach(async () => {
    ({ counterContract } = await deployFixture());
  });

  it("部署后计数值应为零", async function () {
    const count = await counterContract.getCount();
    console.log(`Counter.getCount() === ${count}`);
    // 部署后初始计数值应为 0
    expect(count).to.eq(0);
  });

  it("计数器加 1", async function () {
    const countBeforeInc = await counterContract.getCount();
    const tx = await counterContract.connect(signers.alice).increment(1);
    await tx.wait();
    const countAfterInc = await counterContract.getCount();
    expect(countAfterInc).to.eq(countBeforeInc + 1n);
  });

  it("计数器减 1", async function () {
    // 首先加 1，计数值变为 1
    let tx = await counterContract.connect(signers.alice).increment();
    await tx.wait();
    // 然后减 1，计数值回到 0
    tx = await counterContract.connect(signers.alice).decrement(1);
    await tx.wait();
    const count = await counterContract.getCount();
    expect(count).to.eq(0);
  });
});
```

{% endtab %}

{% endtabs %}

## 一个 FHE 计数器

{% tabs %}

{% tab title="FHECounter.sol" %}

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, euint32, externalEuint32 } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

/// @title 一个简单的 FHE 计数器合约
contract FHECounter is SepoliaConfig {
  euint32 private _count;

  /// @notice 返回当前计数值
  function getCount() external view returns (euint32) {
    return _count;
  }

  /// @notice 将计数器增加一个指定的加密值。
  /// @dev 为简单易读，本示例省略了溢出/下溢检查。
  /// 在生产合约中，应实现适当的范围检查。
  function increment(externalEuint32 inputEuint32, bytes calldata inputProof) external {
    euint32 encryptedEuint32 = FHE.fromExternal(inputEuint32, inputProof);

    _count = FHE.add(_count, encryptedEuint32);

    FHE.allowThis(_count);
    FHE.allow(_count, msg.sender);
  }

  /// @notice 将计数器减去一个指定的加密值。
  /// @dev 为简单易读，本示例省略了溢出/下溢检查。
  /// 在生产合约中，应实现适当的范围检查。
  function decrement(externalEuint32 inputEuint32, bytes calldata inputProof) external {
    euint32 encryptedEuint32 = FHE.fromExternal(inputEuint32, inputProof);

    _count = FHE.sub(_count, encryptedEuint32);

    FHE.allowThis(_count);
    FHE.allow(_count, msg.sender);
  }
}
```

{% endtab %}

{% tab title="FHECounter.ts" %}

```ts
import { FHECounter, FHECounter__factory } from "../types";
import { FhevmType } from "@fhevm/hardhat-plugin";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers, fhevm } from "hardhat";

type Signers = {
  deployer: HardhatEthersSigner;
  alice: HardhatEthersSigner;
  bob: HardhatEthersSigner;
};

async function deployFixture() {
  const factory = (await ethers.getContractFactory("FHECounter")) as FHECounter__factory;
  const fheCounterContract = (await factory.deploy()) as FHECounter;
  const fheCounterContractAddress = await fheCounterContract.getAddress();

  return { fheCounterContract, fheCounterContractAddress };
}

describe("FHECounter", function () {
  let signers: Signers;
  let fheCounterContract: FHECounter;
  let fheCounterContractAddress: string;

  before(async function () {
    const ethSigners: HardhatEthersSigner[] = await ethers.getSigners();
    signers = { deployer: ethSigners[0], alice: ethSigners[1], bob: ethSigners[2] };
  });

  beforeEach(async () => {
    ({ fheCounterContract, fheCounterContractAddress } = await deployFixture());
  });

  it("部署后加密计数值应未初始化", async function () {
    const encryptedCount = await fheCounterContract.getCount();
    // 部署后初始计数值应为 bytes32(0)，
    // (表示加密计数值未初始化)
    expect(encryptedCount).to.eq(ethers.ZeroHash);
  });

  it("计数器加 1", async function () {
    const encryptedCountBeforeInc = await fheCounterContract.getCount();
    expect(encryptedCountBeforeInc).to.eq(ethers.ZeroHash);
    const clearCountBeforeInc = 0;

    // 将常量 1 加密为 euint32
    const clearOne = 1;
    const encryptedOne = await fhevm
      .createEncryptedInput(fheCounterContractAddress, signers.alice.address)
      .add32(clearOne)
      .encrypt();

    const tx = await fheCounterContract
      .connect(signers.alice)
      .increment(encryptedOne.handles[0], encryptedOne.inputProof);
    await tx.wait();

    const encryptedCountAfterInc = await fheCounterContract.getCount();
    const clearCountAfterInc = await fhevm.userDecryptEuint(
      FhevmType.euint32,
      encryptedCountAfterInc,
      fheCounterContractAddress,
      signers.alice,
    );

    expect(clearCountAfterInc).to.eq(clearCountBeforeInc + clearOne);
  });

  it("计数器减 1", async function () {
    // 将常量 1 加密为 euint32
    const clearOne = 1;
    const encryptedOne = await fhevm
      .createEncryptedInput(fheCounterContractAddress, signers.alice.address)
      .add32(clearOne)
      .encrypt();

    // 首先加 1，计数值变为 1
    let tx = await fheCounterContract
      .connect(signers.alice)
      .increment(encryptedOne.handles[0], encryptedOne.inputProof);
    await tx.wait();

    // 然后减 1，计数值回到 0
    tx = await fheCounterContract.connect(signers.alice).decrement(encryptedOne.handles[0], encryptedOne.inputProof);
    await tx.wait();

    const encryptedCountAfterDec = await fheCounterContract.getCount();
    const clearCountAfterInc = await fhevm.userDecryptEuint(
      FhevmType.euint32,
      encryptedCountAfterDec,
      fheCounterContractAddress,
      signers.alice,
    );

    expect(clearCountAfterInc).to.eq(0);
  });
});
```

{% endtab %}

{% endtabs %}
