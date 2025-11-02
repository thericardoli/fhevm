# 如何将您的智能合约转换为 FHEVM 智能合约？

本简短指南将引导您将标准的 Solidity 合约转换为利用 FHEVM 的全同态加密 (FHE) 的合约。这种方法让您可以像往常一样开发您的合约逻辑，然后对其进行调整以支持加密计算以保护隐私。

在本指南中，我们将重点关注一个投票合约示例。

---

## 1. 从标准 Solidity 合约开始

首先像往常一样用 Solidity 编写您的投票合约。专注于实现核心逻辑和功能。

```solidity
// 标准 Solidity 投票合约示例
pragma solidity ^0.8.0;

contract SimpleVoting {
    mapping(address => bool) public hasVoted;
    uint64 public yesVotes;
    uint64 public noVotes;
    uint256 public voteDeadline;

    function vote(bool support) public {
        require(block.timestamp <= voteDeadline, "Too late to vote");
        require(!hasVoted[msg.sender], "Already voted");
        hasVoted[msg.sender] = true;

        if (support) {
            yesVotes += 1;
        } else {
            noVotes += 1;
        }
    }

    function getResults() public view returns (uint64, uint64) {
        return (yesVotes, noVotes);
    }
}
```

---

## 2. 识别敏感数据和操作

检查您的合约并确定哪些变量、函数或计算需要隐私。
在此示例中，投票计数（`yesVotes`、`noVotes`）和个人投票应被加密。

---

## 3. 集成 FHEVM 并相应地更新您的业务逻辑。

对于已识别的敏感部分，将标准数据类型和操作替换为其 FHEVM 等效项。使用加密类型和 FHEVM 库函数对加密数据执行计算。

```solidity
pragma solidity ^0.8.0;

import "@fhevm/solidity/lib/FHE.sol";
import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract EncryptedSimpleVoting is SepoliaConfig {
    enum VotingStatus {
        Open,
        DecryptionInProgress,
        ResultsDecrypted
    }
    mapping(address => bool) public hasVoted;

    VotingStatus public status;

    uint64 public decryptedYesVotes;
    uint64 public decryptedNoVotes;

    uint256 public voteDeadline;

    euint64 private encryptedYesVotes;
    euint64 private encryptedNoVotes;

    constructor() {
        encryptedYesVotes = FHE.asEuint64(0);
        encryptedNoVotes = FHE.asEuint64(0);

        FHE.allowThis(encryptedYesVotes);
        FHE.allowThis(encryptedNoVotes);
    }

    function vote(externalEbool support, bytes memory inputProof) public {
        require(block.timestamp <= voteDeadline, "Too late to vote");
        require(!hasVoted[msg.sender], "Already voted");
        hasVoted[msg.sender] = true;
        ebool isSupport = FHE.fromExternal(support, inputProof);
        encryptedYesVotes = FHE.select(isSupport, FHE.add(encryptedYesVotes, 1), encryptedYesVotes);
        encryptedNoVotes = FHE.select(isSupport, encryptedNoVotes, FHE.add(encryptedNoVotes, 1));
        FHE.allowThis(encryptedYesVotes);
        FHE.allowThis(encryptedNoVotes);

    }

    function requestVoteDecryption() public {
        require(block.timestamp > voteDeadline, "Voting is not finished");
        bytes32[] memory cts = new bytes32[](2);
        cts[0] = FHE.toBytes32(encryptedYesVotes);
        cts[1] = FHE.toBytes32(encryptedNoVotes);
        uint256 requestId = FHE.requestDecryption(cts, this.callbackDecryptVotes.selector);
        status = VotingStatus.DecryptionInProgress;
    }

    function callbackDecryptVotes(uint256 requestId, bytes memory cleartexts, bytes memory decryptionProof) public {
        FHE.checkSignatures(requestId, cleartexts, decryptionProof);

        (uint64 yesVotes, uint64 noVotes) = abi.decode(cleartexts, (uint64, uint64));
        decryptedYesVotes = yesVotes;
        decryptedNoVotes = noVotes;
        status = VotingStatus.ResultsDecrypted;
    }

    function getResults() public view returns (uint64, uint64) {
        require(status == VotingStatus.ResultsDecrypted, "Results were not decrypted");
        return (
            decryptedYesVotes,
            decryptedNoVotes
        );
    }
}
```

在必要时调整您的合约代码以接受和返回加密数据。这可能涉及更改函数参数和返回类型以使用密文而不是明文值，如上所示。

- `vote` 函数现在有两个参数：`support` 和 `inputProof`。
- `getResults` 只能在解密发生后调用。否则，解密的结果对任何人都不可见。

然而，这远非主要的变化。如此示例所示，使用 FHEVM 通常需要重新架构原始逻辑以支持隐私。

在更新的代码中，逻辑变为异步；结果是隐藏的，直到必须明确向预言机发出请求以公开解密投票结果。

## 结论

如此简短的指南所示，与 FHEVM 集成不仅需要与 FHEVM 堆栈集成，还需要重构您的业务逻辑以支持在逻辑的加密和非加密组件之间切换的机制。
