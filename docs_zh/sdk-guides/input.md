# 输入注册

本文档介绍了如何将密文注册到 FHEVM。
将密文注册到 FHEVM 允许将来在链上使用 `FHE.fromExternal` solidity 函数。
所有为与 FHEVM 一起使用而加密的值都是在协议的公钥下加密的。

```ts
// 我们首先创建一个缓冲区，用于加密值并将其注册到 fhevm
const buffer = instance.createEncryptedInput(
  // 允许与“新”密文交互的合约地址
  contractAddress,
  // 允许将密文导入到 `contractAddress` 处合约的实体地址
  userAddress,
);

// 我们添加具有关联数据类型方法的值
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

对于实现了以下内容的合约 `MyContract`，可以添加两个“新”密文。

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

使用 `ethers` 中的合约 `my_contract`，可以如下调用 add 函数。

```js
my_contract.add(ciphertexts.handles[0], ciphertexts.handles[1], ciphertexts.inputProof);
```
