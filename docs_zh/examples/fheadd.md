本示例演示了如何使用 FHEVM 编写一个简单的“a + b”合约。

{% hint style="info" %}
为正确运行此示例，请确保将文件放置在以下目录中：

- `.sol` 文件 → `<your-project-root-dir>/contracts/`
- `.ts` 文件 → `<your-project-root-dir>/test/`

这能确保 Hardhat 能够按预期编译和测试您的合约。
{% endhint %}

{% tabs %}

{% tab title="FHEAdd.sol" %}

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, euint8, externalEuint8 } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

contract FHEAdd is SepoliaConfig {
  euint8 private _a;
  euint8 private _b;
  // solhint-disable-next-line var-name-mixedcase
  euint8 private _a_plus_b;

  // solhint-disable-next-line no-empty-blocks
  constructor() {}

  function setA(externalEuint8 inputA, bytes calldata inputProof) external {
    _a = FHE.fromExternal(inputA, inputProof);
    FHE.allowThis(_a);
  }

  function setB(externalEuint8 inputB, bytes calldata inputProof) external {
    _b = FHE.fromExternal(inputB, inputProof);
    FHE.allowThis(_b);
  }

  function computeAPlusB() external {
    // `a + b` 的和由合约本身 (`address(this)`) 计算。
    // 由于合约对 `a` 和 `b` 都拥有 FHE 权限，
    // 因此它有权对这些值执行 `FHE.add` 操作。
    // 合约调用者 (`msg.sender`) 是否拥有 FHE 权限无关紧要。
    _a_plus_b = FHE.add(_a, _b);

    // 此时，合约本身 (`address(this)`) 已被授予对 `_a_plus_b` 的临时 FHE 权限。
    // 此 FHE 权限将在函数退出时被撤销。
    //
    // 现在，为确保 `_a_plus_b` 可以由合约调用者 (`msg.sender`) 解密，
    // 我们需要向合约本身 (`address(this)`) 和合约调用者 (`msg.sender`)
    // 授予永久 FHE 权限。
    FHE.allowThis(_a_plus_b);
    FHE.allow(_a_plus_b, msg.sender);
  }

  function result() public view returns (euint8) {
    return _a_plus_b;
  }
}
```

{% endtab %}

{% tab title="FHEAdd.ts" %}

```ts
import { FHEAdd, FHEAdd__factory } from "../../../types";
import type { Signers } from "../../types";
import { FhevmType, HardhatFhevmRuntimeEnvironment } from "@fhevm/hardhat-plugin";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";
import * as hre from "hardhat";

async function deployFixture() {
  // 默认情况下，合约使用第一个签名者/账户进行部署
  const factory = (await ethers.getContractFactory("FHEAdd")) as FHEAdd__factory;
  const fheAdd = (await factory.deploy()) as FHEAdd;
  const fheAdd_address = await fheAdd.getAddress();

  return { fheAdd, fheAdd_address };
}

/**
 * 这个简单的示例演示了 FHE 加密机制，
 * 并强调了开发人员可能遇到的一个常见陷阱。
 */
describe("FHEAdd", function () {
  let contract: FHEAdd;
  let contractAddress: string;
  let signers: Signers;
  let bob: HardhatEthersSigner;

  before(async function () {
    // 检查测试是否在 FHEVM 模拟环境中运行
    if (!hre.fhevm.isMock) {
      throw new Error(`此 hardhat 测试套件无法在 Sepolia 测试网上运行`);
    }

    const ethSigners: HardhatEthersSigner[] = await ethers.getSigners();
    signers = { owner: ethSigners[0], alice: ethSigners[1] };
    bob = ethSigners[2];
  });

  beforeEach(async function () {
    // 每次运行新测试时都部署一个新合约
    const deployment = await deployFixture();
    contractAddress = deployment.fheAdd_address;
    contract = deployment.fheAdd;
  });

  it("a + b 应成功", async function () {
    const fhevm: HardhatFhevmRuntimeEnvironment = hre.fhevm;

    let tx;

    // 让我们计算 80 + 123 = 203
    const a = 80;
    const b = 123;

    // Alice 加密并将 `a` 设置为 80
    const inputA = await fhevm.createEncryptedInput(contractAddress, signers.alice.address).add8(a).encrypt();
    tx = await contract.connect(signers.alice).setA(inputA.handles[0], inputA.inputProof);
    await tx.wait();

    // Alice 加密并将 `b` 设置为 203
    const inputB = await fhevm.createEncryptedInput(contractAddress, signers.alice.address).add8(b).encrypt();
    tx = await contract.connect(signers.alice).setB(inputB.handles[0], inputB.inputProof);
    await tx.wait();

    // 在这种情况下，为什么 Bob 拥有执行操作的 FHE 权限？
    // 有关详细答案，请参见 `FHEAdd.sol` 中的 `computeAPlusB()`
    tx = await contract.connect(bob).computeAPlusB();
    await tx.wait();

    const encryptedAplusB = await contract.result();

    const clearAplusB = await fhevm.userDecryptEuint(
      FhevmType.euint8, // 指定加密类型
      encryptedAplusB,
      contractAddress, // 合约地址
      bob, // 用户钱包
    );

    expect(clearAplusB).to.equal(a + b);
  });
});
```

{% endtab %}

{% endtabs %}
