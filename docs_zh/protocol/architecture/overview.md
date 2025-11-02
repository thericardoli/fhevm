# 区块链上的 FHE

本节深入解释了 Zama 机密区块链协议（Zama 协议），并演示了如何使用全同态加密（FHE）将加密计算引入智能合约。

FHEVM 是 Zama 协议的核心技术。它由以下关键组件组成。

<figure><img src="../.gitbook/assets/FHEVM.png" alt=""><figcaption></figcaption></figure>

- [**FHEVM Solidity 库**](library.md)：使开发人员能够使用加密数据类型和操作，以纯 Solidity 编写机密智能合约。
- [**主机合约**](hostchain.md)：部署在 EVM 兼容区块链上的受信任的链上合约。它们管理访问控制并触发链下加密计算。
- [**协处理器**](coprocessor.md) – 验证加密输入、运行 FHE 计算并提交结果的分散式服务。
- [**网关**](gateway.md)**–** 协议的中央协调器。它验证加密输入、管理访问控制列表（ACL）、跨链桥接密文，并协调协处理器和 KMS。
- [**密钥管理服务 (KMS)**](kms.md) – 一个阈值 MPC 网络，用于生成和轮换 FHE 密钥，并处理安全、可验证的解密。
- [**中继器和预言机**](relayer_oracle.md) – 一个轻量级的链下服务，通过转发加密或解密请求来帮助用户与网关交互。
