# 用户解密

本文档解释如何执行用户解密。
当您希望用户访问其私有数据而不将其暴露给区块链时，需要用户解密。

FHEVM 中的用户解密使得能够在新公钥下安全共享或重用加密数据，而不暴露明文。

此功能对于加密数据必须在合约、dApp 或用户之间传输同时保持其机密性的场景至关重要。

## 何时使用用户解密

用户解密特别适用于**允许个人用户安全访问和解密其私有数据**，例如余额或计数器，同时保持数据机密性。

## 概述

用户解密过程包括从区块链检索密文并在客户端执行用户解密。换句话说，我们获取由 KMS 加密的数据，解密它并使用用户的私钥加密它，因此只有他可以访问该信息。

这确保数据在区块链的 FHE 密钥下保持加密，但可以通过在用户的 NaCl 公钥下重新加密来安全地与用户共享。

用户解密由**中继器**和**密钥管理系统（KMS）**促进。工作流程包括以下内容：

1. 使用合约的视图函数从区块链检索密文。
2. 使用用户的公钥在客户端重新加密密文，确保只有用户可以解密它。

## 步骤 1：检索密文

要检索需要解密的密文，您可以在智能合约中实现视图函数。以下是一个示例实现：

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
为了使用户能够用户解密（也称为重新加密）密文值，需要使用持有密文的 solidity 合约中的 `FHE.allow(ciphertext, address)` 函数正确设置访问控制（ACL）。
有关该主题的更多详细信息，请参阅 [ACL 文档](../solidity-guides/acl/README.md)。
{% endhint %}

## 步骤 2：解密密文

使用该密文句柄，用户解密在客户端使用 `@zama-fhe/relayer-sdk` 库执行。
用户需要在此之前创建一个实例对象（有关更多上下文，请参阅[中继器-sdk 设置页面](./initialization.md)）。

```ts
// instance: [`FhevmInstance`] 来自 `zama-fhe/relayer-sdk`
// signer: [`Signer`] 来自 ethers（可以是 [`Wallet`]）
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
const durationDays = "10"; // 字符串以保持一致性
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
