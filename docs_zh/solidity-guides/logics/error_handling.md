# 错误处理

本文档介绍了如何在 FHEVM 智能合约中有效地处理错误。由于涉及加密数据的交易在条件不满足时不会自动回滚，因此开发人员需要替代机制来向用户传达错误。

## **错误处理中的挑战**

在加密数据的背景下：

1. **无自动回滚**：如果条件失败，交易不会回滚，这使得通知用户资金不足或输入无效等问题具有挑战性。
2. **有限的反馈**：加密计算缺乏直接的机制来在保持机密性的同时暴露失败原因。

## **推荐方法：使用处理程序进行错误记录**

为了应对这些挑战，请实现一个**错误处理程序**，为每个用户记录最近的错误。这允许 dApp 或前端查询错误状态并向用户提供适当的反馈。

### **示例实现**

以下合约代码段演示了如何实现和使用错误处理程序：

```solidity
struct LastError {
  euint8 error;      // 加密的错误代码
  uint timestamp;    // 错误的时间戳
}

// 定义错误代码
euint8 internal NO_ERROR;
euint8 internal NOT_ENOUGH_FUNDS;

constructor() {
  NO_ERROR = FHE.asEuint8(0);           // 代码 0：无错误
  NOT_ENOUGH_FUNDS = FHE.asEuint8(1);   // 代码 1：资金不足
}

// 存储每个地址的最后一个错误
mapping(address => LastError) private _lastErrors;

// 通知错误状态更改的事件
event ErrorChanged(address indexed user);

/**
 * @dev 为特定地址设置最后一个错误。
 * @param error 加密的错误代码。
 * @param addr 用户的地址。
 */
function setLastError(euint8 error, address addr) private {
  _lastErrors[addr] = LastError(error, block.timestamp);
  emit ErrorChanged(addr);
}

/**
 * @dev 带错误处理的内部转移函数。
 * @param from 发送方的地址。
 * @param to 接收方的地址。
 * @param amount 加密的转移金额。
 */
function _transfer(address from, address to, euint32 amount) internal {
  // 检查发送方是否有足够的余额进行转移
  ebool canTransfer = FHE.le(amount, balances[from]);

  // 记录错误状态：NO_ERROR 或 NOT_ENOUGH_FUNDS
  setLastError(FHE.select(canTransfer, NO_ERROR, NOT_ENOUGH_FUNDS), msg.sender);

  // 有条件地执行转移操作
  balances[to] = FHE.add(balances[to], FHE.select(canTransfer, amount, FHE.asEuint32(0)));
  FHE.allowThis(balances[to]);
  FHE.allow(balances[to], to);

  balances[from] = FHE.sub(balances[from], FHE.select(canTransfer, amount, FHE.asEuint32(0)));
  FHE.allowThis(balances[from]);
  FHE.allow(balances[from], from);
}
```

## **工作原理**

1. **定义错误代码**：
   - `NO_ERROR`：表示操作成功。
   - `NOT_ENOUGH_FUNDS`：表示转移余额不足。
2. **记录错误**：
   - 使用 `setLastError` 函数为特定地址记录最新的错误以及当前时间戳。
   - 发出 `ErrorChanged` 事件以通知外部系统（例如 dApp）错误状态的更改。
3. **条件更新**：
   - 使用 `FHE.select` 函数根据转移条件 (`canTransfer`) 更新余额并记录错误。
4. **前端集成**：
   - dApp 可以查询 `_lastErrors` 以获取用户的最新错误，并显示适当的反馈，例如“资金不足”或“交易成功”。

## **错误查询示例**

前端或其他合约可以查询 `_lastErrors` 映射以检索错误详细信息：

```solidity
/**
 * @dev 获取特定地址的最后一个错误。
 * @param user 用户的地址。
 * @return error 加密的错误代码。
 * @return timestamp 错误的时间戳。
 */
function getLastError(address user) public view returns (euint8 error, uint timestamp) {
  LastError memory lastError = _lastErrors[user];
  return (lastError.error, lastError.timestamp);
}
```

## **此方法的优点**

1. **用户反馈**：
   - 提供可操作的错误消息，而不会损害加密计算的机密性。
2. **可扩展的错误跟踪**：
   - 按用户记录错误，可以轻松识别和调试特定问题。
3. **事件驱动的通知**：
   - 使前端能够通过 `ErrorChanged` 事件实时对错误做出反应。

通过如上所示实现错误处理程序，开发人员可以确保无缝的用户体验，同时保持加密数据操作的隐私和完整性。
