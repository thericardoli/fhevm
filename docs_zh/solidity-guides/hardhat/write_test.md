在本节中，您将找到设置一个新的 [Hardhat](https://hardhat.org) 项目并使用 [FHEVM Hardhat 插件](https://www.npmjs.com/package/@fhevm/hardhat-plugin) 从头开始开发 FHEVM 智能合约所需的一切。

## 在您的 Hardhat 项目中启用 FHEVM Hardhat 插件

与任何 Hardhat 插件一样，必须通过将以下 `import` 语句添加到您的 `hardhat.config.ts` 文件中来启用 [FHEVM Hardhat 插件](https://www.npmjs.com/package/@fhevm/hardhat-plugin)：

```typescript
import "@fhevm/hardhat-plugin";
```

{% hint style="warning" %}
没有此导入，Hardhat FHEVM API 将**不**在您的 Hardhat 运行时环境 (HRE) 中可用。
{% endhint %}

## 访问 Hardhat FHEVM API

该插件使用新的 `fhevm` Hardhat 模块扩展了标准的 [Hardhat 运行时环境](https://hardhat.org/hardhat-runner/docs/advanced/hardhat-runtime-environment)（或简称 `hre`）。

您可以通过以下任一方式访问它：

```typescript
import { fhevm } from "hardhat";
```

或

```typescript
import * as hre from "hardhat";

// 然后访问： hre.fhevm
```

## 使用 Hardhat FHEVM API 加密值

假设您要测试的 FHEVM 智能合约有一个名为 `foo` 的函数，该函数接受一个加密的 `uint32` 值作为输入。Solidity 函数 `foo` 应声明如下：

```solidity
function foo(externalEunit32 value, bytes calldata memory inputProof);
```

其中：

- `externalEunit32 value`：是一个 `bytes32`，表示加密的 `uint32`
- `bytes calldata memory inputProof`：是一个 `bytes` 数组，表示验证加密的零知识知识证明

要在 TypeScript 中计算这些参数，您需要：

- **目标智能合约的地址**
- **签名者的地址**（即发送交易的帐户）

{% stepper %}

{% step %}

#### 创建一个新的加密输入

```ts
// 使用 Hardhat 运行时环境中的 `fhevm` API 模块
const input = fhevm.createEncryptedInput(contractAddress, signers.alice.address);
```

{% endstep %}

{% step %}

#### 添加您要加密的值。

```ts
input.add32(12345);
```

{% endstep %}

{% step %}

#### 执行本地加密。

```ts
const encryptedInputs = await input.encrypt();
```

{% endstep %}

{% step %}

#### 调用 Solidity 函数

```ts
const externalUint32Value = encryptedInputs.handles[0];
const inputProof = encryptedInputs.proof;

const tx = await input.foo(externalUint32Value, inputProof);
await tx.wait();
```

{% endstep %}

{% endstepper %}

### 加密示例

- [基本加密示例](https://docs.zama.ai/protocol/examples/basic/encryption)
- [FHECounter](https://docs.zama.ai/protocol/examples#an-fhe-counter)

## 使用 Hardhat FHEVM API 解密值

假设用户 **Alice** 想要解密存储在智能合约中的 `euint32` 值，该合约公开了以下
Solidity `view` 函数：

```solidity
function getEncryptedUint32Value() public view returns (euint32) { returns _encryptedUint32Value; }
```

{% hint style="warning" %}
为简单起见，我们假设 Alice 的帐户和目标智能合约都已具有解密此值所需的 FHE 权限。有关 FHE 权限如何工作的详细说明，请参阅 [DecryptSingleValue.sol](https://docs.zama.ai/protocol/examples/basic/decryption/fhe-decrypt-single-value#tab-decryptsinglevalue.sol) 中的 [`initializeUint32()`](https://docs.zama.ai/protocol/examples/basic/decryption/fhe-decrypt-single-value#tab-decryptsinglevalue.sol) 函数。
{% endhint %}

{% stepper %}

{% step %}

#### 从智能合约中检索加密值（一个 `bytes32` 句柄）：

```ts
const encryptedUint32Value = await contract.getEncryptedUint32Value();
```

{% endstep %}

{% step %}

#### 使用 FHEVM API 执行解密：

```ts
const clearUint32Value = await fhevm.userDecryptEuint(
  FhevmType.euint32, // 加密类型（必须与 Solidity 类型匹配）
  encryptedUint32Value, // Alice 想要解密的 bytes32 句柄
  contractAddress, // 目标合约地址
  signers.alice, // Alice 的钱包
);
```

{% hint style="warning" %}
如果目标智能合约或用户**没有** FHE 权限，则解密调用将失败！
{% endhint %}

{% endstep %}

{% endstepper %}

### 支持的解密类型

为每个加密数据类型使用适当的函数：

| 类型 | 函数 |
| --- | --- |
| `euintXXX` | `fhevm.userDecryptEuint(...)` |
| `ebool` | `fhevm.userDecryptEbool(...)` |
| `eaddress` | `fhevm.userDecryptEaddress(...)` |

### 解密示例

- [基本解密示例](https://docs.zama.ai/protocol/examples/basic/decryption)
- [FHECounter](https://docs.zama.ai/protocol/examples#an-fhe-counter)
