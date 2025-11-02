此合约是使用 FHEVM 构建的机密密封投标拍卖的示例。请参阅[教程](sealed-bid-auction-tutorial.md)以逐步了解其实现方式。

{% hint style="info" %}
为正确运行此示例，请确保将文件放置在以下目录中：

- `.sol` 文件 → `<your-project-root-dir>/contracts/`
- `.ts` 文件 → `<your-project-root-dir>/test/`

这能确保 Hardhat 能够按预期编译和测试您的合约。
{% endhint %}

{% tabs %}

{% tab title="BlindAuction.sol" %}

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import {FHE, externalEuint64, euint64, eaddress, ebool} from "@fhevm/solidity/lib/FHE.sol";
import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";
import {Ownable2Step, Ownable} from "@openzeppelin/contracts/access/Ownable2Step.sol";
import {IERC20Errors} from "@openzeppelin/contracts/interfaces/draft-IERC6093.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

import {ConfidentialFungibleToken} from "@openzeppelin/confidential-contracts/token/ConfidentialFungibleToken.sol";

contract BlindAuction is SepoliaConfig, ReentrancyGuard {
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

    /// @notice 加密的拍卖信息
    euint64 private highestBid;
    eaddress private winningAddress;

    /// @notice 拍卖结束时定义的获胜者地址
    address public winnerAddress;

    /// @notice 指示拍卖的 NFT 是否已被领取
    bool public isNftClaimed;

    /// @notice 用于解密的请求 ID
    uint256 internal _decryptionRequestId;

    /// @notice 从出价者到其出价金额的映射
    mapping(address account => euint64 bidAmount) private bids;

    // ========== 错误 ==========

    /// @notice 当函数调用过早时抛出的错误
    /// @dev 包括可以调用函数的时间
    error TooEarlyError(uint256 time);

    /// @notice 当函数调用过晚时抛出的错误
    /// @dev 包括函数不能再调用的时间
    error TooLateError(uint256 time);

    /// @notice 尝试需要解决获胜者的操作时抛出
    /// @dev 表示获胜者尚未解密
    error WinnerNotYetRevealed();

    // ========== 修饰符 ==========

    /// @notice 确保函数在拍卖结束前调用的修饰符。
    /// @dev 如果在拍卖结束时间后调用，则回滚。
    modifier onlyDuringAuction() {
        if (block.timestamp < auctionStartTime) revert TooEarlyError(auctionStartTime);
        if (block.timestamp >= auctionEndTime) revert TooLateError(auctionEndTime);
        _;
    }

    /// @notice 确保函数在拍卖结束后调用的修饰符。
    /// @dev 如果在拍卖结束时间前调用，则回滚。
    modifier onlyAfterEnd() {
        if (block.timestamp < auctionEndTime) revert TooEarlyError(auctionEndTime);
        _;
    }

    /// @notice 确保函数在获胜者揭晓后调用的修饰符。
    /// @dev 如果在获胜者揭晓前调用，则回滚。
    modifier onlyAfterWinnerRevealed() {
        if (winnerAddress == address(0)) revert WinnerNotYetRevealed();
        _;
    }

    // ========== 视图 ==========

    function getEncryptedBid(address account) external view returns (euint64) {
        return bids[account];
    }

    /// @notice 拍卖结束后获取获胜者地址
    /// @dev 只能在获胜者地址解密后调用
    /// @return winnerAddress 解密的获胜者地址
    function getWinnerAddress() external view returns (address) {
        require(winnerAddress != address(0), "Winning address has not been decided yet");
        return winnerAddress;
    }

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

    /// @notice 启动获胜者地址的解密
    /// @dev 只能在拍卖结束后调用
    function decryptWinningAddress() public onlyAfterEnd {
        bytes32[] memory cts = new bytes32[](1);
        cts[0] = FHE.toBytes32(winningAddress);
        _decryptionRequestId = FHE.requestDecryption(cts, this.resolveAuctionCallback.selector);
    }

    /// @notice 领取 NFT 奖品。
    /// @dev 只有获胜者可以在拍卖结束后调用此函数。
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

    /// @notice 从拍卖中撤回出价
    /// @dev 只能在拍卖结束后且由非获胜出价者调用
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

    // ========== 预言机回调 ==========

    /// @notice 设置解密的获胜者地址的回调函数
    /// @dev 只能由网关调用
    /// @param requestId 由预言机创建的请求 ID。
    /// @param resultWinnerAddress 解密的获胜者地址。
    /// @param signatures 用于验证解密数据的签名。
    function resolveAuctionCallback(uint256 requestId, bytes memory cleartexts, bytes memory decryptionProof) public {
        require(requestId == _decryptionRequestId, "Invalid requestId");
        FHE.checkSignatures(requestId, cleartexts, decryptionProof);

        (address resultWinnerAddress) = abi.decode(cleartexts, (address));
        winnerAddress = resultWinnerAddress;
    }
}
```

{% endtab %}

{% tab title="BlindAuction.ts" %}

```ts
import { FhevmType } from "@fhevm/hardhat-plugin";
import { expect } from "chai";
import { ethers } from "hardhat";
import { time } from "@nomicfoundation/hardhat-network-helpers";
import * as hre from "hardhat";

