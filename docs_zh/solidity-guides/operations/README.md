# 对加密类型的操作

本文档概述了 `FHE` 库中对加密类型支持的操作，从而能够在全同态加密 (FHE) 密文上进行算术、按位、比较等操作。

## 算术运算

加密整数（`euintX`）支持以下算术运算：

| 名称 | 函数名称 | 符号 | 类型 |
| --- | --- | --- | --- |
| 加 | `FHE.add` | `+` | 二元 |
| 减 | `FHE.sub` | `-` | 二元 |
| 乘 | `FHE.mul` | `*` | 二元 |
| 除（明文除数） | `FHE.div` | | 二元 |
| 取余（明文除数） | `FHE.rem` | | 二元 |
| 取反 | `FHE.neg` | `-` | 一元 |
| 最小 | `FHE.min` | | 二元 |
| 最大 | `FHE.max` | | 二元 |

{% hint style="info" %}
除法 (FHE.div) 和取余 (FHE.rem) 运算目前仅支持明文除数。
{% endhint %}

## 按位运算

FHE 库还支持按位运算，包括移位和旋转：

| 名称 | 函数名称 | 符号 | 类型 |
| --- | --- | --- | --- |
| 按位与 | `FHE.and` | `&` | 二元 |
| 按位或 | `FHE.or` | `\|` | 二元 |
| 按位异或 | `FHE.xor` | `^` | 二元 |
| 按位非 | `FHE.not` | `~` | 一元 |
| 右移 | `FHE.shr` | | 二元 |
| 左移 | `FHE.shl` | | 二元 |
| 右旋 | `FHE.rotr` | | 二元 |
| 左旋 | `FHE.rotl` | | 二元 |

移位运算符 `FHE.shr` 和 `FHE.shl` 可以接受任何加密类型 `euintX` 作为第一个操作数，以及 `uint8` 或 `euint8` 作为第二个操作数，但是第二个操作数将始终对第一个操作数的位数取模。例如，`FHE.shr(euint64 x, 70)` 等效于 `FHE.shr(euint64 x, 6)`，因为 `70 % 64 = 6`。这与 Solidity 中的经典移位运算符不同，后者没有中间的取模运算，因此例如任何通过 `>>` 右移的 `uint64` 都会得到一个空结果。

## 比较运算

可以使用以下函数比较加密的整数：

| 名称 | 函数名称 | 符号 | 类型 |
| --- | --- | --- | --- |
| 等于 | `FHE.eq` | | 二元 |
| 不等于 | `FHE.ne` | | 二元 |
| 大于或等于 | `FHE.ge` | | 二元 |
| 大于 | `FHE.gt` | | 二元 |
| 小于或等于 | `FHE.le` | | 二元 |
| 小于 | `FHE.lt` | | 二元 |

## 三元运算

`FHE.select` 函数是一个三元运算，它根据一个加密的条件选择两个加密值中的一个：

| 名称 | 函数名称 | 符号 | 类型 |
| --- | --- | --- | --- |
| 选择 | `FHE.select` | | 三元 |

## 随机运算

您可以在链上完全生成加密安全的随机数：

<table data-header-hidden><thead><tr><th></th><th width="206"></th><th></th><th></th></tr></thead><tbody><tr><td><strong>名称</strong></td><td><strong>函数名称</strong></td><td><strong>符号</strong></td><td><strong>类型</strong></td></tr><tr><td>随机无符号整数</td><td><code>FHE.randEuintX()</code></td><td></td><td>随机</td></tr></tbody></table>

有关更多详细信息，请参阅[随机加密数](random.md)文档。

## 最佳实践

在您的智能合约中使用加密操作时，请遵循以下一些最佳实践：

### 使用适当的加密类型大小

选择能够容纳您的数据的最小加密类型以优化 gas 成本。例如，对于小数字（0-255），使用 `euint8` 而不是 `euint256`。

❌ 避免使用过大的类型：

```solidity
// 错误：对小数字使用 euint256 会浪费 gas
euint64 age = FHE.asEuint128(25);  // age 永远不会超过 255
euint64 percentage = FHE.asEuint128(75);  // percentage 是 0-100
```

✅ 相反，使用最小的适当类型：

```solidity
// 正确：使用适当大小的类型
euint8 age = FHE.asEuint8(25);  // age 适合 8 位
euint8 percentage = FHE.asEuint8(75);  // percentage 适合 8 位
```

### 尽可能使用标量操作数以节省 gas

一些 FHE 运算符有两个版本：一个版本的所有操作数都是密文句柄，另一个版本的一个操作数是未加密的标量。只要有可能，就使用标量操作数版本，因为这将节省大量 gas。

❌ 例如，此代码片段的 gas 成本要高得多：

```solidity
euint32 x;
...
x = FHE.add(x,FHE.asEuint(42));
```

✅ 而不是这个：

```solidity
euint32 x;
// ...
x = FHE.add(x,42);
```

尽管两者都会得到相同的加密结果！

### 注意 FHE 算术运算符的溢出

FHE 算术运算符可能会溢出。在实现 FHEVM 智能合约时，不要忘记考虑这种可能性。

❌ 例如，如果您想为一个加密的 ERC20 代币创建一个带有加密的 `totalSupply` 状态变量的 mint 函数，此代码容易受到溢出的影响：

```solidity
function mint(externalEuint32 encryptedAmount, bytes calldata inputProof) public {
  euint32 mintedAmount = FHE.asEuint32(encryptedAmount, inputProof);
  totalSupply = FHE.add(totalSupply, mintedAmount);
  balances[msg.sender] = FHE.add(balances[msg.sender], mintedAmount);
  FHE.allowThis(balances[msg.sender]);
  FHE.allow(balances[msg.sender], msg.sender);
}
```

✅ 但是您可以通过使用 `FHE.select` 在发生溢出的情况下取消 mint 来解决此问题：

```solidity
function mint(externalEuint32 encryptedAmount, bytes calldata inputProof) public {
  euint32 mintedAmount = FHE.asEuint32(encryptedAmount, inputProof);
  euint32 tempTotalSupply = FHE.add(totalSupply, mintedAmount);
  ebool isOverflow = FHE.lt(tempTotalSupply, totalSupply);
  totalSupply = FHE.select(isOverflow, totalSupply, tempTotalSupply);
  euint32 tempBalanceOf = FHE.add(balances[msg.sender], mintedAmount);
  balances[msg.sender] = FHE.select(isOverflow, balances[msg.sender], tempBalanceOf);
  FHE.allowThis(balances[msg.sender]);
  FHE.allow(balances[msg.sender], msg.sender);
}
```

请注意，我们没有单独检查 `balances[msg.sender]` 的溢出，而只检查了 `totalSupply` 变量，因为 `totalSupply` 是所有用户余额的总和，所以如果 `totalSupply` 没有溢出，`balances[msg.sender]` 也不可能溢出。
