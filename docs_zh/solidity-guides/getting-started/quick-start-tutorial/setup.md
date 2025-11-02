# 设置 Hardhat

在本节中，您将学习如何使用 **FHEVM Hardhat 模板** 设置 FHEVM Hardhat 开发环境，作为构建和测试全同态加密智能合约的起点。

## 创建本地 Hardhat 项目

{% stepper %} {% step %}

### 安装 Node.js TLS 版本

确保您的机器上已安装 Node.js。

- 从[官方网站](https://nodejs.org/en)下载并安装推荐的 LTS（长期支持）版本。
- 使用**偶数版本**（例如 `v18.x`、`v20.x`）

{% hint style="warning" %}
**Hardhat** 不支持奇数版本的 Node.js。如果您使用的是奇数版本（例如 v21.x、v23.x），Hardhat 将显示持续的警告消息并且可能会出现意外行为。
{% endhint %}

要验证您的安装：

```sh
node -v
npm -v
```

{% endstep %}

{% step %}

### 从 FHEVM Hardhat 模板创建一个新的 GitHub 存储库。

1. 在 GitHub 上，导航到 [FHEVM Hardhat 模板](https://github.com/zama-ai/fhevm-hardhat-template) 存储库的主页。
2. 在文件列表上方，单击绿色的**使用此模板**按钮。
3. 按照说明从 FHEVM Hardhat 模板创建一个新的存储库。

{% hint style="info" %}
请参阅 Github 文档：[从模板创建存储库](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template#creating-a-repository-from-a-template)
{% endhint %}

{% endstep %}

{% step %}

### 在本地克隆您新创建的 GitHub 存储库

现在您的 GitHub 存储库已创建，您可以将其克隆到您的本地机器上：

```sh
cd <your-preferred-location>
git clone <url-to-your-new-repo>

# 导航到您的新 FHEVM Hardhat 项目的根目录
cd <your-new-repo-name>
```

接下来，让我们安装您的本地 Hardhat 开发环境。 {% endstep %}

{% step %}

### 安装您的 FHEVM Hardhat 项目依赖项

从项目根目录运行：

```sh
npm install
```

这将安装 `package.json` 中定义的所有必需依赖项，从而设置您的本地 FHEVM Hardhat 开发环境。 {% endstep %}

{% step %}

### 设置 Hardhat 配置变量（可选）

如果您计划部署到 Sepolia 以太坊测试网，则需要设置以下 [Hardhat 配置变量](https://hardhat.org/hardhat-runner/docs/guides/configuration-variables)。

`MNEMONIC`

助记词是用于生成您的以太坊钱包密钥的 12 个单词的种子短语。

1. 通过使用 [MetaMask](https://metamask.io/) 创建钱包或使用任何受信任的助记词生成器来获取一个。
2. 在您的 Hardhat 项目中进行设置：

```sh
npx hardhat vars set MNEMONIC
```

`INFURA_API_KEY`

INFURA 项目密钥允许您连接到像 Sepolia 这样的以太坊测试网。

1. 按照 [Infura + MetaMask](https://docs.metamask.io/services/get-started/infura/) 设置指南获取一个。
2. 在您的项目中进行配置：

```sh
npx hardhat vars set INFURA_API_KEY
```

#### 默认值

如果您跳过此步骤，Hardhat 将回退到这些默认值：

- `MNEMONIC` = "test test test test test test test test test test test junk"
- `INFURA_API_KEY` = "zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz"

{% hint style="warning" %}
这些默认值不适用于实际部署。
{% endhint %}

{% hint style="warning" %}

### 缺少变量错误：

如果缺少任何请求的 Hardhat 配置变量，您将收到如下错误消息：`Error HH1201: Cannot find a value for the configuration variable 'MNEMONIC'. Use 'npx hardhat vars set MNEMONIC' to set it or 'npx hardhat var setup' to list all the configuration variables used by this project.`

{% endhint %} {% endstep %} {% endstepper %}

恭喜！您已准备好开始构建您的机密 dApp。

## 可选设置

### 安装 VSCode 扩展

如果您正在使用 Visual Studio Code，可以使用一些扩展来改善您的开发体验：

- [Prettier - Code formatter by prettier.io](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode) — ID:`esbenp.prettier-vscode`,
- [ESLint by Microsoft](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint) — ID:`dbaeumer.vscode-eslint`

Solidity 支持（仅选择一个）：

- [Solidity by Juan Blanco](https://marketplace.visualstudio.com/items?itemName=JuanBlanco.solidity) — ID:`juanblanco.solidity`
- [Solidity by Nomic Foundation](https://marketplace.visualstudio.com/items?itemName=NomicFoundation.hardhat-solidity) — ID:`nomicfoundation.hardhat-solidity`

### 重置 Hardhat 项目

如果您想从头开始，可以通过删除所有示例代码和生成的文件来重置您的 FHEVM Hardhat 项目。

```sh
# 导航到您的新 FHEVM Hardhat 项目的根目录
cd <your-new-repo-name>
```

然后运行：

```sh
rm -rf test/* src/* tasks/* deploy ./fhevmTemp ./artifacts ./cache ./coverage ./types ./coverage.json ./dist
```
