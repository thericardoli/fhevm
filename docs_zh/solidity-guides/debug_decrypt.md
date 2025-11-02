# 使用 `debug.decrypt[XX]` 进行调试

本指南介绍了如何在 FHEVM 开发期间使用 `debug.decrypt[XX]` 函数在模拟环境中调试加密数据。

{% hint style="warning" %}
`debug.decrypt[XX]` 函数不应在生产环境中使用，因为它们依赖于私钥。
{% endhint %}

## 概述

`debug.decrypt[XX]` 函数允许您将加密的句柄解密为明文值。此功能对于调试加密操作非常有用，例如涉及 FHE 加密数据的传输、余额检查和其他计算。

### 要点

- **环境**：`debug.decrypt[XX]` 函数**仅在模拟环境**（例如 `hardhat` 网络）中有效。
- **生产限制**：在生产环境中，解密是通过中继器异步执行的，并且需要授权的链上请求。
- **加密类型**：`debug.decrypt[XX]` 函数支持各种加密类型，包括整数和布尔值。
- **绕过 ACL 授权**：`debug.decrypt[XX]` 函数允许在没有 ACL 授权的情况下进行解密，这对于在开发和测试期间验证加密操作非常有用。

## 支持的函数

### 整数解密

解密不同位宽的加密整数（`euint8`、`euint16`、...、`euint256`）。

| 函数名称 | 返回值 | 加密类型 |
| --- | --- | --- |
| `decrypt8` | `bigint` | `euint8` |
| `decrypt16` | `bigint` | `euint16` |
| `decrypt32` | `bigint` | `euint32` |
| `decrypt64` | `bigint` | `euint64` |
| `decrypt128` | `bigint` | `euint128` |
| `decrypt256` | `bigint` | `euint256` |

### 布尔值解密

解密加密的布尔值（`ebool`）。

| 函数名称 | 返回值 | 加密类型 |
| --- | --- | --- |
| `decryptBool` | `boolean` | `ebool` |

### 地址解密

解密加密的地址。

| 函数名称 | 返回值 | 加密类型 |
| --- | --- | --- |
| `decryptAddress` | `string` | `eaddress` |

## 函数用法

### 示例：解密加密值

```typescript
import { debug } from "../utils";

// 解密一个 64 位加密整数
const handle64: bigint = await this.erc20.balanceOf(this.signers.alice);
const plaintextValue: bigint = await debug.decrypt64(handle64);
console.log("解密的余额：", plaintextValue);
```

{% hint style="info" %}
要使用调试函数，请导入 [utils.ts](https://github.com/zama-ai/fhevm-hardhat-template/blob/main/test/utils.ts) 文件。
{% endhint %}

有关更完整的示例，请参阅 [ConfidentialERC20 测试文件](https://github.com/zama-ai/fhevm-hardhat-template/blob/f9505a67db31c988f49b6f4210df47ca3ce97841/test/confidentialERC20/ConfidentialERC20.ts#L181-L205)。

### 示例：解密字节数组

```typescript
// 解密一个 128 字节的加密值
const ebytes128Handle: bigint = ...; // 获取加密字节的句柄
const decryptedBytes: string = await debug.decryptEbytes128(ebytes128Handle);
console.log("解密的字节：", decryptedBytes);
```

## **工作原理**

### 验证类型

每个解密函数都包含一个**类型验证步骤**，以确保提供的句柄与预期的加密类型匹配。如果类型不匹配，则会抛出错误。

```typescript
function verifyType(handle: bigint, expectedType: number) {
  const typeCt = handle >> 8n;
  if (Number(typeCt % 256n) !== expectedType) {
    throw "句柄的加密类型错误";
  }
}
```

### 环境检查

{% hint style="danger" %}
这些函数仅在 `hardhat` 网络中有效。尝试在生产环境中使用它们将导致错误。
{% endhint %}

```typescript
if (network.name !== "hardhat") {
  throw Error("此函数只能在模拟模式下调用");
}
```

## **最佳实践**

- **仅用于调试**：这些函数需要访问私钥，并且专门用于本地测试和调试。
- **生产解密**：对于生产环境，请始终使用基于中继器的异步解密。
