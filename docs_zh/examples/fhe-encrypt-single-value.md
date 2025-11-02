本示例演示了 FHE 加密机制，并强调了开发人员可能遇到的一个常见陷阱。

{% hint style="info" %}
为正确运行此示例，请确保将文件放置在以下目录中：

- `.sol` 文件 → `<your-project-root-dir>/contracts/`
- `.ts` 文件 → `<your-project-root-dir>/test/`

这能确保 Hardhat 能够按预期编译和测试您的合约。
{% endhint %}

{% tabs %}

{% tab title="EncryptSingleValue.sol" %}

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, externalEuint32, euint32 } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

/**
 * 这个简单的示例演示了 FHE 加密机制。
 */
contract EncryptSingleValue is SepoliaConfig {
  euint32 private _encryptedEuint32;

  // solhint-disable-next-line no-empty-blocks
  constructor() {}

  function initialize(externalEuint32 inputEuint32, bytes calldata inputProof) external {
    _encryptedEuint32 = FHE.fromExternal(inputEuint32, inputProof);

    // 授予 FHE 权限给合约本身 (`address(this)`) 和调用者 (`msg.sender`)，
    // 以允许调用者 (`msg.sender`) 将来解密。
    FHE.allowThis(_encryptedEuint32);
    FHE.allow(_encryptedEuint32, msg.sender);
  }

  function encryptedUint32() public view returns (euint32) {
    return _encryptedEuint32;
  }
}
```

{% endtab %}

{% tab title="EncryptSingleValue.ts" %}

```ts
import { EncryptSingleValue, EncryptSingleValue__factory } from "../../../types";
import type { Signers } from "../../types";
import { FhevmType, HardhatFhevmRuntimeEnvironment } from "@fhevm/hardhat-plugin";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";
import * as hre from "hardhat";

async function deployFixture() {
  // 默认情况下，合约使用第一个签名者/账户进行部署
  const factory = (await ethers.getContractFactory("EncryptSingleValue")) as EncryptSingleValue__factory;
  const encryptSingleValue = (await factory.deploy()) as EncryptSingleValue;
  const encryptSingleValue_address = await encryptSingleValue.getAddress();

  return { encryptSingleValue, encryptSingleValue_address };
}

/**
 * 这个简单的示例演示了 FHE 加密机制，
 * 并强调了开发人员可能遇到的一个常见陷阱。
 */
describe("EncryptSingleValue", function () {
  let contract: EncryptSingleValue;
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
    contractAddress = deployment.encryptSingleValue_address;
    contract = deployment.encryptSingleValue;
  });

  // ✅ 测试应成功
  it("加密应成功", async function () {
    // 使用 FHEVM Hardhat 插件运行时环境
    // 来执行 FHEVM 输入加密。
    const fhevm: HardhatFhevmRuntimeEnvironment = hre.fhevm;

    // 🔐 加密过程：
    // 值在本地加密并绑定到特定的合约/用户对。
    // 这授予绑定的合约 FHE 权限以接收和处理加密值，
    // 但仅当它由绑定的用户发送时。
    const input = fhevm.createEncryptedInput(contractAddress, signers.alice.address);

    // 将一个 uint32 值添加到要在本地加密的值列表中。
    input.add32(123456);

    // 执行本地加密。此操作产生两个组件：
    // 1. `handles`：一个 FHEVM 句柄数组。在这种情况下，是与
    //    本地加密的 uint32 值 `123456` 关联的单个句柄。
    // 2. `inputProof`：一个零知识证明，证明 `handles` 在密码学上
    //    绑定到 `[contractAddress, signers.alice.address]` 对。
    const enc = await input.encrypt();

    // 一个 32 字节的 FHEVM 句柄，代表一个未来的 Solidity `euint32` 值。
    const inputEuint32 = enc.handles[0];
    const inputProof = enc.inputProof;

    // 现在 `signers.alice.address` 可以将加密值及其关联的零知识证明
    // 发送到部署在 `contractAddress` 的智能合约。
    const tx = await contract.connect(signers.alice).initialize(inputEuint32, inputProof);
    await tx.wait();

    // 让我们尝试解密它以检查一切是否正常！
    const encryptedUint32 = await contract.encryptedUint32();

    const clearUint32 = await fhevm.userDecryptEuint(
      FhevmType.euint32, // 指定加密类型
      encryptedUint32,
      contractAddress, // 合约地址
      signers.alice, // 用户钱包
    );

    expect(clearUint32).to.equal(123456);
  });

  // ❌ 此测试说明了一个非常常见的陷阱
  it("加密应失败", async function () {
    const fhevm: HardhatFhevmRuntimeEnvironment = hre.fhevm;

    const enc = await fhevm.createEncryptedInput(contractAddress, signers.alice.address).add32(123456).encrypt();

    const inputEuint32 = enc.handles[0];
    const inputProof = enc.inputProof;

    try {
      // 这是一个非常常见的错误！
      // `contract.initialize` 将使用用户 `signers.owner`
      // 而不是 `signers.alice` 来签署以太坊交易。
      //
      // 在 Solidity 合约中，会检查以下内容：
      // - 合约是否允许操作 `inputEuint32`？答案是：✅ 是的！
      // - 发送者是否允许操作 `inputEuint32`？答案是：❌ 不！只有 `signers.alice` 可以！
      const tx = await contract.initialize(inputEuint32, inputProof);
      await tx.wait();
    } catch {
      //console.log(e);
    }
  });
});
```

{% endtab %}

{% endtabs %}
