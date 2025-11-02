本示例演示了如何使用 FHEVM 编写一个带条件的简单合约，并与一个简单计数器进行比较。

{% hint style="info" %}
为正确运行此示例，请确保将文件放置在以下目录中：

- `.sol` 文件 → `<your-project-root-dir>/contracts/`
- `.ts` 文件 → `<your-project-root-dir>/test/`

这能确保 Hardhat 能够按预期编译和测试您的合约。
{% endhint %}

{% tabs %}

{% tab title="FHEIfThenElse.sol" %}

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, ebool, euint8, externalEuint8 } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

contract FHEIfThenElse is SepoliaConfig {
  euint8 private _a;
  euint8 private _b;
  euint8 private _max;

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

  function computeMax() external {
    // a >= b
    // solhint-disable-next-line var-name-mixedcase
    ebool _a_ge_b = FHE.ge(_a, _b);

    // a >= b ? a : b
    _max = FHE.select(_a_ge_b, _a, _b);

    // 有关此情况下 FHE 权限的更多信息，
    // 请阅读 `FHEAdd.sol` 中的 `computeAPlusB()` 注释。
    FHE.allowThis(_max);
    FHE.allow(_max, msg.sender);
  }

  function result() public view returns (euint8) {
    return _max;
  }
}
```

{% endtab %}

{% tab title="FHEIfThenElse.ts" %}

```ts
import { FHEIfThenElse, FHEIfThenElse__factory } from "../../../types";
import type { Signers } from "../../types";
import { FhevmType, HardhatFhevmRuntimeEnvironment } from "@fhevm/hardhat-plugin";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";
import * as hre from "hardhat";

async function deployFixture() {
  // 默认情况下，合约使用第一个签名者/账户进行部署
  const factory = (await ethers.getContractFactory("FHEIfThenElse")) as FHEIfThenElse__factory;
  const fheIfThenElse = (await factory.deploy()) as FHEIfThenElse;
  const fheIfThenElse_address = await fheIfThenElse.getAddress();

  return { fheIfThenElse, fheIfThenElse_address };
}

/**
 * 这个简单的示例演示了 FHE 加密机制，
 * 并强调了开发人员可能遇到的一个常见陷阱。
 */
describe("FHEIfThenElse", function () {
  let contract: FHEIfThenElse;
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
    contractAddress = deployment.fheIfThenElse_address;
    contract = deployment.fheIfThenElse;
  });

  it("a >= b ? a : b 应成功", async function () {
    const fhevm: HardhatFhevmRuntimeEnvironment = hre.fhevm;

    let tx;

    // 让我们计算 `a >= b ? a : b`
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
    tx = await contract.connect(bob).computeMax();
    await tx.wait();

    const encryptedMax = await contract.result();

    const clearMax = await fhevm.userDecryptEuint(
      FhevmType.euint8, // 指定加密类型
      encryptedMax,
      contractAddress, // 合约地址
      bob, // 用户钱包
    );

    expect(clearMax).to.equal(a >= b ? a : b);
  });
});
```

{% endtab %}

{% endtabs %}
