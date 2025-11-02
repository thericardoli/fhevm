# 处理分支和条件

本文档介绍了在使用全同态加密 (FHE) 时如何处理分支、循环或条件，特别是当条件/索引被加密时。

## 中断循环

❌ 在 FHE 中，不可能根据加密的条件中断循环。例如，这将不起作用：

```solidity
euint8 maxValue = FHE.asEuint(6); // 可能是 0 到 10 之间的值
euint8 x = FHE.asEuint(0);
// 一些代码
while(FHE.lt(x, maxValue)){
    x = FHE.add(x, 2);
}
```

如果您的代码逻辑需要在加密的布尔条件下循环，我们强烈建议您尝试用一个具有适当恒定最大步数的有限循环来替换它，并在循环内部使用 `FHE.select`。

## 建议的方法

✅ 例如，前面的代码也许可以用以下代码片段替换：

```solidity
euint8 maxValue = FHE.asEuint(6); // 可能是 0 到 10 之间的值
euint8 x;
// 一些代码
for (uint32 i = 0; i < 10; i++) {
    euint8 toAdd = FHE.select(FHE.lt(x, maxValue), 2, 0);
    x = FHE.add(x, toAdd);
}
```

在此代码片段中，我们执行 10 次迭代，只要迭代次数小于 `maxValue`，就在每次迭代中向 `x` 添加 4。如果迭代次数超过 `maxValue`，我们将在剩余的迭代中改为添加 0，因为我们无法中断循环。

## 最佳实践

### 混淆分支

前一段强调，分支逻辑应尽可能依赖于 `FHE.select` 而不是解密。它有效地隐藏了执行了哪个分支。

然而，这有时还不够。增强智能合约的隐私性通常需要重新审视您的应用程序的逻辑。

例如，如果基于线性常数函数为两个加密的 ERC20 代币实现一个简单的 AMM，建议不仅要隐藏正在交换的金额，还要隐藏在一对中交换的代币。

✅ 这是一个非常简化的示例实现，我们在这里假设 tokenA 和 tokenB 之间的汇率是恒定的并且等于 1：

```solidity
// 通常 encryptedAmountAIn 或 encryptedAmountBIn 是一个加密的空值
// 理想情况下，用户已经拥有两种代币的一些数量，并且已经在两种代币上预先批准了 AMM
function swapTokensForTokens(
  externalEuint32 encryptedAmountAIn,
  externalEuint32 encryptedAmountBIn,
  bytes calldata inputProof
) external {
  euint32 encryptedAmountA = FHE.asEuint32(encryptedAmountAIn, inputProof); // 即使金额为空，也进行一次转移以混淆交易方向
  euint32 encryptedAmountB = FHE.asEuint32(encryptedAmountBIn, inputProof); // 即使金额为空，也进行一次转移以混淆交易方向

  // 将代币从用户发送到 AMM 合约
  FHE.allowTransient(encryptedAmountA, tokenA);
  IConfidentialERC20(tokenA).transferFrom(msg.sender, address(this), encryptedAmountA);

  FHE.allowTransient(encryptedAmountB, tokenB);
  IConfidentialERC20(tokenB).transferFrom(msg.sender, address(this), encryptedAmountB);

  // 将代币从 AMM 合约发送到用户
  // tokenA 在 tokenB 中的价格是恒定的并且等于 1，所以我们在这里只交换加密的金额
  FHE.allowTransient(encryptedAmountB, tokenA);
  IConfidentialERC20(tokenA).transfer(msg.sender, encryptedAmountB);

  FHE.allowTransient(encryptedAmountA, tokenB);
  IConfidentialERC20(tokenB).transferFrom(msg.sender, address(this), encryptedAmountA);
}
```

请注意，为了保护机密性，我们必须在用户和 AMM 合约之间对两种代币进行两次输入转移，同样地，从 AMM 到用户也进行两次输出转移，即使技术上讲，大多数情况下，用户输入 `encryptedAmountAIn` 或 `encryptedAmountBIn` 中的一个实际上是加密的零。

这与使用常规 ERC20 代币的经典非机密 AMM 不同：在这种情况下，用户只需在要出售的代币上向 AMM 进行一次输入转移，并在要购买的代币上从 AMM 接收一次输出转移。

### 避免使用加密索引

使用加密索引从数组中选择一个元素而不泄露它不是很有效，因为您仍然需要循环遍历所有索引以保护机密性。

然而，有计划在未来通过为数组添加专门的运算符来使这种操作更加高效。

例如，假设您有一个名为 `encArray` 的加密数组，并且您想更新一个加密值 `x` 以匹配此列表中的一项 `encArray[i]`，而_不_泄露您选择的是哪一项。

❌ 您必须循环遍历所有索引并同态地检查相等性，但是这种模式在 gas 方面非常昂贵，应尽可能避免。

```solidity
euint32 x;
euint32[] encArray;

function setXwithEncryptedIndex(externalEuint32 encryptedIndex, bytes calldata inputProof) public {
    euint32 index = FHE.asEuint32(encryptedIndex, inputProof);
    for (uint32 i = 0; i < encArray.length; i++) {
        ebool isEqual = FHE.eq(index, i);
        x = FHE.select(isEqual, encArray[i], x);
    }
    FHE.allowThis(x);
}
```
