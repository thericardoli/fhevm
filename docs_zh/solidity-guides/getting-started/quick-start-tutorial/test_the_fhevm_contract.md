# 测试 FHEVM 合约

在本教程中，您将学习如何将标准的 Hardhat 测试套件从 `Counter.ts` 迁移到其与 FHEVM 兼容的版本 `FHECounter.ts`——并逐步增强它以支持使用 Zama 的 FHEVM 库进行全同态加密。

## 设置 FHEVM 测试环境

{% stepper %}
{% step %}

## 创建测试脚本 `test/FHECounter.ts`

转到您项目的 `test` 目录

```sh
cd <your-project-root-directory>/test
```

在那里，创建一个名为 `FHECounter.ts` 的新文件，并将以下 Typescript 骨架代码复制并粘贴到其中。

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

  it("should be deployed", async function () {
    console.log(`FHECounter has been deployed at address ${fheCounterContractAddress}`);
    // Test the deployed address is valid
    expect(ethers.isAddress(fheCounterContractAddress)).to.eq(true);
  });

  //   it("count should be zero after deployment", async function () {
  //     const count = await counterContract.getCount();
  //     console.log(`Counter.getCount() === ${count}`);
  //     // Expect initial count to be 0 after deployment
  //     expect(count).to.eq(0);
  //   });

  //   it("increment the counter by 1", async function () {
  //     const countBeforeInc = await counterContract.getCount();
  //     const tx = await counterContract.connect(signers.alice).increment(1);
  //     await tx.wait();
  //     const countAfterInc = await counterContract.getCount();
  //     expect(countAfterInc).to.eq(countBeforeInc + 1n);
  //   });

  //   it("decrement the counter by 1", async function () {
  //     // First increment, count becomes 1
  //     let tx = await counterContract.connect(signers.alice).increment();
  //     await tx.wait();
  //     // Then decrement, count goes back to 0
  //     tx = await counterContract.connect(signers.alice).decrement(1);
  //     await tx.wait();
  //     const count = await counterContract.getCount();
  //     expect(count).to.eq(0);
  //   });
});
```

### 与 `Counter.ts` 有何不同？

- 此测试文件的结构与原始的 `Counter.ts` 相似，但它使用的是与 FHEVM 兼容的智能合约 `FHECounter`，而不是常规的 `Counter`。

– 为清楚起见，`Counter` 单元测试已作为注释包含在内，使您能够更好地了解在迁移到 FHEVM 期间如何调整每个部分。

- 虽然测试逻辑保持不变，但此版本现在已设置为支持通过 FHEVM 库进行加密计算——从而能够直接在链上操作机密值。

{% endstep %}

{% step %}

## 运行测试 `test/FHECounter.ts`

从您项目的根目录运行：

```sh
npx hardhat test
```

输出：

```sh
  FHECounter
FHECounter has been deployed at address 0x7553CB9124f974Ee475E5cE45482F90d5B6076BC
    ✔ should be deployed


  1 passing (1ms)
```

太棒了！您的 Hardhat FHEVM 测试环境已正确设置。

{% endstep %}
{% endstepper %}

## 测试函数

现在一切都已启动并运行，您可以开始测试您的合约函数了。

{% stepper %}
{% step %}

## 调用合约 `getCount()` 视图函数

将旧版 `Counter` 合约的注释掉的测试替换为：

```ts
//   it("count should be zero after deployment", async function () {
//     const count = await counterContract.getCount();
//     console.log(`Counter.getCount() === ${count}`);
//     // Expect initial count to be 0 after deployment
//     expect(count).to.eq(0);
//   });
```

与其 FHEVM 等效项：

```ts
it("encrypted count should be uninitialized after deployment", async function () {
  const encryptedCount = await fheCounterContract.getCount();
  // Expect initial count to be bytes32(0) after deployment,
  // (meaning the encrypted count value is uninitialized)
  expect(encryptedCount).to.eq(ethers.ZeroHash);
});
```

#### 有何不同？

– `encryptedCount` 不再是普通的 TypeScript 数字。它现在是一个表示 Solidity `bytes32` 值的十六进制字符串，称为 **FHEVM 句柄**。此句柄指向类型为 `euint32` 的加密 FHEVM 原语，该原语在内部表示加密的 Solidity `uint32` 原语类型。

- `encryptedCount` 等于 `0x0000000000000000000000000000000000000000000000000000000000000000`，这意味着 `encryptedCount` 未初始化，此时不引用任何加密值。

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
    ✔ encrypted count should be uninitialized after deployment


  2 passing (7ms)
```

{% endstep %}

{% step %}

## 设置 `increment()` 函数单元测试

我们将逐步将 `increment()` 单元测试迁移到 FHEVM。
首先，我们来处理第一次递增前的计数器值。
如上所述，计数器最初是一个等于零的 `bytes32` 值，这意味着 FHEVM `euint32` 变量未初始化。

