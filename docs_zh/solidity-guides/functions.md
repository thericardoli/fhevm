# 智能合约 - FHEVM API

本文档概述了 `FHE` Solidity 库中可用的函数。FHE 库提供了处理加密类型并对其执行操作的功能。它在 Solidity 中实现了全同态加密 (FHE) 操作。

## 概述

`FHE` Solidity 库为在智能合约中使用加密数据类型和执行全同态加密 (FHE) 操作提供了基本功能。它旨在简化开发人员的体验，同时保持灵活性和性能。

### **核心功能**

- **同态操作**：支持对加密值进行算术、按位和比较操作。
- **密文-明文互操作性**：支持混合加密和明文操作数的操作，前提是明文操作数的大小不超过加密操作数的大小。
  - 示例：`add(uint8 a, euint8 b)` 有效，但 `add(uint32 a, euint16 b)` 无效。
  - 密文-明文操作通常比密文-密文操作更快，消耗的 gas 更少。
- **隐式向上转型**：在必要时自动调整操作数类型，以确保在对加密数据进行操作期间的兼容性。

### **主要功能**

- **灵活性**：处理各种加密数据类型，包括布尔值、整数、地址和字节数组。
- **性能优化**：通过支持混合明文和密文输入的优化运算符版本来优先考虑高效计算。
- **易于使用**：在所有支持的数据类型中提供一致的 API，从而实现流畅的开发人员体验。

该库确保对加密数据的所有操作都遵循 FHE 的约束，同时抽象出复杂性，使开发人员能够专注于构建保护隐私的智能合约。

## 类型

### 加密数据类型

#### 布尔值

- `ebool`：加密的布尔值

#### 无符号整数

- `euint8`：加密的 8 位无符号整数
- `euint16`：加密的 16 位无符号整数
- `euint32`：加密的 32 位无符号整数
- `euint64`：加密的 64 位无符号整数
- `euint128`：加密的 128 位无符号整数
- `euint256`：加密的 256 位无符号整数

#### 地址

- `eaddress`：加密的以太坊地址

#### 特殊类型

- `externalEbool`：加密布尔值的输入类型
- `externalEuint8`：加密 8 位无符号整数值的输入类型
- `externalEuint16`：加密 16 位无符号整数值的输入类型
- `externalEuint32`：加密 32 位无符号整数值的输入类型
- `externalEuint64`：加密 64 位无符号整数值的输入类型
- `externalEuint128`：加密 128 位无符号整数值的输入类型
- `externalEuint256`：加密 256 位无符号整数值的输入类型
- `externalEaddress`：加密以太坊地址的输入类型

### 类型转换

- **在加密类型之间进行转换**：`FHE.asEbool` 将加密的整数转换为加密的布尔值
- **转换为加密类型**：`FHE.asEuintX` 将明文值转换为加密类型
- **转换为加密地址**：`FHE.asEaddress` 将明文地址转换为加密地址

#### `asEuint`

`asEuint` 函数有三个用途：

- 验证密文字节并向调用智能合约返回一个有效的句柄；
- 将 `euintX` 类型的密文转换为 `euintY` 类型的密文，其中 `X != Y`；
- 对明文值进行平凡加密。

第一种情况用于处理加密的输入，例如用户提供的密文。这些通常包含在交易负载中。

第二种情况不言自明。当 `X > Y` 时，最高有效位将被丢弃。当 `X < Y` 时，密文将用 `0` 的平凡加密向左填充。