type Signers = {
  owner: HardhatEthersSigner;
  alice: HardhatEthersSigner;
  bob: HardhatEthersSigner;
};

import { deployBlindAuctionFixture } from "./BlindAuction.fixture";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";

describe("BlindAuction", function () {
  before(async function () {
    if (!hre.fhevm.isMock) {
      throw new Error(`此 hardhat 测试套件无法在 Sepolia 测试网上运行`);
    }
    this.signers = {} as Signers;

    const signers = await ethers.getSigners();
    this.signers.owner = signers[0];
    this.signers.alice = signers[1];
    this.signers.bob = signers[2];
  });

  beforeEach(async function () {
    const deployment = await deployBlindAuctionFixture(this.signers.owner);

    this.USDCc = deployment.USDCc;
    this.prizeItem = deployment.prizeItem;
    this.blindAuction = deployment.blindAuction;

    this.USDCcAddress = deployment.USDCc_address;
    this.prizeItemAddress = deployment.prizeItem_address;
    this.blindAuctionAddress = deployment.blindAuction_address;

    this.getUSDCcBalance = async (signer: HardhatEthersSigner) => {
      const encryptedBalance = await this.USDCc.confidentialBalanceOf(signer.address);
      return await hre.fhevm.userDecryptEuint(FhevmType.euint64, encryptedBalance, this.USDCcAddress, signer);
    };

    this.encryptBid = async (targetContract: string, userAddress: string, amount: number) => {
      const bidInput = hre.fhevm.createEncryptedInput(targetContract, userAddress);
      bidInput.add64(amount);
      return await bidInput.encrypt();
    };

    this.approve = async (signer: HardhatEthersSigner) => {
      // 批准发送资金
      const approveTx = await this.USDCc.connect(signer)["setOperator(address, uint48)"](
        this.blindAuctionAddress,
        Math.floor(Date.now() / 1000) + 60 * 60,
      );
      await approveTx.wait();
    };

    this.bid = async (signer: HardhatEthersSigner, amount: number) => {
      const encryptedBid = await this.encryptBid(this.blindAuctionAddress, signer.address, amount);
      const bidTx = await this.blindAuction.connect(signer).bid(encryptedBid.handles[0], encryptedBid.inputProof);
      await bidTx.wait();
    };

    this.mintUSDc = async (signer: HardhatEthersSigner, amount: number) => {
      // 使用更简单的 mint 函数，不需要 FHE 加密
      const mintTx = await this.USDCc.mint(signer.address, amount);
      await mintTx.wait();
    };
  });

  it("应铸造机密 USDC", async function () {
    const aliceSigner = this.signers.alice;
    const aliceAddress = aliceSigner.address;

    // 检查初始余额
    const initialEncryptedBalance = await this.USDCc.confidentialBalanceOf(aliceAddress);
    console.log("初始加密余额：", initialEncryptedBalance);

    // 铸造一些机密 USDC
    await this.mintUSDc(aliceSigner, 1_000_000);

    // 检查铸造后的余额
    const finalEncryptedBalance = await this.USDCc.confidentialBalanceOf(aliceAddress);
    console.log("最终加密余额：", finalEncryptedBalance);

    // 余额应该不同（不为零）
    expect(finalEncryptedBalance).to.not.equal(initialEncryptedBalance);
  });

  it("应放置一个加密的出价", async function () {
    const aliceSigner = this.signers.alice;
    const aliceAddress = aliceSigner.address;

    // 铸造一些机密 USDC
    await this.mintUSDc(aliceSigner, 1_000_000);

    // 出价金额
    const bidAmount = 10_000;

    await this.approve(aliceSigner);
    await this.bid(aliceSigner, bidAmount);

    // 检查支付转移
    const aliceEncryptedBalance = await this.USDCc.confidentialBalanceOf(aliceAddress);
    const aliceClearBalance = await hre.fhevm.userDecryptEuint(
      FhevmType.euint64,
      aliceEncryptedBalance,
      this.USDCcAddress,
      aliceSigner,
    );
    expect(aliceClearBalance).to.equal(1_000_000 - bidAmount);

    // 检查出价
    const aliceEncryptedBid = await this.blindAuction.getEncryptedBid(aliceAddress);
    const aliceClearBid = await hre.fhevm.userDecryptEuint(
      FhevmType.euint64,
      aliceEncryptedBid,
      this.blindAuctionAddress,
      aliceSigner,
    );
    expect(aliceClearBid).to.equal(bidAmount);
  });

  it("bob 应赢得拍卖", async function () {
    const aliceSigner = this.signers.alice;
    const bobSigner = this.signers.bob;
    const beneficiary = this.signers.owner;

    // 铸造一些机密 USDC
    await this.mintUSDc(aliceSigner, 1_000_000);
    await this.mintUSDc(bobSigner, 1_000_000);

    // Alice 出价
    await this.approve(aliceSigner);
    await this.bid(aliceSigner, 10_000);

    // Bob 出价
    await this.approve(bobSigner);
    await this.bid(bobSigner, 15_000);

    // 等待拍卖结束
    await time.increase(3600);

    await this.blindAuction.decryptWinningAddress();
    await hre.fhevm.awaitDecryptionOracle();

    // 验证获胜者
    expect(await this.blindAuction.getWinnerAddress()).to.be.equal(bobSigner.address);

    // Bob 不能提取任何钱
    await expect(this.blindAuction.withdraw(bobSigner.address)).to.be.reverted;

    // 领取的 NFT 物品
    expect(await this.prizeItem.ownerOf(await this.blindAuction.tokenId())).to.be.equal(this.blindAuctionAddress);
    await this.blindAuction.connect(bobSigner).winnerClaimPrize();
    expect(await this.prizeItem.ownerOf(await this.blindAuction.tokenId())).to.be.equal(bobSigner.address);

    // 退款用户
    const aliceBalanceBefore = await this.getUSDCcBalance(aliceSigner);
    await this.blindAuction.withdraw(aliceSigner.address);
    const aliceBalanceAfter = await this.getUSDCcBalance(aliceSigner);
    expect(aliceBalanceAfter).to.be.equal(aliceBalanceBefore + 10_000n);

    // Bob 不能提取任何钱
    await expect(this.blindAuction.withdraw(bobSigner.address)).to.be.reverted;

    // 检查受益人余额
    const beneficiaryBalance = await this.getUSDCcBalance(beneficiary);
    expect(beneficiaryBalance).to.be.equal(15_000);
  });
});
```

{% endtab %}

{% tab title="BlindAuction.fixture.ts" %}

```ts
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { ethers } from "hardhat";

