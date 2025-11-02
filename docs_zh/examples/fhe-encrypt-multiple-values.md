本示例演示了使用多个值进行 FHE 加密的机制。

{% hint style="info" %}
为正确运行此示例，请确保将文件放置在以下目录中：

- `.sol` 文件 → `<your-project-root-dir>/contracts/`
- `.ts` 文件 → `<your-project-root-dir>/test/`

这能确保 Hardhat 能够按预期编译和测试您的合约。
{% endhint %}

{% tabs %}

{% tab title="EncryptMultipleValues.sol" %}

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import {
  FHE,
  externalEbool,
  externalEuint32,
  externalEaddress,
  ebool,
  euint32,
  eaddress
} from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

/**
 * 这个简单的示例演示了 FHE 加密机制。
 */
contract EncryptMultipleValues is SepoliaConfig {
  ebool private _encryptedEbool;
  euint32 private _encryptedEuint32;
  eaddress private _encryptedEaddress;

  // solhint-disable-next-line no-empty-blocks
  constructor() {}

  function initialize(
    externalEbool inputEbool,
    externalEuint32 inputEuint32,
    externalEaddress inputEaddress,
    bytes calldata inputProof
  ) external {
    _encryptedEbool = FHE.fromExternal(inputEbool, inputProof);
    _encryptedEuint32 = FHE.fromExternal(inputEuint32, inputProof);
    _encryptedEaddress = FHE.fromExternal(inputEaddress, inputProof);

    // 对于这 3 个值中的每一个：
    // 授予 FHE 权限给合约本身 (`address(this)`) 和调用者 (`msg.sender`)，
    // 以允许调用者 (`msg.sender`) 将来解密。

    FHE.allowThis(_encryptedEbool);
    FHE.allow(_encryptedEbool, msg.sender);

    FHE.allowThis(_encryptedEuint32);
    FHE.allow(_encryptedEuint32, msg.sender);

    FHE.allowThis(_encryptedEaddress);
    FHE.allow(_encryptedEaddress, msg.sender);
  }

  function encryptedBool() public view returns (ebool) {
    return _encryptedEbool;
  }

  function encryptedUint32() public view returns (euint32) {
    return _encryptedEuint32;
  }

  function encryptedAddress() public view returns (eaddress) {
    return _encryptedEaddress;
  }
}
```

{% endtab %}

{% tab title="EncryptMultipleValues.ts" %}

```ts
//TODO;
import { EncryptMultipleValues, EncryptMultipleValues__factory } from "../../../types";
import type { Signers } from "../../types";
import { FhevmType, HardhatFhevmRuntimeEnvironment } from "@fhevm/hardhat-plugin";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";
import * as hre from "hardhat";

async function deployFixture() {
  // 默认情况下，合约使用第一个签名者/账户进行部署
  const factory = (await ethers.getContractFactory("EncryptMultipleValues")) as EncryptMultipleValues__factory;
  const encryptMultipleValues = (await factory.deploy()) as EncryptMultipleValues;
  const encryptMultipleValues_address = await encryptMultipleValues.getAddress();

  return { encryptMultipleValues, encryptMultipleValues_address };
}

/**
 * 这个简单的示例演示了 FHE 加密机制，
 * 并强调了开发人员可能遇到的一个常见陷阱。
 */
describe("EncryptMultipleValues", function () {
  let contract: EncryptMultipleValues;
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
    contractAddress = deployment.encryptMultipleValues_address;
    contract = deployment.encryptMultipleValues;
  });

  // ✅ 测试应成功
  it("加密应成功", async function () {
    // 使用 FHEVM Hardhat 插件运行时环境
    // 来执行 FHEVM 输入加密。
    const fhevm: HardhatFhevmRuntimeEnvironment = hre.fhevm;

    const input = fhevm.createEncryptedInput(contractAddress, signers.alice.address);

    input.addBool(true);
    input.add32(123456);
    input.addAddress(signers.owner.address);

    const enc = await input.encrypt();

    const inputEbool = enc.handles[0];
    const inputEuint32 = enc.handles[1];
    const inputEaddress = enc.handles[2];
    const inputProof = enc.inputProof;

    // 不要忘记调用 `connect(signers.alice)` 以确保
    // Solidity 的 `msg.sender` 是 `signers.alice.address`。
    const tx = await contract.connect(signers.alice).initialize(inputEbool, inputEuint32, inputEaddress, inputProof);
    await tx.wait();

    const encryptedBool = await contract.encryptedBool();
    const encryptedUint32 = await contract.encryptedUint32();
    const encryptedAddress = await contract.encryptedAddress();

    const clearBool = await fhevm.userDecryptEbool(
      encryptedBool,
      contractAddress, // 合约地址
      signers.alice, // 用户钱包
    );

    const clearUint32 = await fhevm.userDecryptEuint(
      FhevmType.euint32, // 指定加密类型
      encryptedUint32,
      contractAddress, // 合约地址
      signers.alice, // 用户钱包
    );

    const clearAddress = await fhevm.userDecryptEaddress(
      encryptedAddress,
      contractAddress, // 合约地址
      signers.alice, // 用户钱包
    );

    expect(clearBool).to.equal(true);
    expect(clearUint32).to.equal(123456);
    expect(clearAddress).to.equal(signers.owner.address);
  });
});
```

{% endtab %}

{% endtabs %}