我们将此解释为底层明文值为 0。

将旧版 `Counter` 合约的注释掉的测试替换为：

```ts
//   it("increment the counter by 1", async function () {
//     const countBeforeInc = await counterContract.getCount();
//     const tx = await counterContract.connect(signers.alice).increment(1);
//     await tx.wait();
//     const countAfterInc = await counterContract.getCount();
//     expect(countAfterInc).to.eq(countBeforeInc + 1n);
//   });
```

替换为以下内容：

```ts
it("increment the counter by 1", async function () {
  const encryptedCountBeforeInc = await fheCounterContract.getCount();
  expect(encryptedCountBeforeInc).to.eq(ethers.ZeroHash);
  const clearCountBeforeInc = 0;

  // const tx = await counterContract.connect(signers.alice).increment(1);
  // await tx.wait();
  // const countAfterInc = await counterContract.getCount();
  // expect(countAfterInc).to.eq(countBeforeInc + 1n);
});
```

{% endstep %}

{% step %}

## 加密 `increment()` 函数参数

`increment()` 函数接受一个参数：计数器应增加的值。在 `Counter.sol` 的初始版本中，此值为明文 `uint32`。

我们将改为传递一个加密值，使用 FHEVM `externalEuint32` 原语类型。这使我们能够安全地递增计数器，而无需在链上泄露输入值。

{% hint style="info" %}
我们使用的是 `externalEuint32` 而不是常规的 `euint32`。这告诉 FHEVM，加密的 `uint32` 是由外部提供的（例如，由用户），并且必须在合约中使用之前验证其完整性和真实性。
{% endhint %}

替换：

```ts
it("increment the counter by 1", async function () {
  const encryptedCountBeforeInc = await fheCounterContract.getCount();
  expect(encryptedCountBeforeInc).to.eq(ethers.ZeroHash);
  const clearCountBeforeInc = 0;

  // const tx = await counterContract.connect(signers.alice).increment(1);
  // await tx.wait();
  // const countAfterInc = await counterContract.getCount();
  // expect(countAfterInc).to.eq(countBeforeInc + 1n);
});
```

替换为以下内容：

```ts
it("increment the counter by 1", async function () {
  const encryptedCountBeforeInc = await fheCounterContract.getCount();
  expect(encryptedCountBeforeInc).to.eq(ethers.ZeroHash);
  const clearCountBeforeInc = 0;

  // Encrypt constant 1 as a euint32
  const clearOne = 1;
  const encryptedOne = await fhevm
    .createEncryptedInput(fheCounterContractAddress, signers.alice.address)
    .add32(clearOne)
    .encrypt();

  // const tx = await counterContract.connect(signers.alice).increment(1);
  // await tx.wait();
  // const countAfterInc = await counterContract.getCount();
  // expect(countAfterInc).to.eq(countBeforeInc + 1n);
});
```

{% hint style="info" %}
`fhevm.createEncryptedInput(fheCounterContractAddress, signers.alice.address)` 创建一个绑定到合约 (`fheCounterContractAddress`) 和用户 (`signers.alice.address`) 的加密值。
这意味着只有 Alice 才能使用此加密值，并且只能在特定地址的 `FHECounter.sol` 合约中使用。**它不能被其他用户或在其他合约中重复使用，从而确保了数据机密性和绑定上下文特定的加密。**
{% endhint %}

{% endstep %}

{% step %}

## 使用加密参数调用 `increment()` 函数

现在我们有了一个加密的参数，我们可以用它来调用 `increment()` 函数。

下面，您会注意到更新后的 `increment()` 函数现在接受**两个参数而不是一个。**

这是因为 FHEVM 两者都需要：

1. `externalEuint32`——加密值本身
2. 附带的**零知识知识证明** (`inputProof`)——它验证加密的输入是否安全地绑定到：
   - 调用者（Alice，交易签名者），以及
   - 目标智能合约（正在执行 `increment()` 的地方）

这确保了加密的值不能在不同的上下文或由不同的用户重复使用，从而保持**机密性和完整性。**

替换：

```ts
// const tx = await counterContract.connect(signers.alice).increment(1);
// await tx.wait();
```

替换为以下内容：

```ts
const tx = await fheCounterContract.connect(signers.alice).increment(encryptedOne.handles[0], encryptedOne.inputProof);
await tx.wait();
```

此时，计数器已使用**全同态加密 (FHE)** 成功递增了 1。在下一步中，我们将检索更新后的加密计数器值并将其在本地解密。
但在我们继续之前，让我们快速运行测试以确保一切正常。

---

#### 运行测试

从您项目的根目录运行：

```sh
npx hardhat test
```

#### 预期输出

