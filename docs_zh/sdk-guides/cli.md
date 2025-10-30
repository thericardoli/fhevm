# 使用命令行界面

`fhevm` 命令行界面（CLI）工具提供了一种简单高效的方式来加密数据，以便与区块链的全同态加密（FHE）系统一起使用。本指南解释如何安装和使用 CLI 加密整数和布尔值以用于机密智能合约。

## 安装

在继续之前，请确保您的系统上安装了 [Node.js](https://nodejs.org/)。然后，全局安装 `@zama-fhe/relayer-sdk` 包以启用 CLI 工具：

```bash
npm install -g @zama-fhe/relayer-sdk
```

安装后，您可以使用 `relayer` 命令访问 CLI。使用以下命令验证安装并探索可用命令：

```bash
relayer help
```

## 加密数据

CLI 允许您加密整数和布尔值以在智能合约中使用。加密使用区块链的 FHE 公钥执行，确保数据的机密性。

### 语法

```bash
relayer encrypt --node <NODE_URL> <CONTRACT_ADDRESS> <USER_ADDRESS> <DATA:TYPE>...
```

- **`--node`**：指定区块链节点的 RPC URL（例如 `http://localhost:8545`）。
- **`<CONTRACT_ADDRESS>`**：与加密数据交互的合约地址。
- **`<USER_ADDRESS>`**：与加密数据关联的用户地址。
- **`<DATA:TYPE>`**：要加密的数据，后跟其类型：
  - `:64` 表示 64 位整数
  - `:1` 表示布尔值

### 示例用法

为合约地址 `0x8Fdb26641d14a80FCCBE87BF455338Dd9C539a50` 和用户地址 `0xa5e1defb98EFe38EBb2D958CEe052410247F4c80` 加密整数 `71721075`（64 位）和布尔值 `1`：

```bash
relayer encrypt 0x8Fdb26641d14a80FCCBE87BF455338Dd9C539a50 0xa5e1defb98EFe38EBb2D958CEe052410247F4c80 71721075:64 1:1
```
