本教程介绍了如何使用全同态加密（FHE）构建一个密封投标的 NFT 拍卖。在该系统中，参与者为一个 NFT 提交加密的出价。在拍卖期间，出价保持机密，只有获胜者的信息在最后才会揭晓。

通过本指南，您将学习如何：

- 接受和处理加密的出价
- 在不泄露其价值的情况下安全地比较出价
- 在拍卖结束后揭晓获胜者
- 设计一个私密、公平和透明的拍卖

# 为什么选择 FHE

在大多数链上拍卖中，**出价是完全公开的**。任何人都可以检查区块链或监控待处理的交易，以查看每个参与者出价多少。这破坏了公平性，因为获胜只需要发送一个比当前最高价高一 wei 的新出价。

现有的解决方案，如承诺-揭示方案，试图在初步的承诺阶段隐藏出价。然而，它们有几个缺点：增加了交易开销、用户体验差（例如，要求用户通过 `CREATE2` 将资金发送到 EOA），以及由于需要多个拍卖阶段而导致的延迟。

全同态加密（FHE）使参与者能够一步直接向智能合约提交加密的出价，消除了多阶段的复杂性，改善了用户体验，并在不泄露或解密的情况下保护了出价的机密性。

# 项目设置

在开始本教程之前，请确保您已：

1. 安装了 FHEVM hardhat 模板
2. 设置了 OpenZeppelin 机密合约库
3. 部署了您的机密代币

有关这些步骤的帮助，请参阅以下教程：
- [设置 OpenZeppelin 机密合约](./openzeppelin/README.md)
- [部署机密代币](./openzeppelin/erc7984-tutorial.md)

# 创建智能合约

现在，让我们在 `./contracts/` 文件夹中创建一个名为 `BlindAuction.sol` 的新合约。要在我们的合约中启用 FHE 操作，我们需要让我们的合约继承自 `SepoliaConfig`。此配置提供了与 Zama 的 FHEVM 交互所需的必要参数和特定于网络的设置。

我们还需要创建一些将在我们的拍卖中使用的状态变量。
对于支付，我们将依赖于一个 `ConfidentialFungibleToken`。事实上，我们不能使用传统的 ERC20，因为即使我们拍卖中的状态是私密的，任何人仍然可以监控区块链交易并猜测出价。通过使用 `ConfidentialFungibleToken`，我们确保金额保持隐藏。这个 `ConfidentialFungibleToken` 可以与任何 ERC20 一起使用，您只需要包装您的代币以隐藏未来的转账。

我们的合约还将包含一个代表正在拍卖的 NFT 的 `ERC721` 代币和拍卖受益人的地址。最后，我们将定义一些与时间相关的参数来控制拍卖的持续时间。

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, externalEuint64, euint64, ebool } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";
import {ConfidentialFungibleToken} from "@openzeppelin/confidential-contracts/token/ConfidentialFungibleToken.sol";
// ...

contract BlindAuction is SepoliaConfig {
  /// @notice 拍卖结束后最高出价的接收者
  address public beneficiary;

  /// @notice 机密支付代币
  ConfidentialFungibleToken public confidentialFungibleToken;

  /// @notice 拍卖代币
  IERC721 public nftContract;
  uint256 public tokenId;

  /// @notice 拍卖持续时间
  uint256 public auctionStartTime;
  uint256 public auctionEndTime;

  // ...

  constructor(
    address _nftContractAddress,
    address _confidentialFungibleTokenAddress,
    uint256 _tokenId,
    uint256 _auctionStartTime,
    uint256 _auctionEndTime
  ) {
    beneficiary = msg.sender;
    confidentialFungibleToken = ConfidentialFungibleToken(_confidentialFungibleTokenAddress);
    nftContract = IERC721(_nftContractAddress);

    // 将 NFT 转移到合约中进行拍卖
    nftContract.safeTransferFrom(msg.sender, address(this), _tokenId);

    require(_auctionStartTime < _auctionEndTime, "INVALID_TIME");
    auctionStartTime = _auctionStartTime;
    auctionEndTime = _auctionEndTime;
  }

  // ...
}
```

现在，我们需要一种方法来存储最高出价和潜在的获胜者。要私密地存储这些信息，我们将使用 FHE 库提供的一些工具。对于存储加密地址，我们可以使用 `eaddress` 类型，对于最高出价，我们可以使用 `euint64` 存储金额。此外，我们可以创建一个映射来跟踪用户的出价。

```solidity
/// @notice 加密的拍卖信息
euint64 private highestBid;
eaddress private winningAddress;