```sh
  FHECounter
FHECounter has been deployed at address 0x7553CB9124f974Ee475E5cE45482F90d5B6076BC
    ✔ should be deployed
    ✔ encrypted count should be uninitialized after deployment
    ✔ increment the counter by 1


  3 passing (7ms)
```

{% endstep %}

{% step %}

## 调用 `getCount()` 函数并解密值

现在计数器已使用加密输入递增，是时候从智能合约中**读取更新后的加密值**并使用 FHEVM Hardhat 插件提供的 `userDecryptEuint` 函数**解密它**了。

`userDecryptEuint` 函数接受四个参数：

1. **FhevmType**：FHE 加密值的整数类型。在本例中，我们使用 `FhevmType.euint32`，因为计数器是 `uint32`。
2. **加密句柄**：一个 32 字节的 FHEVM 句柄，表示您要解密的加密值。
3. **智能合约地址**：有权访问加密句柄的合约地址。
4. **用户签名者**：有权访问句柄的签名者（例如，signers.alice）。

{% hint style="info" %}
注意：访问 FHEVM 句柄的权限是在链上使用 `FHE.allow()` Solidity 函数设置的（请参阅 FHECounter.sol）。
{% endhint %}

替换：

```ts
// const countAfterInc = await counterContract.getCount();
// expect(countAfterInc).to.eq(countBeforeInc + 1n);
```

替换为以下内容：

```ts
const encryptedCountAfterInc = await fheCounterContract.getCount();
const clearCountAfterInc = await fhevm.userDecryptEuint(
  FhevmType.euint32,
  encryptedCountAfterInc,
  fheCounterContractAddress,
  signers.alice,
);
expect(clearCountAfterInc).to.eq(clearCountBeforeInc + clearOne);
```

---

#### 运行测试

从您项目的根目录运行：

```sh
npx hardhat test
```

#### 预期输出

```sh
  FHECounter
FHECounter has been deployed at address 0x7553CB9124f974Ee475E5cE45482F90d5B6076BC
    ✔ should be deployed
    ✔ encrypted count should be uninitialized after deployment
    ✔ increment the counter by 1


  3 passing (7ms)
```

{% endstep %}

{% step %}

## 调用合约 `decrement()` 函数

与前面的测试类似，我们现在将使用加密输入调用 `decrement()` 函数。

替换：

```ts
//   it("decrement the counter by 1", async function () {
//     // First increment, count becomes 1
//     let tx = await counterContract.connect(signers.alice).increment();
//     await tx.wait();
//     // Then decrement, count goes back to 0
//     tx = await counterContract.connect(signers.alice).decrement(1);
//     await tx.wait();
//     const count = await counterContract.getCount();
//     expect(count).to.eq(0);
//   });
```

替换为以下内容：

```ts
it("decrement the counter by 1", async function () {
  // Encrypt constant 1 as a euint32
  const clearOne = 1;
  const encryptedOne = await fhevm
    .createEncryptedInput(fheCounterContractAddress, signers.alice.address)
    .add32(clearOne)
    .encrypt();

  // First increment by 1, count becomes 1
  let tx = await fheCounterContract.connect(signers.alice).increment(encryptedOne.handles[0], encryptedOne.inputProof);
  await tx.wait();

  // Then decrement by 1, count goes back to 0
  tx = await fheCounterContract.connect(signers.alice).decrement(encryptedOne.handles[0], encryptedOne.inputProof);
  await tx.wait();

  const encryptedCountAfterDec = await fheCounterContract.getCount();
  const clearCountAfterDec = await fhevm.userDecryptEuint(
    FhevmType.euint32,
    encryptedCountAfterDec,
    fheCounterContractAddress,
    signers.alice,
  );

  expect(clearCountAfterDec).to.eq(0);
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
  FHECounter
FHECounter has been deployed at address 0x7553CB9124f974Ee475E5cE45482F90d5B6076BC
    ✔ should be deployed
    ✔ encrypted count should be uninitialized after deployment
    ✔ increment the counter by 1
    ✔ decrement the counter by 1


  4 passing (7ms)
```

{% endstep %}

{% endstepper %}

## 恭喜！您已完成整个教程。

您已成功编写并测试了基于 FHEVM 的计数器智能合约。
到目前为止，您的项目应包含以下文件：

- [`contracts/FHECounter.sol`](https://docs.zama.ai/protocol/examples#tab-fhecounter.sol) — 您的 Solidity 智能合约
- [`test/FHECounter.ts`](https://docs.zama.ai/protocol/examples#tab-fhecounter.ts) — 您用 TypeScript 编写的 Hardhat 测试套件

## 下一步
如果您想在测试网上部署您的项目，或了解有关使用 FHEVM Hardhat 插件的更多信息，请转到[部署合约并运行测试](../../hardhat/run_test.md)。
