# 公共解密

本文档介绍了如何对 FHEVM 密文执行公共解密。
当您希望每个人都看到密文中的值时，例如私下拍卖的结果，就需要进行公共解密。
公共解密可以通过中继器 HTTP 端点或调用链上解密预言机来完成。

## HTTP 公共解密

使用以下代码片段可以轻松调用中继器的公共解密端点。

```ts
// 要解密的密文句柄列表
const handles = [
  "0x830a61b343d2f3de67ec59cb18961fd086085c1c73ff0000000000aa36a70000",
  "0x98ee526413903d4613feedb9c8fa44fe3f4ed0dd00ff0000000000aa36a70400",
  "0xb837a645c9672e7588d49c5c43f4759a63447ea581ff0000000000aa36a70700",
];

// 解密值列表
// {
//  '0x830a61b343d2f3de67ec59cb18961fd086085c1c73ff0000000000aa36a70000': true,
//  '0x98ee526413903d4613feedb9c8fa44fe3f4ed0dd00ff0000000000aa36a70400': 242n,
//  '0xb837a645c9672e7588d49c5c43f4759a63447ea581ff0000000000aa36a70700': '0xfC4382C084fCA3f4fB07c3BCDA906C01797595a8'
// }
const values = instance.publicDecrypt(handles);
```

## 链上公共解密

有关更多详细信息，请参阅[链上预言机公共解密页面](https://docs.zama.ai/protocol/solidity-guides/smart-contract/oracle)。
