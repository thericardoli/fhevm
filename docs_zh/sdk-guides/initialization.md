# 设置

使用 `@zama-fhe/relayer-sdk` 需要一个设置阶段。
这包括实例化 `FhevmInstance`。
此对象包含使用中继器与 FHEVM 交互所需的所有配置和方法。
可以使用以下代码片段创建它：

```ts
import { createInstance } from "@zama-fhe/relayer-sdk";

const instance = await createInstance({
  // ACL_CONTRACT_ADDRESS (FHEVM 主机链)
  aclContractAddress: "0x687820221192C5B662b25367F70076A37bc79b6c",
  // KMS_VERIFIER_CONTRACT_ADDRESS (FHEVM 主机链)
  kmsContractAddress: "0x1364cBBf2cDF5032C47d8226a6f6FBD2AFCDacAC",
  // INPUT_VERIFIER_CONTRACT_ADDRESS (FHEVM 主机链)
  inputVerifierContractAddress: "0xbc91f3daD1A5F19F8390c400196e58073B6a0BC4",
  // DECRYPTION_ADDRESS (网关链)
  verifyingContractAddressDecryption: "0xb6E160B1ff80D67Bfe90A85eE06Ce0A2613607D1",
  // INPUT_VERIFICATION_ADDRESS (网关链)
  verifyingContractAddressInputVerification: "0x7048C39f048125eDa9d678AEbaDfB22F7900a29F",
  // FHEVM 主机链 ID
  chainId: 11155111,
  // 网关链 ID
  gatewayChainId: 55815,
  // 主机链的可选 RPC 提供商
  network: "https://eth-sepolia.public.blastapi.io",
  // 中继器 URL
  relayerUrl: "https://relayer.testnet.zama.cloud",
});
```

或者更简单的：

```ts
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk";

const instance = await createInstance(SepoliaConfig);
```

有关 Zama 维护的 Sepolia 的 FHEVM 和相关中继器配置的信息可以在 `SepoliaConfig` 对象或[合约地址页面](https://docs.zama.ai/protocol/solidity-guides/smart-contract/configure/contract_addresses)中找到。
`gatewayChainId` 是 `55815`。
`chainId` 是 FHEVM 链的链 ID，因此对于 Sepolia，它是 `11155111`。

{% hint style="info" %}
有关中继器在整体架构中的作用的更多信息，请参阅[架构文档中的中继器页面](https://docs.zama.ai/protocol/protocol/overview/relayer_oracle)。
{% endhint %}
