# 区块链上的 FHE

本节深入解释了 Zama 机密区块链协议（Zama 协议），并展示了它如何使用全同态加密（FHE）为智能合约带来加密计算。

FHEVM 是支撑 Zama 协议的核心技术。它由以下关键组件组成。

<figure><img src="../.gitbook/assets/FHEVM.png" alt=""><figcaption></figcaption></figure>

- [**FHEVM Solidity 库**](library.md)：使开发人员能够使用加密数据类型和操作在纯 Solidity 中编写机密智能合约。
- [**主机合约**](hostchain.md)：部署在兼容 EVM 的区块链上的可信链上合约。它们管理访问控制并触发链外加密计算。
- [**协处理器**](coprocessor.md)：验证加密输入、运行 FHE 计算和提交结果的去中心化服务。
- [**网关**](gateway.md)：协议的中央协调器。它验证加密输入，管理访问控制列表（ACL），跨链桥接密文，并协调协处理器和 KMS。
- [**密钥管理服务（KMS）**](kms.md)：生成和轮换 FHE 密钥并处理安全、可验证解密的门限 MPC 网络。
- [**中继器和预言机**](relayer_oracle.md)：一种轻量级的链外服务，通过\
  转发加密或解密请求帮助用户与网关交互。