/// @notice 从出价者到其出价金额的映射
mapping(address account => euint64 bidAmount) private bids;
```

{% hint style="info" %}

您可能会注意到，在我们的代码中，我们使用的是 euint64，它代表一个加密的 64 位无符号整数。与标准的 Solidity 类型不同，uint64 和 uint256 之间的差异不大，但在 FHE 中，数据的大小对性能有显著影响。表示越大，计算就越昂贵。因此，我们建议您根据您的用例明智地选择数字表示。例如，在这里，euint64 足以处理代币余额。

{% endhint %}

## 创建我们的出价函数

现在让我们创建我们的出价函数，用户将通过该函数转移机密金额并将其发送到拍卖智能合约。
由于我们希望出价保持私密，用户必须首先在本地加密他们的出价金额。然后，这个加密的值将用于从我们设置为支付方式的 `ConfidentialFungibleToken` 代币中安全地转移资金。
我们可以这样创建我们的函数：

```solidity
function bid(
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) public onlyDuringAuction nonReentrant {
    // 从用户处获取并验证金额
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);

    // ...
```

在这里，我们接受两个参数：

- 加密金额：用户的出价金额，使用 FHE 加密。
- 输入证明：确保加密数据有效性的零知识证明。

我们可以使用我们的帮助函数 `FHE.fromExternal()` 来验证这些参数，它为我们提供了对加密金额的引用。

然后，我们需要将机密代币转移到合约中。

```solidity
euint64 balanceBefore = confidentialFungibleToken.confidentialBalanceOf(address(this));
confidentialFungibleToken.confidentialTransferFrom(msg.sender, address(this), amount);
euint64 balanceAfter = confidentialFungibleToken.confidentialBalanceOf(address(this));
euint64 sentBalance = FHE.sub(balanceAfter, balanceBefore);
```

请注意，在这里，我们不使用用户提供的金额作为信任来源。事实上，如果用户资金不足，在调用 `confidentialTransferFrom()` 时，**交易不会被回滚，而是会静默地转移一个 `0` 值**。这种设计选择可以防止潜在的泄露，因为回滚的交易可能会无意中泄露有关数据的一些信息。

> 注意：要深入了解 FHE 的工作原理，链上完成的每个 FHE 操作都会发出一个事件，用于构建一个计算图。然后，Zama FHEVM 会执行此图。因此，FHE 操作不是直接在智能合约端完成的，而是遵循它生成的源图。

支付完成后，我们需要更新用户的出价余额。请注意，如果用户愿意，他可以增加之前的出价：

```solidity
euint64 previousBid = bids[msg.sender];
if (FHE.isInitialized(previousBid)) {  // 用户增加他的出价
    euint64 newBid = FHE.add(previousBid, sentBalance);
    bids[msg.sender] = newBid;
} else {
    // 用户的第一次出价
    bids[msg.sender] = sentBalance;
}
```

最后，我们可以检查是否需要更新加密的获胜者：

```solidity
// 将用户的总价值与最高出价进行比较
euint64 currentBid = bids[msg.sender];
FHE.allowThis(currentBid);
FHE.allow(currentBid, msg.sender);

