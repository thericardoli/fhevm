# 重组处理

本页面提供了有关在使用 FHEVM 时如何在以太坊上处理重组风险的详细说明。

由于 ACL 事件在被包含在一个区块后立即从 FHEVM 主机链传播到[网关](https://docs.zama.ai/protocol/protocol/overview/gateway)，因此当加密信息至关重要时，dApp 开发人员必须特别小心。例如，如果一个加密句柄隐藏了一个持有大量资金的比特币钱包的私钥，我们需要确保这些信息不会因为 FHEVM 主机链上的重组而无意中泄露给错误的人。因此，dApp 开发人员有责任通过在请求和 ACL 调用之间实现一个带有时间锁的两步 ACL 授权过程来防止此类情况的发生。

## 简单示例：处理以太坊上的重组风险

在以太坊上，在最坏的情况下，重组的深度可能达到 95 个插槽，因此等待超过 95 个区块应该可以确保先前发送的交易已经最终确定——除非超过 1/3 的节点是恶意的并且愿意损失他们的权益，这是极不可能的。

❌ **不要编写此合约：**

```solidity
contract PrivateKeySale {
  euint256 privateKey;
  bool isAlreadyBought = false;

  constructor(externalEuint256 _privateKey, bytes inputProof) {
    privateKey = FHE.fromExternal(_privateKey, inputProof);
    FHE.allowThis(privateKey);
  }

  function buyPrivateKey() external payable {
    require(msg.value == 1 ether, "Must pay 1 ETH");
    require(!isBought, "Private key already bought");
    isBought = true;
    FHE.allow(encryptedPrivateKey, msg.sender);
  }
}
```

由于 `privateKey` 加密变量包含关键信息，如果发生重组，我们不希望错误地免费泄露它。在前面的示例中可能会发生这种情况，因为我们在处理销售的同一笔交易中立即向买方授予授权。

✅ **我们建议改为编写类似这样的代码：**

```solidity
contract PrivateKeySale {
  euint256 privateKey;
  bool isAlreadyBought = false;
  uint256 blockWhenBought = 0;
  address buyer;

  constructor(externalEuint256 _privateKey, bytes inputProof) {
    privateKey = FHE.fromExternal(_privateKey, inputProof);
    FHE.allowThis(privateKey);
  }

  function buyPrivateKey() external payable {
    require(msg.value == 1 ether, "Must pay 1 ETH");
    require(!isBought, "Private key already bought");
    isBought = true;
    blockWhenBought = block.number;
    buyer = msg.sender;
  }

  function requestACL() external {
    require(isBought, "Private key has not been bought yet");
    require(block.number > blockWhenBought + 95, "Too early to request ACL, risk of reorg");
    FHE.allow(privateKey, buyer);
  }
}
```

这种方法确保了在购买私钥的交易和授权买方解密的交易之间至少经过了 96 个区块。

{% hint style="info" %}
这种类型的合约通过在用户可以解密数据之前添加一个时间锁来降低用户体验，因此应该谨慎使用：仅当泄露的信息可能至关重要且价值很高时。
{% endhint %}
