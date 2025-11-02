# ACL 示例

本页面提供了有关如何在 FHEVM 中使用和实现 ACL（访问控制列表）的详细说明和示例。有关 ACL 概念及其重要性的概述，请参阅[访问控制列表 (ACL) 概述](./)。

## 控制访问：永久和临时许可

ACL 系统允许您定义两种类型的访问密文权限：

### 永久许可

- **函数**：`FHE.allow(ciphertext, address)`
- **目的**：授予特定地址对密文的永久访问权限。
- **存储**：权限保存在专用的 ACL 合约中，使其可在交易之间使用。

#### 替代的 Solidity 语法

由于 FHE 是一个 Solidity 库，您还可以使用方法链式语法来授予许可。

```solidity
using FHE for *;
ciphertext.allow(address1).allow(address2);
```

这等效于调用 `FHE.allow(ciphertext, address1)`，然后调用 `FHE.allow(ciphertext, address2)`。

### 临时许可

- **函数**：`FHE.allowTransient(ciphertext, address)`
- **目的**：在单个交易期间授予临时访问权限。
- **存储**：权限存储在临时存储中以节省 gas 成本。
- **用例**：非常适合在交易期间在函数或合约之间传递加密值。

#### 替代的 Solidity 语法

由于 FHE 是一个 Solidity 库，因此方法链式也适用于临时许可。

```solidity
using FHE for *;
ciphertext.allowTransient(address1).allowTransient(address2);
```

### 语法糖

- **函数**：`FHE.allowThis(ciphertext)`
- **等效于**：`FHE.allow(ciphertext, address(this))`
- **目的**：简化授予当前合约管理密文的永久访问权限。

#### 替代的 Solidity 语法

由于 FHE 是一个 Solidity 库，您还可以将方法链式语法用于 allowThis。

```solidity
using FHE for *;
ciphertext.allowThis();
```

#### 公开解密

要使密文可公开解密，您可以使用 `FHE.makePubliclyDecryptable(ciphertext)` 函数。这会授予任何人解密权限，这对于希望所有人都可访问加密值的场景很有用。

```solidity
// 授予密文的公共解密权限
FHE.makePubliclyDecryptable(ciphertext);

// 或使用方法语法：
ciphertext.makePubliclyDecryptable();
```

- **函数**：`FHE.makePubliclyDecryptable(ciphertext)`
- **目的**：使密文可被任何人解密。
- **用例**：当您想发布加密结果或数据时。

> 您可以直接在密文对象上组合多种许可方法（例如 `.allow()`、`.allowThis()`、`.allowTransient()`），以便在一个流畅的语句中向多个地址或合约授予访问权限。
>
> **示例**
>
> ```solidity
> // 授予一个地址临时访问权限，并授予另一个地址永久访问权限
> ciphertext.allowTransient(address1).allow(address2);
>
> // 授予当前合约和另一个地址永久访问权限
> ciphertext.allowThis().allow(address1);
> ```

## 最佳实践

### 验证发送者访问权限

在将密文作为输入进行处理时，必须验证发送者是否有权与提供的加密数据进行交互。未能执行此验证可能会使系统遭受推断攻击，恶意行为者试图推断私有信息。

#### 示例场景：机密 ERC20 攻击

考虑一个**机密 ERC20 代币**。控制两个帐户（**帐户 A** 和**帐户 B**）的攻击者，在帐户 A 中有 100 个代币，可以如下利用该系统：

1. 攻击者尝试将目标用户的加密余额从**帐户 A** 发送到**帐户 B**。
2. 观察交易结果，攻击者获得信息：
   - **如果成功**：目标的余额等于或小于 100 个代币。
   - **如果失败**：目标的余额超过 100 个代币。

这种类型的攻击允许攻击者在没有明确访问权限的情况下推断私人余额。

为防止这种情况，请始终使用 `FHE.isSenderAllowed()` 函数来验证发送者是否具有对正在转移的加密金额的合法访问权限。

---

#### 示例：安全验证

```solidity
function transfer(address to, euint64 encryptedAmount) public {
  // 确保发送者有权访问加密金额
  require(FHE.isSenderAllowed(encryptedAmount), "Unauthorized access to encrypted amount.");

  // 继续执行其他逻辑
  ...
}
```

通过强制执行此检查，您可以防范推断攻击，并确保只有授权实体才能操作加密值。

## 用户解密的 ACL

如果用户可以解密密文，则必须向他们明确授予访问权限。此外，用户解密机制需要与合约地址关联的公钥签名。因此，需要解密的值必须明确授权给用户和合约。

由于用户解密机制，用户签署了与特定合约关联的公钥；因此，密文也需要被允许用于该合约。

### 示例：ConfidentialERC20 中的安全转移

```solidity
function transfer(address to, euint64 encryptedAmount) public {
  require(FHE.isSenderAllowed(encryptedAmount), "The caller is not authorized to access this encrypted amount.");
  euint64 amount = FHE.asEuint64(encryptedAmount);
  ebool canTransfer = FHE.le(amount, balances[msg.sender]);

  euint64 newBalanceTo = FHE.add(balances[to], FHE.select(canTransfer, amount, FHE.asEuint64(0)));
  balances[to] = newBalanceTo;
  // 允许此新余额用于合约和所有者。
  FHE.allowThis(newBalanceTo);
  FHE.allow(newBalanceTo, to);

  euint64 newBalanceFrom = FHE.sub(balances[from], FHE.select(canTransfer, amount, FHE.asEuint64(0)));
  balances[from] = newBalanceFrom;
  // 允许此新余额用于合约和所有者。
  FHE.allowThis(newBalanceFrom);
  FHE.allow(newBalanceFrom, from);
}
```

通过了解如何授予和验证权限，您可以有效地管理 FHEVM 智能合约中对加密数据的访问。有关其他上下文，请参阅 [ACL 概述](./)。
