# FHE 中的分支

本文档介绍了在 FHEVM 中使用加密值时如何实现条件逻辑（if/else 分支）。与典型的 Solidity 编程不同，使用全同态加密 (FHE) 需要专门的方法来处理加密数据上的条件。

本文档涵盖了加密分支以及如何在您的智能合约中从加密条件转移到非加密业务逻辑。

## 什么是机密分支？

在 FHEVM 中，当您执行[比较操作](../operations/README.md#comparison-operations)时，结果是一个加密的布尔值 (`ebool`)。由于加密的布尔值不支持像 `if` 语句或逻辑运算符这样的标准布尔运算，因此必须使用专门的方法来实现条件逻辑。

为了便于条件赋值，FHEVM 提供了 `FHE.select` 函数，它充当加密值的三元运算符。

## **使用 `FHE.select` 实现条件逻辑**

`FHE.select` 函数通过根据加密条件 (`ebool`) 选择两个加密值中的一个来实现分支逻辑。它的工作原理如下：

```solidity
FHE.select(condition, valueIfTrue, valueIfFalse);
```

- **`condition`**：比较产生的加密布尔值 (`ebool`)。
- **`valueIfTrue`**：如果条件为真，则返回的加密值。
- **`valueIfFalse`**：如果条件为假，则返回的加密值。

## **示例：拍卖出价逻辑**

以下是在猜谜游戏中用于更新最高中奖号码的条件逻辑示例：

```solidity
function bid(externalEuint64 encryptedValue, bytes calldata inputProof) external onlyBeforeEnd {
  // 将加密输入转换为加密的 64 位整数
  euint64 bid = FHE.asEuint64(encryptedValue, inputProof);

  // 将当前最高出价与新出价进行比较
  ebool isAbove = FHE.lt(highestBid, bid);

  // 如果新出价更高，则更新最高出价
  highestBid = FHE.select(isAbove, bid, highestBid);

  // 允许合约使用更新后的最高出价密文
  FHE.allowThis(highestBid);
}
```

{% hint style="info" %}
这是一个简化的示例，旨在演示功能。
{% endhint %}

### 它是如何工作的？

- **比较**：
  - `FHE.lt` 函数比较 `highestBid` 和 `bid`，返回一个 `ebool` (`isAbove`)，指示新出价是否更高。
- **选择**：
  - `FHE.select` 函数根据加密条件 `isAbove` 将 `highestBid` 更新为新出价或先前的最高出价。
- **权限处理**：
  - 更新 `highestBid` 后，合约使用 `FHE.allowThis` 重新授权自己操作更新后的密文。

## 关键注意事项

- **值更改行为：** 每次 `FHE.select` 赋值时，都会创建一个新的密文，即使底层的明文值保持不变。这种行为是 FHE 固有的，可确保数据机密性，但开发人员在设计智能合约时应考虑到这一点。
- **Gas 消耗：** 与传统的 Solidity 逻辑相比，使用 `FHE.select` 和其他加密操作会产生额外的 gas 成本。优化您的代码以最大限度地减少不必要的操作。
- **访问控制：** 始终使用适当的 ACL 函数（例如 `FHE.allowThis`、`FHE.allow`）来确保更新后的密文被授权用于未来的计算或交易。

---

## 如何分支到非机密路径？

到目前为止，本节仅介绍了如何使用加密变量进行分支。但是，在许多情况下，“公共”合约逻辑将取决于加密路径的结果。

为此，只有一种方法可以从加密路径分支到非加密路径：它需要使用预言机进行公共解密。因此，任何需要从加密输入转移到非加密路径的合约逻辑始终需要异步合约逻辑。

## **示例：拍卖出价逻辑：物品发放**

回到我们之前关于拍卖出价逻辑的示例。假设拍卖的获胜者可以获得一些非机密的奖品。

```solidity
bool public isPrizeDistributed;
eaddress internal highestBidder;
euint64 internal highestBid;

function bid(externalEuint64 encryptedValue, bytes calldata inputProof) external onlyBeforeEnd {
  // 将加密输入转换为加密的 64 位整数
  euint64 bid = FHE.asEuint64(encryptedValue, inputProof);

  // 将当前最高出价与新出价进行比较
  ebool isAbove = FHE.lt(highestBid, bid);

  // 如果新出价更高，则更新最高出价
  highestBid = FHE.select(isAbove, bid, highestBid);

  // 如果新出价更高，则更新最高出价者地址
  highestBidder = FHE.select(isAbove, FHE.asEaddress(msg.sender), currentBidder));

  // 允许合约使用最高出价者地址
  FHE.allowThis(highestBidder);

  // 允许合约使用更新后的最高出价密文
  FHE.allowThis(highestBid);
}

function revealWinner() external onlyAfterEnd {
  bytes32[] memory cts = new bytes32[](2);
  cts[0] = FHE.toBytes32(highestBidder);
  uint256 requestId = FHE.requestDecryption(cts, this.transferPrize.selector);
}

function transferPrize(uint256 requestId, address auctionWinner, bytes memory signatures) external {
  require(!isPrizeDistributed, "Prize has already been distributed");
  FHE.verifySignatures(requestId, signatures)

  isPrizeDistributed = true;
  // 将奖品转移给拍卖获胜者的业务逻辑
}
```

{% hint style="info" %}
这是一个简化的示例，旨在演示功能。
{% endhint %}

正如您在上面的示例中所看到的，从加密条件转移到解密业务逻辑的路径必须是异步的，并且需要调用解密预言机合约以使用加密变量显示逻辑的结果。

## 总结

- **`FHE.select`** 是对加密值进行条件逻辑的强大工具。
- 加密的布尔值 (`ebool`) 和值保持机密性，从而实现保护隐私的逻辑。
- 在设计条件操作时，开发人员应考虑 gas 成本和密文行为。
