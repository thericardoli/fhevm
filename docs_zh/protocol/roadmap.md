# 路线图

本文档预览了 FHEVM 即将推出的功能。除了此处列出的内容外，您还可以在 GitHub 上[提交您的功能请求](https://github.com/zama-ai/fhevm/issues/new)。

## 功能

| 名称 | 描述 | 预计完成时间 |
| --- | --- | --- |
| Foundry 模板 | [Forge](https://book.getfoundry.sh/reference/forge/forge) | 25 年第一季度 |

## 操作

| 名称 | 函数名称 | 类型 | 预计完成时间 |
| --- | --- | --- | --- |
| 有符号整数 | `eintX` | | 即将推出 |
| 带溢出检查的加法 | `FHE.safeAdd` | 二元，解密 | 即将推出 |
| 带溢出检查的减法 | `FHE.safeSub` | 二元，解密 | 即将推出 |
| 带溢出检查的乘法 | `FHE.safeMul` | 二元，解密 | 即将推出 |
| 随机有符号整数 | `FHE.randEintX()` | 随机 | - |
| 除法 | `FHE.div` | 二元 | - |
| 取余 | `FHE.rem` | 二元 | - |
| 集合包含 | `FHE.isIn()` | 二元 | - |

{% hint style="info" %}
完全在链上生成的随机加密整数。目前，通过在明文中​​使用 PRNG 作为模型来实现。不适用于生产！
{% endhint %}
