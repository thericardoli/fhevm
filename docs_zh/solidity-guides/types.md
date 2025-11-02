# 支持的类型

本文档介绍了 FHEVM 中 `FHE` 库提供的加密整数类型，并解释了它们的用法，包括类型转换、状态变量声明和特定于类型的注意事项。

## 简介

`FHE` 库提供了一个具有加密整数类型的强大类型系统，可在智能合约中对机密数据进行安全计算。这些加密类型在编译时和运行时都经过验证，以确保正确性和安全性。

### 加密类型的主要特点

- 加密整数的功能与 Solidity 的本机整数类型类似，但它们对**全同态加密 (FHE)** 密文进行操作。
- `e(u)int` 类型上的算术运算是**未经检查的**，这意味着它们在溢出时会回绕。这种设计选择通过避免通过错误检测泄漏信息来确保机密性。
- `FHE` 库的未来版本将支持带溢出检查的加密整数，但代价是会暴露有关操作数的有限信息。

{% hint style="info" %}
带溢出检查的加密整数很快就会在 `FHE` 库中提供。这些将允许可逆的算术运算，但可能会泄露有关输入值的一些信息。
{% endhint %}

FHEVM 中的加密整数表示为 FHE 密文，使用密文句柄进行抽象。这些类型以 `e` 为前缀（例如 `euint64`），充当密文句柄的安全包装器。

## 加密类型列表

`FHE` 库目前支持以下加密类型：

| 类型 | 位长 | 支持的运算符 | 别名（带支持的运算符） |
| --- | --- | --- | --- |
| Ebool | 2 | and, or, xor, eq, ne, not, select, rand | |
| Euint8 | 8 | add, sub, mul, div, rem, and, or, xor, shl, shr, rotl, rotr, eq, ne, ge, gt, le, lt, min, max, neg, not, select, rand, randBounded | |
| Euint16 | 16 | add, sub, mul, div, rem, and, or, xor, shl, shr, rotl, rotr, eq, ne, ge, gt, le, lt, min, max, neg, not, select, rand, randBounded | |
| Euint32 | 32 | add, sub, mul, div, rem, and, or, xor, shl, shr, rotl, rotr, eq, ne, ge, gt, le, lt, min, max, neg, not, select, rand, randBounded | |
| Euint64 | 64 | add, sub, mul, div, rem, and, or, xor, shl, shr, rotl, rotr, eq, ne, ge, gt, le, lt, min, max, neg, not, select, rand, randBounded | |
| Euint128 | 128 | add, sub, mul, div, rem, and, or, xor, shl, shr, rotl, rotr, eq, ne, ge, gt, le, lt, min, max, neg, not, select, rand, randBounded | |
| Euint160 | 160 | | Eaddress (eq, ne, select) |
| Euint256 | 256 | and, or, xor, shl, shr, rotl, rotr, eq, ne, neg, not, select, rand, randBounded | |

{% hint style="info" %}
除法 (`div`) 和取余 (`rem`) 运算仅在右侧 (`rhs`) 操作数是明文（未加密）值时才受支持。尝试使用加密值作为 `rhs` 将导致紧急情况。此限制确保了当前框架内的正确和安全计算。
{% endhint %}

{% hint style="info" %}
`TFHE-rs` 库中提供了更高精度的整数类型，可以根据需要添加到 `fhevm` 中。
{% endhint %}
