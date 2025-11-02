# 用户解密

本文档介绍了如何执行用户解密。
当您希望用户访问其私有数据而不将其暴露给区块链时，就需要进行用户解密。

FHEVM 中的用户解密能够在不暴露明文的情况下，使用新的公钥安全地共享或重用加密数据。

此功能对于需要在合约、dApp 或用户之间传输加密数据同时保持其机密性的场景至关重要。

## 何时使用用户解密

用户解密对于**允许单个用户安全地访问和解密其私有数据**（例如余额或计数器），同时保持数据机密性特别有用。

## 概述

用户解密过程涉及从区块链检索密文并在客户端执行用户解密。换句话说，我们获取由 KMS 加密的数据，对其进行解密并使用用户的私钥对其进行加密，这样只有他才能访问信息。

这确保了数据在区块链的 FHE 密钥下保持加密，但可以通过使用用户的 NaCl 公钥对其进行重新加密来安全地与用户共享。

用户解密由**中继器**和**密钥管理系统 (KMS)** 促成。工作流程包括以下内容：

1. 使用合约的视图函数从区块链检索密文。
2. 在客户端使用用户的公钥重新加密密文，确保只有用户才能解密。

## 第 1 步：检索密文

要检索需要解密的密文，您可以在智能合约中实现一个视图函数。以下是一个示例实现：

```solidity
import "@fhevm/solidity/lib/FHE.sol";

contract ConfidentialERC20 {
  ...
  function balanceOf(account address) public view returns (euint64) {
    return balances[msg.sender];
  }
  ...
}
```

在这里，`balanceOf` 允许检索存储在区块链上的用户加密余额句柄。
这样做将返回密文句柄，即底层密文的标识符。

{% hint style="warning" %}
为了让用户能够用户解密（也称为重新加密）密文值，需要使用持有密文的 solidity 合约中的 `FHE.allow(ciphertext, address)` 函数正确设置访问控制 (ACL)。
有关该主题的更多详细信息，请参阅 [ACL 文档](../solidity-guides/acl/README.md)。
{% endhint %}

## 第 2 步：解密密文

使用该密文句柄，可以使用 `@zama-fhe/relayer-sdk` 库在客户端执行用户解密。
在此之前，用户需要创建一个实例对象（有关更多上下文，请参阅[中继器-sdk 设置页面](./initialization.md)）。

```ts
// instance: 来自 `zama-fhe/relayer-sdk` 的 [`FhevmInstance`]
// signer: 来自 ethers 的 [`Signer`]（可以是 [`Wallet`]）
// ciphertextHandle: [`string`]
// contractAddress: [`string`]

const keypair = instance.generateKeypair();
const handleContractPairs = [
  {
    handle: ciphertextHandle,
    contractAddress: contractAddress,
  },
];
const startTimeStamp = Math.floor(Date.now() / 1000).toString();
const durationDays = "10"; // 为保持一致性使用字符串
const contractAddresses = [contractAddress];

const eip712 = instance.createEIP712(keypair.publicKey, contractAddresses, startTimeStamp, durationDays);

const signature = await signer.signTypedData(
  eip712.domain,
  {
    UserDecryptRequestVerification: eip712.types.UserDecryptRequestVerification,
  },
  eip712.message,
);

const result = await instance.userDecrypt(
  handleContractPairs,
  keypair.privateKey,
  keypair.publicKey,
  signature.replace("0x", ""),
  contractAddresses,
  signer.address,
  startTimeStamp,
  durationDays,
);

const decryptedValue = result[ciphertextHandle];
```