import type { ConfidentialTokenExample, PrizeItem, BlindAuction } from "../../types";
import type { ConfidentialTokenExample__factory, PrizeItem__factory, BlindAuction__factory } from "../../types";

export async function deployBlindAuctionFixture(owner: HardhatEthersSigner) {
  const [deployer] = await ethers.getSigners();

  // 创建机密 ERC20
  const USDCcFactory = (await ethers.getContractFactory(
    "ConfidentialTokenExample",
  )) as ConfidentialTokenExample__factory;
  const USDCc = (await USDCcFactory.deploy(0, "USDCc", "USDCc", "")) as ConfidentialTokenExample;
  const USDCc_address = await USDCc.getAddress();

  // 创建 NFT 奖品
  const PrizeItemFactory = (await ethers.getContractFactory("PrizeItem")) as PrizeItem__factory;
  const prizeItem = (await PrizeItemFactory.deploy()) as PrizeItem;
  const prizeItem_address = await prizeItem.getAddress();

  // 创建第一个奖品
  const mintTx = await prizeItem.newItem();
  await mintTx.wait();

  const nonce = await deployer.getNonce();

  // 预计算 BlindAuction 合约的地址
  const precomputedBlindAuctionAddress = ethers.getCreateAddress({
    from: deployer.address,
    nonce: nonce + 1,
  });

  // 批准将其发送到拍卖
  const approveTx = await prizeItem.approve(precomputedBlindAuctionAddress, 0);
  await approveTx.wait();

  // 默认情况下，合约使用第一个签名者/账户进行部署
  const BlindAuctionFactory = (await ethers.getContractFactory("BlindAuction")) as BlindAuction__factory;
  const blindAuction = (await BlindAuctionFactory.deploy(
    prizeItem_address,
    USDCc_address,
    0,
    Math.floor(Date.now() / 1000),
    Math.floor(Date.now() / 1000) + 60 * 60,
  )) as BlindAuction;
  const blindAuction_address = await blindAuction.getAddress();

  return { USDCc, USDCc_address, prizeItem, prizeItem_address, blindAuction, blindAuction_address };
}
```

{% endtab %}

{% endtabs %}
