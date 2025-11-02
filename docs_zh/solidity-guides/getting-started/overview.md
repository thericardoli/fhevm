# 主要功能

本文档概述了 FHEVM 智能合约库的主要功能。

### 配置和初始化

使用 FHEVM 的智能合约需要正确的配置和初始化：

- **环境设置**：导入并继承特定于环境的配置合约
- **中继器配置**：为加密操作配置安全的中继器访问
- **初始化检查**：在使用前验证加密变量是否已正确初始化

有关更多信息，请参阅[配置](../configure.md)。

### 加密数据类型

FHEVM 引入了与 Solidity 兼容的加密数据类型：

- **布尔值**：`ebool`
- **无符号整数**：`euint8`、`euint16`、`euint32`、`euint64`、`euint128`、`euint256`
- **地址**：`eaddress`
- **输入**：`externalEbool`、`externalEaddress`、`externalEuintXX`，用于处理加密的输入数据

加密数据表示为密文句柄，确保了安全的计算和交互。

有关更多信息，请参阅[加密类型的使用](../types.md)。

### 类型转换

fhevm 提供了在加密类型之间进行转换的函数：

- **在加密类型之间进行转换**：`FHE.asEbool` 将加密的整数转换为加密的布尔值
- **转换为加密类型**：`FHE.asEuintX` 将明文值转换为加密类型
- **转换为加密地址**：`FHE.asEaddress` 将明文地址转换为加密地址

有关更多信息，请参阅[加密类型的使用](../types.md)。

### 机密计算

fhevm 支持加密操作的符号执行，支持：

- **算术**：`FHE.add`、`FHE.sub`、`FHE.mul`、`FHE.min`、`FHE.max`、`FHE.neg`、`FHE.div`、`FHE.rem`
  - 注意：`div` 和 `rem` 操作仅支持明文除数
- **按位**：`FHE.and`、`FHE.or`、`FHE.xor`、`FHE.not`、`FHE.shl`、`FHE.shr`、`FHE.rotl`、`FHE.rotr`
- **比较**：`FHE.eq`、`FHE.ne`、`FHE.lt`、`FHE.le`、`FHE.gt`、`FHE.ge`
- **高级**：`FHE.select` 用于对加密条件进行分支，`FHE.randEuintX` 用于链上随机性。

有关操作的更多信息，请参阅[对加密类型的操作](../operations/README.md)。

有关条件分支的更多信息，请参阅[FHE 中的条件逻辑](../logics/conditions.md)。

有关随机数生成的更多信息，请参阅[生成随机加密数](../operations/random.md)。

### 访问控制机制

fhevm 使用基于区块链的访问控制列表 (ACL) 强制执行访问控制：

- **持久访问**：`FHE.allow`、`FHE.allowThis` 授予密文的永久权限。
- **瞬态访问**：`FHE.allowTransient` 为特定交易提供临时访问权限。
- **验证**：`FHE.isSenderAllowed` 确保只有授权实体才能与密文交互。

有关更多信息，请参阅 [ACL](../acl/README.md)。