第三种情况用于“加密”一个公共值，以便可以将其用作密文。请注意，我们所谓的平凡加密在任何意义上都**不**安全。当对明文值进行平凡加密时，该值在密文字节中仍然可见。有关平凡加密的更多信息，请参阅[此处](https://www.zama.ai/post/tfhe-deep-dive-part-1)。

**示例**

```solidity
// 第一种情况
function asEuint8(bytes memory ciphertext) internal view returns (euint8)
// 第二种情况
function asEuint16(euint8 ciphertext) internal view returns (euint16)
// 第三种情况
function asEuint16(uint16 value) internal view returns (euint16)
```

#### `asEbool`

`asEbool` 函数的行为与 `asEuint` 函数类似，但用于加密的布尔值。

## 核心函数

### 配置

```solidity
function setFHEVM(FHEVMConfig.FHEVMConfigStruct memory fhevmConfig) internal
```

设置 FHEVM 的加密操作配置。

### 初始化检查

```solidity
function isInitialized(T v) internal pure returns (bool)
```

如果加密值已初始化，则返回 true，否则返回 false。支持所有加密类型（T 可以是 ebool、euintX、eaddress、ebytesX）。

### 算术运算

可用于 euint* 类型：

```solidity
function add(T a, T b) internal returns (T)
function sub(T a, T b) internal returns (T)
function mul(T a, T b) internal returns (T)
```

- 算术：`FHE.add`、`FHE.sub`、`FHE.mul`、`FHE.min`、`FHE.max`、`FHE.neg`、`FHE.div`、`FHE.rem`
  - 注意：`div` 和 `rem` 操作仅支持明文除数

> :warning: 带有 FHE 操作的函数不能标记为 `view`，因为 FHE 操作需要消耗 gas 来执行，因为它们总是涉及状态更改。例如，您不能在 view 函数中计算并返回两个加密值的加密和。

#### 算术运算（`add`、`sub`、`mul`、`div`、`rem`）

同态地执行操作。

请注意，除法/取余仅支持明文除数。

**示例**

```solidity
// a + b
function add(euint8 a, euint8 b) internal view returns (euint8)
function add(euint8 a, euint16 b) internal view returns (euint16)
function add(uint32 a, euint32 b) internal view returns (euint32)

// a / b
function div(euint8 a, uint8 b) internal pure returns (euint8)
function div(euint16 a, uint16 b) internal pure returns (euint16)
function div(euint32 a, uint32 b) internal pure returns (euint32)
```

#### 最小/最大运算 - `min`、`max`

可用于 euint* 类型：

```solidity
function min(T a, T b) internal returns (T)
function max(T a, T b) internal returns (T)
```

返回两个给定值的最小值（或最大值）。

**示例**

```solidity
// min(a, b)
function min(euint32 a, euint16 b) internal view returns (euint32)

// max(a, b)
function max(uint32 a, euint8 b) internal view returns (euint32)
```

#### 一元运算符（`neg`、`not`）

有两个一元运算符：`neg` (`-`) 和 `not` (`!`)。请注意，由于我们处理的是无符号整数，因此取反的结果被解释为模逆。`not` 运算符返回对操作数的所有位取反后获得的值。

{% hint style="info" %}
有关这些运算符行为的更多信息，请参阅 [TFHE-rs 文档](https://docs.zama.ai/tfhe-rs/fhe-computation/operations/arithmetic-operations)。
{% endhint %}

### 按位运算

- 按位：`FHE.and`、`FHE.or`、`FHE.xor`、`FHE.not`、`FHE.shl`、`FHE.shr`、`FHE.rotl`、`FHE.rotr`

#### 按位运算（`AND`、`OR`、`XOR`）

与其他二元运算不同，按位运算本身不支持混合密文和明文输入。为了简化开发人员的体验，`FHE` 库为这些操作添加了函数重载。如此处示例所示，此类重载在实际调用操作函数之前隐式地进行平凡加密。

可用于 euint* 类型：

```solidity
function and(T a, T b) internal returns (T)
function or(T a, T b) internal returns (T)
function xor(T a, T b) internal returns (T)
```

**示例**

```solidity
// a & b
function and(euint8 a, euint8 b) internal view returns (euint8)

// 在调用运算符之前对 `b` 进行隐式平凡加密
function and(euint8 a, uint16 b) internal view returns (euint16)
```

#### 位移运算（`<<`, `>>`）

将 `a` 的二进制表示的位移动 `b` 个位置。

**示例**

```solidity
// a << b
function shl(euint16 a, euint8 b) internal view returns (euint16)
// a >> b
function shr(euint32 a, euint16 b) internal view returns (euint32)
```

#### 旋转运算

将 `a` 的二进制表示的位旋转 `b` 个位置。

**示例**

```solidity
function rotl(euint16 a, euint8 b) internal view returns (euint16)
function rotr(euint32 a, euint16 b) internal view returns (euint32)
```

### 比较运算（`eq`, `ne`, `ge`, `gt`, `le`, `lt`）

{% hint style="info" %}
**请注意**，在密文-明文操作的情况下，由于我们的后端仅接受明文右操作数，因此使用明文左操作数调用操作实际上会颠倒操作数顺序并调用_相反的_比较。
{% endhint %}

比较运算的结果是一个加密的布尔值（`ebool`）。在后端，布尔值由位宽为 8 的加密无符号整数表示，但这已被 Solidity 库抽象掉了。

可用于所有加密类型：

```solidity
function eq(T a, T b) internal returns (ebool)
function ne(T a, T b) internal returns (ebool)
```

euint* 类型的其他比较：

```solidity
function ge(T a, T b) internal returns (ebool)
function gt(T a, T b) internal returns (ebool)
function le(T a, T b) internal returns (ebool)
function lt(T a, T b) internal returns (ebool)
```

#### 示例

```solidity
// a == b
function eq(euint32 a, euint16 b) internal view returns (ebool)

// 实际上返回 `lt(b, a)`
function gt(uint32 a, euint16 b) internal view returns (ebool)

// 实际上返回 `gt(a, b)`
function gt(euint16 a, uint32 b) internal view returns (ebool)
```

### 多路复用器运算符（`select`）

```solidity
function select(ebool control, T a, T b) internal returns (T)
```

如果 control 为 true，则返回 a，否则返回 b。可用于 ebool、eaddress 和 ebytes* 类型。

此运算符接受三个输入。第一个输入 `b` 的类型为 `ebool`，另外两个的类型为 `euintX`。如果 `b` 是 `true` 的加密，则返回第一个整数参数。否则，返回第二个整数参数。

#### 示例

```solidity
// if (b == true) return val1 else return val2
function select(ebool b, euint8 val1, euint8 val2) internal view returns (euint8) {
  return FHE.select(b, val1, val2);
}
```

### 生成随机加密整数

随机加密整数可以完全在链上生成。

这只能在交易期间完成，而不能在 `eth_call` RPC 方法上完成，因为 PRNG 状态需要在生成期间在链上进行突变。

#### 示例

```solidity
// 生成一个随机加密的无符号整数 `r`。
euint32 r = FHE.randEuint32();
```

## 访问控制函数

`FHE` 库提供了一组强大的访问控制函数，用于管理对加密值的权限。这些函数确保只有授权的帐户或合约才能访问或操作加密数据。

### 权限管理

#### 函数

```solidity
function allow(T value, address account) internal
function allowThis(T value) internal
function allowTransient(T value, address account) internal
```

**描述**

- **`allow`**：授予特定地址**永久访问权限**。权限持久地存储在专用的 ACL 合约中。
- **`allowThis`**：授予**当前合约**对加密值的访问权限。
- **`allowTransient`**：在交易期间授予特定地址**临时访问权限**。权限存储在瞬态存储中，以降低 gas 成本。

#### 访问控制列表 (ACL) 概述

`allow` 和 `allowTransient` 函数可以对谁可以访问和解密加密值进行精细控制。临时权限（`allowTransient`）非常适合在只需要在单个交易中访问的情况下最大限度地减少 gas 使用。

**示例：授予访问权限**

```solidity
// 存储一个加密值。
euint32 r = FHE.asEuint32(94);

// 授予当前合约永久访问权限。
FHE.allowThis(r);

// 授予调用者永久访问权限。
FHE.allow(r, msg.sender);

// 授予外部帐户临时访问权限。
FHE.allowTransient(r, 0x1234567890abcdef1234567890abcdef12345678);
```

### 权限检查

#### 函数

```solidity
function isAllowed(T value, address account) internal view returns (bool)
function isSenderAllowed(T value) internal view returns (bool)
```

#### 描述

- **`isAllowed`**：检查特定地址是否具有访问密文的权限。
- **`isSenderAllowed`**：与 `isAllowed` 类似，但自动检查 `msg.sender` 的权限。

{% hint style="info" %}
如果密文已为指定地址授权，则这两个函数都返回 `true`，无论许可是存储在 ACL 合约中还是瞬态存储中。
{% endhint %}

#### 验证权限

这些函数有助于确保只有授权的帐户或合约才能访问加密值。

**示例：权限验证**

```solidity
// 存储一个加密值。
euint32 r = FHE.asEuint32(94);

// 验证当前合约是否被允许访问该值。
bool isContractAllowed = FHE.isAllowed(r, address(this)); // 返回 true

// 验证调用者是否具有访问该值的权限。
bool isCallerAllowed = FHE.isSenderAllowed(r); // 取决于 msg.sender
```

## 存储管理

### **函数**

```solidity
function cleanTransientStorage() internal
```

### 描述

- **`cleanTransientStorage`**：从瞬态存储中删除所有临时权限。在交易结束时使用此函数可确保没有残留的权限。

### 示例

```solidity
// 在函数结束时清理瞬态存储。
function finalize() public {
  // 执行操作...

  // 清理瞬态存储。
  FHE.cleanTransientStorage();
}
```

## 附加说明

- **底层实现**：
  所有加密操作和访问控制功能都通过底层的 `Impl` 库执行。
- **未初始化的值**：
  未初始化的加密值在计算中被视为 `0`（对于整数）或 `false`（对于布尔值）。
- **隐式转换**：
  通过隐式转换支持不同位宽的加密整数之间的类型转换，无需额外的开发人员干预即可实现无缝操作。
