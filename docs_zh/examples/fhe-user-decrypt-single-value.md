本示例演示了使用单个值进行 FHE 用户解密的机制。

用户解密是一种允许特定用户解密加密值，同时对其他人保持隐藏的机制。与公共解密（解密后的值对所有人可见）不同，用户解密通过仅允许拥有适当权限的授权用户查看数据来维护隐私。虽然权限通过智能合约在链上授予，但实际的**解密调用是在前端应用程序中离线进行**的。

{% hint style="info" %}
为正确运行此示例，请确保将文件放置在以下目录中：

- `.sol` 文件 → `<your-project-root-dir>/contracts/`
- `.ts` 文件 → `<your-project-root-dir>/test/`

这能确保 Hardhat 能够按预期编译和测试您的合约。
{% endhint %}

{% tabs %}

{% tab title="UserDecryptSingleValue.sol" %}

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, euint32 } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

/**
 * 这个简单的示例演示了 FHE 解密机制，
 * 并强调了开发人员可能遇到的常见陷阱。
 */
contract UserDecryptSingleValue is SepoliaConfig {
  euint32 private _trivialEuint32;

  // solhint-disable-next-line no-empty-blocks
  constructor() {}

  function initializeUint32(uint32 value) external {
    // 计算一个简单的 FHE 公式 _trivialEuint32 = value + 1
    _trivialEuint32 = FHE.add(FHE.asEuint32(value), FHE.asEuint32(1));

    // 授予 FHE 权限给：
    // ✅ 合约调用者 (`msg.sender`)：允许他们解密 `_trivialEuint32`。
    // ✅ 合约本身 (`address(this)`)：允许它操作 `_trivialEuint32` 并
    //    也使调用者能够执行用户解密。
    //
    // 注意：如果您忘记调用 `FHE.allowThis(_trivialEuint32)`，用户将无法
    //       用户解密该值！合约和调用者都必须拥有 FHE 权限
    //       才能成功进行用户解密。
    FHE.allowThis(_trivialEuint32);
    FHE.allow(_trivialEuint32, msg.sender);
  }

  function initializeUint32Wrong(uint32 value) external {
    // 计算一个简单的 FHE 公式 _trivialEuint32 = value + 1
    _trivialEuint32 = FHE.add(FHE.asEuint32(value), FHE.asEuint32(1));

    // ❌ 常见的 FHE 权限错误：
    // ================================================================
    // 我们授予 FHE 权限给合约调用者 (`msg.sender`)，
    // 期望他们以后能够用户解密加密值。
    //
    // 然而，这将会失败！💥
    // 合约本身 (`address(this)`) 也需要 FHE 权限才能允许用户解密。
    // 如果不使用 `FHE.allowThis(...)` 授予合约访问权限，
    // 用户的用户解密尝试将不会成功。
    FHE.allow(_trivialEuint32, msg.sender);
  }

  function encryptedUint32() public view returns (euint32) {
    return _trivialEuint32;
  }
}
```

{% endtab %}

{% tab title="UserDecryptSingleValue.ts" %}

```ts
import { UserDecryptSingleValue, UserDecryptSingleValue__factory } from "../../../types";
import type { Signers } from "../../types";
import { FhevmType, HardhatFhevmRuntimeEnvironment } from "@fhevm/hardhat-plugin";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";
import * as hre from "hardhat";

async function deployFixture() {
  // 默认情况下，合约使用第一个签名者/账户进行部署
  const factory = (await ethers.getContractFactory("UserDecryptSingleValue")) as UserDecryptSingleValue__factory;
  const userUserDecryptSingleValue = (await factory.deploy()) as UserDecryptSingleValue;
  const userUserDecryptSingleValue_address = await userUserDecryptSingleValue.getAddress();

  return { userUserDecryptSingleValue, userUserDecryptSingleValue_address };
}

/**
 * 这个简单的示例演示了 FHE 用户解密机制，
 * 并强调了开发人员可能遇到的一个常见陷阱。
 */
describe("UserDecryptSingleValue", function () {
  let contract: UserDecryptSingleValue;
  let contractAddress: string;
  let signers: Signers;

  before(async function () {
    // 检查测试是否在 FHEVM 模拟环境中运行
    if (!hre.fhevm.isMock) {
      throw new Error(`此 hardhat 测试套件无法在 Sepolia 测试网上运行`);
    }

    const ethSigners: HardhatEthersSigner[] = await ethers.getSigners();
    signers = { owner: ethSigners[0], alice: ethSigners[1] };
  });

  beforeEach(async function () {
    // 每次运行新测试时都部署一个新合约
    const deployment = await deployFixture();
    contractAddress = deployment.userUserDecryptSingleValue_address;
    contract = deployment.userUserDecryptSingleValue;
  });

  // ✅ 测试应成功
  it("用户解密应成功", async function () {
    const tx = await contract.connect(signers.alice).initializeUint32(123456);
    await tx.wait();

    const encryptedUint32 = await contract.encryptedUint32();

    // FHEVM Hardhat 插件提供了一组方便的帮助函数，
    // 可以轻松地在您的 Hardhat 环境中执行 FHEVM 操作。
    const fhevm: HardhatFhevmRuntimeEnvironment = hre.fhevm;

    const clearUint32 = await fhevm.userDecryptEuint(
      FhevmType.euint32, // 指定加密类型
      encryptedUint32,
      contractAddress, // 合约地址
      signers.alice, // 用户钱包
    );

    expect(clearUint32).to.equal(123456 + 1);
  });

  // ❌ 测试应失败
  it("用户解密应失败", async function () {
    const tx = await contract.connect(signers.alice).initializeUint32Wrong(123456);
    await tx.wait();

    const encryptedUint32 = await contract.encryptedUint32();

    await expect(
      hre.fhevm.userDecryptEuint(FhevmType.euint32, encryptedUint32, contractAddress, signers.alice),
    ).to.be.rejectedWith(new RegExp("^dapp contract (.+) is not authorized to user decrypt handle (.+)."));
  });
});
```

{% endtab %}

{% endtabs %}
