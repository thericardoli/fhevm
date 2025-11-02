# 类型转换和普通加密

本文档介绍了 FHE 库为在 FHEVM 中处理加密数据而提供的 `asEbool`、`asEuintXX` 和 `asEaddress` 操作。这些操作对于在明文和加密类型之间进行转换以及处理加密输入至关重要。

这些操作可以分为两个主要用例：

1. **普通加密**：将明文值转换为加密类型
2. **类型转换**：在不同的加密类型之间进行转换

## 1. 普通加密

简单来说，普通加密就是以密文格式存在的明文。

### 概述

普通加密是将明文值转换为与 FHE 运算符兼容的加密类型（密文）的过程。虽然数据采用密文格式，但它在链上仍然是公开可见的，这使其可用于公共值和私有值之间的操作。

这种类型的转换涉及将明文（未加密）值转换为其加密等效项，例如：

- `bool` → `ebool`
- `uint` → `euintXX`
- `address` → `eaddress`

{% hint style="info" %}
进行普通加密时，数据与 FHE 操作兼容，但在链上仍然是公开可见的，除非明确加密。
{% endhint %}

#### **示例**

```solidity
euint64 value64 = FHE.asEuint64(7262);  // 对 uint64 进行普通加密
ebool valueBool = FHE.asEbool(true);   // 对布尔值进行普通加密
```

## 2. 在加密类型之间进行转换

这种类型的转换用于将一种加密类型重新解释或转换为另一种加密类型。例如：

- `euint32` → `euint64`

在处理需要特定大小或精度的操作时，通常需要在加密类型之间进行转换。

> **重要提示**：在加密类型之间进行转换时：
>
> - 从较小的类型转换为较大的类型（例如 `euint32` → `euint64`）会保留所有信息
> - 从较大的类型转换为较小的类型（例如 `euint64` → `euint32`）会截断并丢失信息

下表总结了可用的转换函数：

| 从类型 | 到类型 | 函数 |
| --- | --- | --- |
| `euintX` | `euintX` | `FHE.asEuintXX` |
| `ebool` | `euintX` | `FHE.asEuintXX` |
| `euintX` | `ebool` | `FHE.asEboolXX` |

{% hint style="info" %}
在处理具有不同精度要求的数据时，在加密类型之间进行转换是有效的并且通常是必要的。
{% endhint %}

### **加密类型的工作流程**

```solidity
// 在加密类型之间进行转换
euint32 value32 = FHE.asEuint32(value64); // 转换为 euint32
ebool valueBool = FHE.asEbool(value32);   // 转换为 ebool
```
## 总体操作摘要

| 转换类型 | 函数 | 输入类型 | 输出类型 |
| --- | --- | --- | --- |
| 普通加密 | `FHE.asEuintXX(x)` | `uintX` | `euintX` |
| | `FHE.asEbool(x)` | `bool` | `ebool` |
| | `FHE.asEaddress(x)` | `address` | `eaddress` |
| 类型之间的转换 | `FHE.asEuintXX(x)` | `euintXX`/`ebool` | `euintYY` |
| | `FHE.asEbool(x)` | `euintXX` | `ebool` |
