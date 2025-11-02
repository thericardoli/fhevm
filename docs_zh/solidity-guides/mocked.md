# 模拟模式

本文档概述了 FHEVM 框架中的模拟模式，解释了它如何实现使用全同态加密 (FHE) 的智能合约的更快开发和测试。

## 概述

**模拟模式**是 FHEVM 框架中提供的一项开发和测试功能，允许开发人员模拟全同态加密 (FHE) 的行为，而无需执行完整的加密和解密过程。这通过用模拟值替换实际的加密操作来使开发和测试周期更快、更高效，这些模拟值的行为类似于加密数据，但没有真正加密的计算开销。

## 如何使用模拟模式

### 1. **Hardhat 模板**

模拟模式目前在 [Zama Hardhat 模板](https://github.com/zama-ai/fhevm-hardhat-template) 中受支持。开发人员可以启用模拟模式来在本地构建和测试智能合约时模拟加密操作。Hardhat 模板包含预配置的脚本和库，以简化模拟模式的设置过程。

有关使用 Hardhat 模板的说明，请参阅 [[快速入门 - Hardhat]](../getting-started/overview-1/hardhat/README.md) 指南。

### 2. **Foundry (即将推出)**

模拟模式支持计划在未来版本中用于 [Foundry](./write_contract/foundry.md)。

## 模拟模式的工作原理

为了加快测试迭代速度，您可以使用 FHEVM 的模拟版本，方法是运行 `pnpm test`，而不是在本地 FHEVM 节点上启动所有测试，这可能需要几分钟。相同的测试应该（几乎总是）按原样通过，无需任何修改；JavaScript 文件和 Solidity 文件都无需在模拟版本和真实版本之间进行更改。

模拟模式**实际上并不会**为加密类型执行真正的加密，而是在本地 Hardhat 节点上运行测试，该节点实现了原始的 EVM（即非 FHEVM）。

此外，模拟模式将让您使用所有与 Hardhat 相关的特殊测试和调试方法，例如 `evm_mine`、`evm_snapshot`、`evm_revert` 等，这些方法对于测试非常有帮助。

## 使用模拟模式的开发工作流程

在开发机密合约时，我们建议首先使用 FHEVM 的模拟版本，以便通过 `pnpm test` 进行更快的测试，并通过 `pnpm coverage` 进行覆盖率计算，这将带来更好的开发人员体验。

使用真实的 FHEVM 运行最终合约版本的测试至关重要。您可以在部署前通过运行 `pnpm test` 来执行此操作。

要运行模拟测试，请使用以下任一命令：

```sh
pnpm test
```

或等效地：

```sh
npx hardhat test --network hardhat
```

在模拟模式下，所有测试都应在几秒钟内通过，而不是几分钟，从而提供更好的开发人员体验。此外，只有在模拟模式下才能获得测试的覆盖率。只需使用以下命令：

```
pnpm coverage
```

或等效地：

```
npx hardhat coverage
```

然后打开 `coverage/index.html` 文件以查看覆盖率结果。这将通过指出当前测试套件尚未覆盖的缺失分支来提高安全性。

{% hint style="info" %}
由于 `solidity-coverage` 包的限制，测试覆盖率计算不适用于使用 `evm_snapshot` Hardhat 测试方法的测试。
{% endhint %}

如果您在测试中使用 Hardhat 快照，我们建议在测试描述的末尾添加 `[skip-on-coverage]` 标签。这是一个例子：

```js
import { expect } from 'chai';
import { ethers, network } from 'hardhat';

import { createInstances, decrypt8, decrypt16, decrypt32, decrypt64 } from '../instance';
import { getSigners, initSigners } from '../signers';
import { deployRandFixture } from './Rand.fixture';

describe('Rand', function () {
  before(async function () {
    await initSigners();
    this.signers = await getSigners();
  });

  beforeEach(async function () {
    const contract = await deployRandFixture();
    this.contractAddress = await contract.getAddress();
    this.rand = contract;
    this.instances = await createInstances(this.signers);
  });

  it('64 位生成带上限并解密', async function () {
    const values: bigint[] = [];
    for (let i = 0; i < 5; i++) {
      const txn = await this.rand.generate64UpperBound(262144);
      await txn.wait();
      const valueHandle = await this.rand.value64();
      const value = await decrypt64(valueHandle);
      expect(value).to.be.lessThanOrEqual(262141);
      values.push(value);
    }
    // 期望至少有两个不同的生成值。
    const unique = new Set(values);
    expect(unique.size).to.be.greaterThanOrEqual(2);
  });

  it('8 位和 16 位生成并使用 hardhat 快照解密 [skip-on-coverage]', async function () {
    if (network.name === 'hardhat') {
      // 快照仅在 hardhat 节点中可用，即在模拟模式下
      this.snapshotId = await ethers.provider.send('evm_snapshot');
      const values: number[] = [];
      for (let i = 0; i < 5; i++) {
        const txn = await this.rand.generate8();
        await txn.wait();
        const valueHandle = await this.rand.value8();
        const value = await decrypt8(valueHandle);
        expect(value).to.be.lessThanOrEqual(0xff);
        values.push(value);
      }
      // 期望至少有两个不同的生成值。
      const unique = new Set(values);
      expect(unique.size).to.be.greaterThanOrEqual(2);

      await ethers.provider.send('evm_revert', [this.snapshotId]);
      const values2: number[] = [];
      for (let i = 0; i < 5; i++) {
        const txn = await this.rand.generate8();
        await txn.wait();
        const valueHandle = await this.rand.value8();
        const value = await decrypt8(valueHandle);
        expect(value).to.be.lessThanOrEqual(0xff);
        values2.push(value);
      }
      // 期望至少有两个不同的生成值。
      const unique2 = new Set(values2);
      expect(unique2.size).to.be.greaterThanOrEqual(2);
    }
  });
});
```

- **第一个测试**始终在模拟模式 (`pnpm test`) 和覆盖率模式 (`pnpm coverage`) 下运行。
- **第二个测试**仅在测试模拟模式 (`pnpm test`) 时运行，因为它使用的快照仅在该模式下可用。由于其描述中的 `[skip-on-coverage]` 后缀，该测试在覆盖率模式下被跳过。它还检查网络是否为“hardhat”，以避免在非模拟环境中失败。
