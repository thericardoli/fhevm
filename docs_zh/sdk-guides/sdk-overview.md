# 中继器 SDK

**欢迎来到中继器 SDK 文档。**

本节概述了 Zama 的 FHEVM 中继器 JavaScript SDK 的主要功能。
该 SDK 允许您与 FHEVM 智能合约交互，而无需直接处理[网关链](https://docs.zama.ai/protocol/protocol/overview/gateway)。

使用中继器，FHEVM 客户端只需要在 FHEVM 主机链上拥有一个钱包。与网关链的所有交互都通过 HTTP 调用 Zama 的中继器来处理，中继器在网关链上为其支付费用。

## 接下来去哪里

如果您是 Zama 协议的新手，请从 [Litepaper](https://docs.zama.ai/protocol/zama-protocol-litepaper) 或[协议概述](https://docs.zama.ai/protocol)开始，了解基础知识。

否则：

🟨 前往[**设置指南**](initialization.md)了解如何为您的项目配置中继器 SDK。

🟨 前往[**输入注册**](input.md)了解如何为您的智能合约注册新的加密输入。

🟨 前往[**用户解密**](user-decryption.md)，使用户能够在通过访问控制列表（ACL）授予权限后使用自己的密钥解密数据。

🟨 前往[**公共解密**](public-decryption.md)了解如何解密可公开访问的输出，无论是通过 HTTP 还是链上预言机。

🟨 前往 [**Solidity ACL 指南**](https://docs.zama.ai/protocol/solidity-guides/smart-contract/acl)获取有关访问控制的更详细说明。

## 帮助中心

提出技术问题并与社区讨论。

- [社区论坛](https://community.zama.ai/c/zama-protocol/15)
- [Discord 频道](https://discord.com/invite/zama)
