# 中继器 SDK

**欢迎阅读中继器 SDK 文档。**

本节概述了 Zama FHEVM 中继器 JavaScript SDK 的主要功能。
该 SDK 使您无需直接与[网关链](https://docs.zama.ai/protocol/protocol/overview/gateway)交互即可与 FHEVM 智能合约进行交互。

使用中继器，FHEVM 客户端只需要 FHEVM 主机链上的一个钱包。与网关链的所有交互都通过对 Zama 中继器的 HTTP 调用来处理，Zama 中继器在网关链上支付费用。

## 下一步

如果您是 Zama 协议的新手，请从[白皮书](https://docs.zama.ai/protocol/zama-protocol-litepaper)或[协议概述](https://docs.zama.ai/protocol)开始了解基础知识。

否则：

🟨 前往[**设置指南**](initialization.md)了解如何为您的项目配置中继器 SDK。

🟨 前往[**输入注册**](input.md)查看如何为您的智能合约注册新的加密输入。

🟨 前往[**用户解密**](user-decryption.md)以使用户能够在通过访问控制列表（ACL）授予权限后使用自己的密钥解密数据。

🟨 前往[**公共解密**](public-decryption.md)了解如何解密可通过 HTTP 或链上预言机公开访问的输出。

🟨 前往[**Solidity ACL 指南**](https://docs.zama.ai/protocol/solidity-guides/smart-contract/acl)获取有关访问控制的更详细说明。

## 帮助中心

提出技术问题并与社区讨论。

- [社区论坛](https://community.zama.ai/c/zama-protocol/15)
- [Discord 频道](https://discord.com/invite/zama)