if (FHE.isInitialized(highestBid)) {
    ebool isNewWinner = FHE.lt(highestBid, currentBid);
    highestBid = FHE.select(isNewWinner, currentBid, highestBid);
    winningAddress = FHE.select(isNewWinner, FHE.asEaddress(msg.sender), winningAddress);
} else {
    highestBid = currentBid;
    winningAddress = FHE.asEaddress(msg.sender);
}
FHE.allowThis(highestBid);
FHE.allowThis(winningAddress);
```

如您所见，我们在这里使用了一些 FHE 函数。让我们来谈谈 `FHE.allow()` 和 `FHE.allowThis()`。每个加密的值都有关于谁可以读取此值的限制。为了能够访问此值甚至对其进行一些计算，我们需要明确请求访问权限。这就是我们需要明确请求访问权限的原因。例如，在这里，我们希望合约和用户能够访问出价。然而，只有合约才能访问将在拍卖结束时揭晓的最高出价和获胜者地址。

我们想提到的另一点是 `FHE.select()` 函数。如前所述，在使用 FHE 时，我们不希望交易被回滚。相反，在构建我们的 FHE 操作图时，我们希望根据加密的值创建两条路径。这就是我们使用**分支**的原因，它允许我们定义我们想要的过程类型。例如，在这里，如果用户的出价高于当前出价，我们将更改金额和地址。然而，如果不是这种情况，我们将保留旧的。这种分支方法特别有用，因为在链上您无法直接访问加密数据，但您仍然希望根据它们调整您的合约逻辑。

好了，我们的出价函数似乎已经准备好了。这是我们到目前为止看到的完整代码：

```solidity
function bid(externalEuint64 encryptedAmount, bytes calldata inputProof) public onlyDuringAuction nonReentrant {
    // 从用户处获取并验证金额
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);

    // 将机密代币作为支付转移
    euint64 balanceBefore = confidentialFungibleToken.confidentialBalanceOf(address(this));
    FHE.allowTransient(amount, address(confidentialFungibleToken));
    confidentialFungibleToken.confidentialTransferFrom(msg.sender, address(this), amount);
    euint64 balanceAfter = confidentialFungibleToken.confidentialBalanceOf(address(this));
    euint64 sentBalance = FHE.sub(balanceAfter, balanceBefore);

    // 需要更新出价余额
    euint64 previousBid = bids[msg.sender];
    if (FHE.isInitialized(previousBid)) {
        // 用户增加他的出价
        euint64 newBid = FHE.add(previousBid, sentBalance);
        bids[msg.sender] = newBid;
    } else {
        // 用户的第一次出价
        bids[msg.sender] = sentBalance;
    }

    // 将用户的总价值与最高出价进行比较
    euint64 currentBid = bids[msg.sender];
    FHE.allowThis(currentBid);
    FHE.allow(currentBid, msg.sender);

    if (FHE.isInitialized(highestBid)) {
        ebool isNewWinner = FHE.lt(highestBid, currentBid);
        highestBid = FHE.select(isNewWinner, currentBid, highestBid);
        winningAddress = FHE.select(isNewWinner, FHE.asEaddress(msg.sender), winningAddress);
    } else {
        highestBid = currentBid;
        winningAddress = FHE.asEaddress(msg.sender);
    }
    FHE.allowThis(highestBid);
    FHE.allowThis(winningAddress);
}
```

## 拍卖解决阶段

一旦所有参与者都出价，就该进入解决阶段了，我们需要揭晓获胜者的地址。首先，我们需要解密获胜者的地址，因为它目前是加密的。为此，我们可以使用 Zama 提供的 `DecryptionOracle`。该预言机将负责安全地处理加密值的解密，并通过回调返回结果。为了实现这一点，让我们创建一个将调用 `DecryptionOracle` 的函数：

```solidity
function decryptWinningAddress() public onlyAfterEnd {
  bytes32[] memory cts = new bytes32[](1);
  cts[0] = FHE.toBytes32(winningAddress);
  _latestRequestId = FHE.requestDecryption(cts, this.resolveAuctionCallback.selector);
}
```

在这里，我们请求解密 `winningAddress` 的单个参数。但是，您可以通过增加 `cts` 数组并添加其他参数来请求多个参数。

另请注意，在调用 `FHE.requestDecryption()` 时，我们在参数中传递了一个选择器。该选择器将是预言机回调的选择器。

另请注意，我们已将此函数限制为仅在拍卖结束后才能调用。在拍卖仍在进行时，我们绝不能调用它，否则会泄露一些信息。

我们现在可以编写我们的 `resolveAuctionCallback` 回调函数：

```solidity
function resolveAuctionCallback(uint256 requestId, bytes memory cleartexts, bytes memory decryptionProof) public {
  require(requestId == _latestRequestId, "Invalid requestId");
  FHE.checkSignatures(requestId, cleartexts, decryptionProof);

  (address resultWinnerAddress) = abi.decode(cleartexts, (address));
  winnerAddress = resultWinnerAddress;
}
```

`cleartexts` 是对应于所有请求的解密值的 ABI 编码的字节数组，在这种情况下是 `abi.encode(winningAddress)`。

为确保它是我们正在等待的预期数据，我们需要验证 `requestId` 参数和签名（包含在 `decryptionProof` 参数中），它们验证了完成的计算逻辑。验证后，我们可以更新获胜者的地址。

## 领取奖励和退款

好了，一旦获胜者揭晓，我们现在可以允许获胜者领取他的奖励，而其他人则可以获得退款。

```solidity
function winnerClaimPrize() public onlyAfterWinnerRevealed {
  require(winnerAddress == msg.sender, "Only winner can claim item");
  require(!isNftClaimed, "NFT has already been claimed");
  isNftClaimed = true;

  // 重置出价
  bids[msg.sender] = FHE.asEuint64(0);
  FHE.allowThis(bids[msg.sender]);
  FHE.allow(bids[msg.sender], msg.sender);

  // 将最高出价转移给受益人
  FHE.allowTransient(highestBid, address(confidentialFungibleToken));
  confidentialFungibleToken.confidentialTransfer(beneficiary, highestBid);

  // 将 NFT 发送给获胜者
  nftContract.safeTransferFrom(address(this), msg.sender, tokenId);
}
```

```solidity
function withdraw(address bidder) public onlyAfterWinnerRevealed {
  if (bidder == winnerAddress) revert TooLateError(auctionEndTime);

  // 获取用户的出价
  euint64 amount = bids[bidder];
  FHE.allowTransient(amount, address(confidentialFungibleToken));

  // 重置用户的出价
  euint64 newBid = FHE.asEuint64(0);
  bids[bidder] = newBid;
  FHE.allowThis(newBid);
  FHE.allow(newBid, bidder);

  // 用他的出价金额退款给用户
  confidentialFungibleToken.confidentialTransfer(bidder, amount);
}
```

# 结论

在本指南中，我们介绍了如何使用链上全同态加密（FHE）构建一个密封投标的 NFT 拍卖。

我们演示了如何使用 FHE 设计一个私密和公平的拍卖机制，保持所有出价加密，并且只在必要时才揭示信息。

现在轮到您了。请随时在此代码的基础上进行构建，用更复杂的逻辑对其进行扩展，或创建您自己的由 FHE 驱动的去中心化应用程序。
