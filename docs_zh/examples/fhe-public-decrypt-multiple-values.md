本示例演示了使用多个值进行 FHE 公共解密的机制。

公共解密是一种一旦解密，加密值就对所有人可见的机制。与用户解密（值对授权用户保持私有）不同，公共解密使数据对所有参与者永久可见。公共解密调用通过智能合约在链上进行，使解密后的值成为区块链公共状态的一部分。

{% hint style="info" %}
为正确运行此示例，请确保将文件放置在以下目录中：

- `.sol` 文件 → `<your-project-root-dir>/contracts/`
- `.ts` 文件 → `<your-project-root-dir>/test/`

这能确保 Hardhat 能够按预期编译和测试您的合约。
{% endhint %}

{% tabs %}

{% tab title="PublicDecryptMultipleValues.sol" %}

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, ebool, euint32, euint64 } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

contract PublicDecryptMultipleValues is SepoliaConfig {
  ebool private _encryptedBool; // = 0 (未初始化)
  euint32 private _encryptedUint32; // = 0 (未初始化)
  euint64 private _encryptedUint64; // = 0 (未初始化)

  bool private _clearBool; // = 0 (未初始化)
  uint32 private _clearUint32; // = 0 (未初始化)
  uint64 private _clearUint64; // = 0 (未初始化)

  // solhint-disable-next-line no-empty-blocks
  constructor() {}

  function initialize(bool a, uint32 b, uint64 c) external {
    // 计算 3 个简单的 FHE 公式

    // _encryptedBool = a ^ false
    _encryptedBool = FHE.xor(FHE.asEbool(a), FHE.asEbool(false));

    // _encryptedUint32 = b + 1
    _encryptedUint32 = FHE.add(FHE.asEuint32(b), FHE.asEuint32(1));

    // _encryptedUint64 = c + 1
    _encryptedUint64 = FHE.add(FHE.asEuint64(c), FHE.asEuint64(1));

    // 有关 FHE 权限和异步公共解密请求的更详细说明，
    // 请参见 `DecryptSingleValueInSolidity.sol`。
    FHE.allowThis(_encryptedBool);
    FHE.allowThis(_encryptedUint32);
    FHE.allowThis(_encryptedUint64);
  }

  function requestDecryptMultipleValues() external {
    // 要公共解密多个值，我们必须构建一个包含
    // 我们想要公共解密的加密值的数组。
    //
    // ⚠️ 警告：数组中值的顺序至关重要！
    // FHEVM 后端将按它们在此数组中出现的完全相同的顺序
    // 将公共解密的值传递给回调函数。
    // 因此，顺序必须与回调中的参数声明匹配。
    bytes32[] memory cypherTexts = new bytes32[](3);
    cypherTexts[0] = FHE.toBytes32(_encryptedBool);
    cypherTexts[1] = FHE.toBytes32(_encryptedUint32);
    cypherTexts[2] = FHE.toBytes32(_encryptedUint64);

    FHE.requestDecryption(
      // 我们想要公共解密的加密值列表
      cypherTexts,
      // Solidity 回调函数的选择器，FHEVM 后端将使用
      // 解密（明文）值作为参数调用该函数
      this.callbackDecryptMultipleValues.selector
    );
  }

  // ⚠️ 警告：`cleartexts` 参数是与句柄关联的解密值的 ABI 编码
  //（使用 `abi.encode`）。
  //
  // 这些值的类型必须完全匹配！类型不匹配——例如使用 `uint32 decryptedUint64`
  // 而不是正确的 `uint64 decryptedUint64`——可能会导致难以检测的细微错误，
  // 特别是对于 FHEVM 技术栈的新手开发人员。
  // 始终确保参数类型与预期的解密值类型一致。
  //
  // !请仔细检查！
  function callbackDecryptMultipleValues(
    uint256 requestID,
    bytes memory cleartexts,
    bytes memory decryptionProof
  ) external {
    // ⚠️ 不要忘记签名检查！（有关详细说明，请参见 `DecryptSingleValueInSolidity.sol`）
    // 签名包含在 `decryptionProof` 参数中。
    FHE.checkSignatures(requestID, cleartexts, decryptionProof);

    (bool decryptedBool, uint32 decryptedUint32, uint64 decryptedUint64) = abi.decode(cleartexts, (bool, uint32, uint64));
    _clearBool = decryptedBool;
    _clearUint32 = decryptedUint32;
    _clearUint64 = decryptedUint64;
  }

  function clearBool() public view returns (bool) {
    return _clearBool;
  }

  function clearUint32() public view returns (uint32) {
    return _clearUint32;
  }

  function clearUint64() public view returns (uint64) {
    return _clearUint64;
  }
}
```

{% endtab %}

{% tab title="PublicDecryptMultipleValues.ts" %}

```ts
import { PublicDecryptMultipleValues, PublicDecryptMultipleValues__factory } from "../../../types";
import type { Signers } from "../../types";
import { HardhatFhevmRuntimeEnvironment } from "@fhevm/hardhat-plugin";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";
import * as hre from "hardhat";

async function deployFixture() {
  // 默认情况下，合约使用第一个签名者/账户进行部署
  const factory = (await ethers.getContractFactory(
    "PublicDecryptMultipleValues",
  )) as PublicDecryptMultipleValues__factory;
  const publicDecryptMultipleValues = (await factory.deploy()) as PublicDecryptMultipleValues;
  const publicDecryptMultipleValues_address = await publicDecryptMultipleValues.getAddress();

  return { publicDecryptMultipleValues, publicDecryptMultipleValues_address };
}

/**
 * 这个简单的示例演示了 FHE 公共解密机制，
 * 并强调了开发人员可能遇到的一个常见陷阱。
 */
describe("PublicDecryptMultipleValues", function () {
  let contract: PublicDecryptMultipleValues;
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
    contract = deployment.publicDecryptMultipleValues;
  });

  // ✅ 测试应成功
  it("公共解密应成功", async function () {
    // 为简单起见，我们在链上创建 3 个简单加密的值。
    let tx = await contract.connect(signers.alice).initialize(true, 123456, 78901234567);
    await tx.wait();

    tx = await contract.requestDecryptMultipleValues();
    await tx.wait();

    // 我们使用 FHEVM Hardhat 插件来模拟异步的链上
    // 公共解密
    const fhevm: HardhatFhevmRuntimeEnvironment = hre.fhevm;

    // 使用内置的 `awaitDecryptionOracle` 帮助程序等待 FHEVM 公共解密预言机
    // 完成所有待处理的 Solidity 公共解密请求。
    await fhevm.awaitDecryptionOracle();

    // 此时，Solidity 回调应该已被 FHEVM 后端调用。
    // 我们现在可以检索 3 个公共解密的（明文）值。
    const clearBool = await contract.clearBool();
    const clearUint32 = await contract.clearUint32();
    const clearUint64 = await contract.clearUint64();

    expect(clearBool).to.equal(true);
    expect(clearUint32).to.equal(123456 + 1);
    expect(clearUint64).to.equal(78901234567 + 1);
  });
});
```

{% endtab %}

{% endtabs %}
