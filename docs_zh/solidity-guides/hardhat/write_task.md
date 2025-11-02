在本节中，您将学习如何编写自定义 FHEVM Hardhat 任务。

编写任务是在 Sepolia 网络上测试 FHEVM 智能合约的一种 gas 高效且灵活的方法。创建一个自定义任务很简单。

# 先决条件

- 您应该熟悉 Hardhat 任务。如果您是新手，请参阅 [Hardhat 任务官方文档](https://hardhat.org/hardhat-runner/docs/guides/tasks#writing-tasks)。
- 您应该已经**完成**了 [FHEVM 教程](https://docs.zama.ai/protocol/solidity-guides/getting-started/setup)。
- 本页面提供了 [fhevm-hardhat-template](https://github.com/zama-ai/fhevm-hardhat-template) 存储库中 [tasks/FHECounter.ts](https://github.com/zama-ai/fhevm-hardhat-template/blob/main/tasks/FHECounter.ts) 文件中包含的 `task:decrypt-count` 任务的逐步演练。

{% stepper %}
{% step %}

# 一个基本的 Hardhat 任务。

让我们从一个简单的示例开始：从一个基本的 `Counter.sol` 合约中获取当前的计数器值。

如果您已经熟悉 Hardhat 和自定义任务，那么下面的 TypeScript 代码应该看起来很熟悉并且很容易理解：

```ts
task("task:get-count", "Calls the getCount() function of Counter Contract")
  .addOptionalParam("address", "Optionally specify the Counter contract address")
  .setAction(async function (taskArguments: TaskArguments, hre) {
    const { ethers, deployments } = hre;

    const CounterDeployement = taskArguments.address
      ? { address: taskArguments.address }
      : await deployments.get("Counter");
    console.log(`Counter: ${CounterDeployement.address}`);

    const counterContract = await ethers.getContractAt("Counter", CounterDeployement.address);

    const clearCount = await counterContract.getCount();

    console.log(`Clear count    : ${clearCount}`);
});
```

现在，让我们修改此任务以使用 FHEVM 加密值。

{% endstep %}
{% step %}

# 注释掉现有逻辑并重命名

首先，注释掉现有逻辑，以便我们可以逐步添加 FHEVM 集成所需的更改。

```ts
task("task:get-count", "Calls the getCount() function of Counter Contract")
  .addOptionalParam("address", "Optionally specify the Counter contract address")
  .setAction(async function (taskArguments: TaskArguments, hre) {
    // const { ethers, deployments } = hre;

    // const CounterDeployement = taskArguments.address
    //   ? { address: taskArguments.address }
    //   : await deployments.get("Counter");
    // console.log(`Counter: ${CounterDeployement.address}`);

    // const counterContract = await ethers.getContractAt("Counter", CounterDeployement.address);

    // const clearCount = await counterContract.getCount();

    // console.log(`Clear count    : ${clearCount}`);
});
```

接下来，通过替换以下内容来重命名任务：

```ts
task("task:get-count", "Calls the getCount() function of Counter Contract")
```

替换为：

```ts
task("task:decrypt-count", "Calls the getCount() function of Counter Contract")
```

这将任务名称从 `task:get-count` 更新为 `task:decrypt-count`，反映出它现在包含 FHE 加密值的解密逻辑。

{% endstep %}
{% step %}

# 初始化 FHEVM CLI API

替换该行：

```ts
    // const { ethers, deployments } = hre;
```

替换为：

```ts
    const { ethers, deployments, fhevm } = hre;

    await fhevm.initializeCLIApi();
```

{% hint style="warning" %}
调用 `initializeCLIApi()` 至关重要。与 `test` 或 `compile` 等自动初始化 FHEVM 运行时环境的内置 Hardhat 任务不同，自定义任务要求您显式调用此函数。
**请确保在任务的最开始调用它**以确保环境已正确设置。
{% endhint %}

{% endstep %}
{% step %}

# 从 FHECounter 合约调用视图函数 `getCount`

替换以下注释掉的行：

```ts
    // const CounterDeployement = taskArguments.address
    //   ? { address: taskArguments.address }
    //   : await deployments.get("Counter");
    // console.log(`Counter: ${CounterDeployement.address}`);

    // const counterContract = await ethers.getContractAt("Counter", CounterDeployement.address);

    // const clearCount = await counterContract.getCount();
```

替换为 FHEVM 等效项：

```ts
    const FHECounterDeployement = taskArguments.address
      ? { address: taskArguments.address }
      : await deployments.get("FHECounter");
    console.log(`FHECounter: ${FHECounterDeployement.address}`);

    const fheCounterContract = await ethers.getContractAt("FHECounter", FHECounterDeployement.address);

    const encryptedCount = await fheCounterContract.getCount();
    if (encryptedCount === ethers.ZeroHash) {
      console.log(`encrypted count: ${encryptedCount}`);
      console.log("clear count    : 0");
      return;
    }
```

在这里，`encryptedCount` 是一个 FHE 加密的 `euint32` 原语。要检索实际值，我们需要在下一步中对其进行解密。

{% endstep %}
{% step %}

# 解密加密的计数值。

现在替换以下注释掉的行：

```ts
    // console.log(`Clear count    : ${clearCount}`);
```

替换为解密逻辑：

```ts
    const signers = await ethers.getSigners();
    const clearCount = await fhevm.userDecryptEuint(
      FhevmType.euint32,
      encryptedCount,
      FHECounterDeployement.address,
      signers[0],
    );
    console.log(`Encrypted count: ${encryptedCount}`);
    console.log(`Clear count    : ${clearCount}`);
```

此时，您的自定义 Hardhat 任务已完全配置为使用 FHE 加密值并准备好运行！

{% endstep %}
{% step %}

# 第 6 步：使用 Hardhat 节点运行您的自定义任务

#### 启动本地 Hardhat 节点：

- 打开一个新的终端窗口。
- 从根项目目录运行以下命令：

```sh
npx hardhat node
```

#### 在本地 Hardhat 节点上部署 FHECounter 智能合约

```sh
npx hardhat deploy --network localhost
```

#### 运行您的自定义任务

```sh
npx hardhat task:decrypt-count --network localhost
```

{% endstep %}
{% step %}

# 第 7 步：使用 Sepolia 运行您的自定义任务

#### 在 Sepolia 测试网上部署 FHECounter 智能合约（如果尚未部署）

```sh
npx hardhat deploy --network sepolia
```

#### 执行您的自定义任务

```sh
npx hardhat task:decrypt-count --network sepolia
```

{% endstep %}
{% endstepper %}
