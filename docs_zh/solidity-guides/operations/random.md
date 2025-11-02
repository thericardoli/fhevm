# 生成随机数

本文档介绍了如何使用 fhevm 中的 `FHE` 库在链上完全生成加密安全的随机加密数。这些数字是加密的并且保持机密，从而实现了保护隐私的智能合约逻辑。

## **关于随机数生成的关键说明**

- **链上执行**：随机数生成必须在交易期间执行，因为它要求在链上更新伪随机数生成器 (PRNG) 的状态。此操作不能使用 `eth_call` RPC 方法执行。
- **加密安全**：生成的随机数是加密安全的并且是加密的，确保了隐私性和不可预测性。

{% hint style="info" %}
随机数生成必须在交易期间执行，因为它要求在链上改变伪随机数生成器 (PRNG) 的状态。因此，它不能使用 `eth_call` RPC 方法执行。
{% endhint %}

## **基本用法**

`FHE` 库允许您生成各种位大小的随机加密数。以下是支持的类型及其用法的列表：

```solidity
// 生成随机加密数
ebool rb = FHE.randEbool();       // 随机加密布尔值
euint8 r8 = FHE.randEuint8();     // 随机 8 位数
euint16 r16 = FHE.randEuint16();  // 随机 16 位数
euint32 r32 = FHE.randEuint32();  // 随机 32 位数
euint64 r64 = FHE.randEuint64();  // 随机 64 位数
euint128 r128 = FHE.randEuint128(); // 随机 128 位数
euint256 r256 = FHE.randEuint256(); // 随机 256 位数
```

### **示例：随机布尔值**

```solidity
function randomBoolean() public returns (ebool) {
  return FHE.randEbool();
}
```

## **有界随机数**

要生成特定范围内的随机数，您可以指定一个**上限**。指定的上限必须是 2 的幂。随机数将在 `[0, upperBound - 1]` 范围内。

```solidity
// 生成带上限的随机数
euint8 r8 = FHE.randEuint8(32);      // 0-31 之间的随机数
euint16 r16 = FHE.randEuint16(512);  // 0-511 之间的随机数
euint32 r32 = FHE.randEuint32(65536); // 0-65535 之间的随机数
```

### **示例：带上限的随机数**

```solidity
function randomBoundedNumber(uint16 upperBound) public returns (euint16) {
  return FHE.randEuint16(upperBound);
}
```

## **安全注意事项**

- **加密安全**：
  随机数是使用加密安全的伪随机数生成器 (CSPRNG) 生成的，并且在明确解密之前保持加密状态。
- **Gas 消耗**：
  每次调用随机数生成函数都会消耗 gas。开发人员应优化这些函数的使用，尤其是在对 gas 敏感的合约中。
- **隐私保证**：
  随机值是完全加密的，确保未经授权的各方无法访问或预测它们。
