# Foundry

本指南介绍了如何将 Foundry 与 FHEVM 一起用于开发智能合约。

虽然 Foundry 模板目前正在开发中，但我们强烈建议您暂时使用 [Hardhat 模板](getting-started/quick-start-tutorial/setup.md))，因为它为 FHEVM 智能合约提供了经过全面测试和支持的开发环境。

但是，您仍然可以将 Foundry 与 FHEVM 的模拟版本一起使用，但请注意，**不**推荐使用此方法，因为模拟版本与真实 FHEVM 节点的实现不完全等效（请参阅 hardhat 中的警告）。为此，您需要在 solidity 源文件中将 `FHE.sol` 的导入从 `@fhevm/solidity/lib/FHE.sol` 重命名为 `fhevm/mocks/FHE.sol`。
