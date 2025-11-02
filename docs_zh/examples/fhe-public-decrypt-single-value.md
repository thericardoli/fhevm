本示例演示了使用单个值进行 FHE 公共解密的机制。

公共解密是一种一旦解密，加密值就对所有人可见的机制。与用户解密（值对授权用户保持私有）不同，公共解密使数据对所有参与者永久可见。公共解密调用通过智能合约在链上进行，使解密后的值成为区块链公共状态的一部分。

{% hint style="info" %}
为正确运行此示例，请确保将文件放置在以下目录中：

- `.sol` 文件 → `<your-project-root-dir>/contracts/`
- `.ts` 文件 → `<your-project-root-dir>/test/`

这能确保 Hardhat 能够按预期编译和测试您的合约。
{% endhint %}

{% tabs %}

{% tab title="PublicDecryptSingleValue.sol" %}

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, euint32 } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

contract PublicDecryptSingleValue is SepoliaConfig {
  euint32 private _encryptedUint32; // = 0 (未初始化)
  uint32 private _clearUint32; // = 0 (未初始化)

  // solhint-disable-next-line no-empty-blocks
  constructor() {}

  function initializeUint32(uint32 value) external {
    // 计算一个简单的 FHE 公式 _trivialEuint32 = value + 1
    _encryptedUint32 = FHE.add(FHE.asEuint32(value), FHE.asEuint32(1));

    // 授予 FHE 权限给：
    // ✅ 合约本身 (`address(this)`)：允许它向 FHEVM 后端请求异步公共解密
    //
    // 注意：如果您忘记调用 `FHE.allowThis(_trivialEuint32)`，
    //       合约本身 (`address(this)`) 对 `_trivialEuint32` 的任何异步公共解密请求
    //       都将失败！
    FHE.allowThis(_encryptedUint32);
  }

  function initializeUint32Wrong(uint32 value) external {
    // 计算一个简单的 FHE 公式 _trivialEuint32 = value + 1
    _encryptedUint32 = FHE.add(FHE.asEuint32(value), FHE.asEuint32(1));
  }

  function requestDecryptSingleUint32() external {
    bytes32[] memory cypherTexts = new bytes32[](1);
    cypherTexts[0] = FHE.toBytes32(_encryptedUint32);

    // 两种可能的结果：
    // ✅ 如果调用了 `initializeUint32`，公共解密请求将成功。
    // ❌ 如果调用了 `initializeUint32Wrong`，公共解密请求将失败 💥
    //
    // 解释：
    // 仅当合约本身 (`address(this)`) 被授予
    // 必要的 FHE 权限时，请求才会成功。缺少 `FHE.allowThis(...)` 将导致失败。
    FHE.requestDecryption(
      // 我们想要公共解密的加密值列表
      cypherTexts,
      // FHEVM 后端将使用明文值作为参数回调的函数选择器
      this.callbackDecryptSingleUint32.selector
    );
  }

  function callbackDecryptSingleUint32(uint256 requestID, bytes memory cleartexts, bytes memory decryptionProof) external {
    // `cleartexts` 参数是与句柄关联的解密值的 ABI 编码
    //（使用 `abi.encode`）。
    //
    // ===============================
    //    ☠️🔒 安全警告！🔒☠️
    // ===============================
    //
    // 必须在这里调用 `FHE.checkSignatures(...)`！
    //            ------------------------
    //
    // 此回调只能由授权的 FHEVM 后端调用。
    // 为强制执行此操作，合约作者必须通过使用 `FHE.checkSignatures` 帮助程序
    // 验证调用者的真实性。这能确保提供的签名
    // 与预期的 FHEVM 后端匹配，并防止未经授权或恶意的调用。
    //
    // 不执行此验证将允许任何人使用伪造的值调用此函数，
    // 从而可能危及合约的完整性。
    //
    // 签名验证的责任完全在于合约作者。
    //
    // 签名包含在 `decryptionProof` 参数中。
    //
    FHE.checkSignatures(requestID, cleartexts, decryptionProof);

    (uint32 decryptedInput) = abi.decode(cleartexts, (uint32));
    _clearUint32 = decryptedInput;
  }

  function clearUint32() public view returns (uint32) {
    return _clearUint32;
  }
}
```

{% endtab %}

{% tab title="PublicDecryptSingleValue.ts" %}

```ts
import { PublicDecryptSingleValue, PublicDecryptSingleValue__factory } from "../../../types";
import type { Signers } from "../../types";
import { HardhatFhevmRuntimeEnvironment } from "@fhevm/hardhat-plugin";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";
import * as hre from "hardhat";

async function deployFixture() {
  // 默认情况下，合约使用第一个签名者/账户进行部署
  const factory = (await ethers.getContractFactory(
    "PublicDecryptSingleValue",
  )) as PublicDecryptSingleValue__factory;
  const publicDecryptSingleValue = (await factory.deploy()) as PublicDecryptSingleValue;
  const publicDecryptSingleValue_address = await publicDecryptSingleValue.getAddress();

  return { publicDecryptSingleValue, publicDecryptSingleValue_address };
}

/**
 * 这个简单的示例演示了 FHE 公共解密机制，
 * 并强调了开发人员可能遇到的一个常见陷阱。
 */
describe("PublicDecryptSingleValue", function () {
  let contract: PublicDecryptSingleValue;
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
    contract = deployment.publicDecryptSingleValue;
  });

  // ✅ 测试应成功
  it("公共解密应成功", async function () {
    let tx = await contract.connect(signers.alice).initializeUint32(123456);
    await tx.wait();

    tx = await contract.requestDecryptSingleUint32();
    await tx.wait();

    // 我们使用 FHEVM Hardhat 插件来模拟异步的链上
    // 公共解密
    const fhevm: HardhatFhevmRuntimeEnvironment = hre.fhevm;

    // 使用内置的 `awaitDecryptionOracle` 帮助程序等待 FHEVM 公共解密预言机
    // 完成所有待处理的 Solidity 公共解密请求。
    await fhevm.awaitDecryptionOracle();

    // 此时，Solidity 回调应该已被 FHEVM 后端调用。
    // 我们现在可以检索解密的（明文）值。
    const clearUint32 = await contract.clearUint32();

    expect(clearUint32).to.equal(123456 + 1);
  });

  // ❌ 测试应失败
  it("解密应失败", async function () {
    const tx = await contract.connect(signers.alice).initializeUint32Wrong(123456);
    await tx.wait();

    const fhevm: HardhatFhevmRuntimeEnvironment = hre.fhevm;

    const senderNotAllowedError = fhevm.revertedWithCustomErrorArgs("ACL", "SenderNotAllowed");

    await expect(contract.connect(signers.alice).requestDecryptSingleUint32()).to.be.revertedWithCustomError(
      ...senderNotAllowedError,
    );
  });
});
```

{% endtab %}

{% endtabs %}
