# 输入注册

本文档说明如何将密文注册到 FHEVM。
将密文注册到 FHEVM 允许在链上使用 `FHE.fromExternal` solidity 函数将来使用。
所有为与 FHEVM 一起使用而加密的值都在协议的公钥下加密。

```ts
// 我们首先为要加密并注册到 fhevm 的值创建一个缓冲区
const buffer = instance.createEncryptedInput(
  // 允许与"新鲜"密文交互的合约地址
  contractAddress,
  // 允许将密文导入到 `contractAddress` 处的合约的实体地址
  userAddress,
);

// 我们使用相关的数据类型方法添加值
buffer.add64(BigInt(23393893233));
buffer.add64(BigInt(1));
// buffer.addBool(false);
// buffer.add8(BigInt(43));
// buffer.add16(BigInt(87));
// buffer.add32(BigInt(2339389323));
// buffer.add128(BigInt(233938932390));
// buffer.addAddress('0xa5e1defb98EFe38EBb2D958CEe052410247F4c80');
// buffer.add256(BigInt('2339389323922393930'));

// 这将加密值，为其生成知识证明，然后使用中继器上传密文。
// 此操作将返回密文句柄列表。
const ciphertexts = await buffer.encrypt();
```

使用实现以下内容的合约 `MyContract`，可以添加两个"新鲜"密文。

```solidity
contract MyContract {
  ...

  function add(
    externalEuint64 a,
    externalEuint64 b,
    bytes calldata proof
  ) public virtual returns (euint64) {
    return FHE.add(FHE.fromExternal(a, proof), FHE.fromExternal(b, proof))
  }
}
```

使用 `ethers` 的合约 `my_contract`，可以按如下方式调用 add 函数。

```js
my_contract.add(ciphertexts.handles[0], ciphertexts.handles[1], ciphertexts.inputProof);
```
