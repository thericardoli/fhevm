# 目录

- [概述](README.md)

## 入门

- [什么是 FHEVM Solidity](getting-started/overview.md)
- [设置 Hardhat](getting-started/quick-start-tutorial/setup.md)
- [快速入门教程](getting-started/quick-start-tutorial/README.md)
  - [1. 设置 Hardhat](getting-started/quick-start-tutorial/setup.md)
  - [2. 编写一个简单的合约](getting-started/quick-start-tutorial/write_a_simple_contract.md)
  - [3. 将其转换为 FHEVM](getting-started/quick-start-tutorial/turn_it_into_fhevm.md)
  - [4. 测试 FHEVM 合约](getting-started/quick-start-tutorial/test_the_fhevm_contract.md)

## 智能合约

- [配置](configure.md)
  - [合约地址](contract_addresses.md)
- [支持的类型](types.md)
- [对加密类型的操作](operations/README.md)
  - [类型转换和平凡加密](operations/casting.md)
  - [生成随机数](operations/random.md)
- [加密输入](inputs.md)
- [访问控制列表](acl/README.md)
  - [ACL 示例](acl/acl_examples.md)
  - [重组处理](acl/reorgs_handling.md)
- [逻辑](<README (1).md>)
  - [分支](logics/conditions.md)
  - [处理分支和条件](logics/loop.md)
  - [错误处理](logics/error_handling.md)
- [解密](decryption/oracle.md)

## 开发指南

- [Hardhat 插件](hardhat/README.md)
  - [设置 Hardhat](getting-started/quick-start-tutorial/setup.md)
  - [在 Hardhat 中编写 FHEVM 测试](hardhat/write_test.md)
  - [部署合约并运行测试](hardhat/run_test.md)
  - [编写支持 FHEVM 的 Hardhat 任务](hardhat//write_task.md)
- [Foundry](foundry.md)
- [HCU](hcu.md)
- [迁移到 v0.7](migration.md)
- [如何将您的智能合约转换为 FHEVM 智能合约？](transform_smart_contract_with_fhevm.md)
