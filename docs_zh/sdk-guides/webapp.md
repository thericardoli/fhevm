# 构建 Web 应用程序

本文档将指导您使用 `@zama-fhe/relayer-sdk` 库构建一个 Web 应用程序。

<!-- 注意：一旦模板更新到最新的测试网，请取消注释 -->

<!-- 您可以从模板开始，也可以直接将库集成到您的项目中。 -->
<!-- ## 使用模板 -->
<!---->
<!-- `@zama-fhe/relayer-sdk` 开箱即用，我们建议您使用它。我们还提供了三个 GitHub 模板，以便您在项目开始时就完成所有设置。 -->
<!---->
<!-- ### React + TypeScript -->
<!---->
<!-- 您可以使用[此模板](https://github.com/zama-ai/fhevmjs-react-template)来启动一个使用 Vite + React + TypeScript 的 @zama-fhe/relayer-sdk 应用程序。 -->
<!---->
<!-- ### NextJS + Typescript -->
<!---->
<!-- 您也可以使用[此模板](https://github.com/zama-ai/fhevmjs-next-template)来启动一个使用 Next + TypeScript 的 @zama-fhe/relayer-sdk 应用程序。 -->
<!---->
<!-- ## 使用模拟协处理器进行前端开发 -->
<!---->
<!-- 作为使用部署在 Sepolia 上的真实协处理器的替代方案，为了帮助您更快地开发 dApp 并且不需要测试网代币，您可以使用模拟的 FHEVM。目前，我们建议您使用 [React 模板](https://github.com/zama-ai/fhevm-react-template/tree/mockedFrontend) 的 `mockedFrontend` 分支上可用的 `ConfidentialERC20` dApp 示例。请按照此分支上的 README 操作，您将能够无缝地在 Sepolia 和模拟协处理器上部署完全相同的 dApp。 -->
<!---->

## 直接使用库

### 第 1 步：设置库

`@zama-fhe/relayer-sdk` 包含多个文件，包括 WASM 文件和 WebWorker，这可能会使在您的设置中正确打包这些组件变得繁琐。为了简化此过程，特别是如果您正在开发具有服务器端渲染 (SSR) 的 dApp，我们建议您使用我们的 CDN。

#### 使用 UMD CDN

在您的项目顶部包含此行。

```html
<script src="https://cdn.zama.ai/relayer-sdk-js/0.2.0/relayer-sdk-js.umd.cjs" type="text/javascript"></script>
```

在您的项目中，如果您安装了 `@zama-fhe/relayer-sdk` 包，则可以使用捆绑包导入：

```javascript
import { initSDK, createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/bundle";
```

#### 使用 ESM CDN

如果您愿意，也可以将 `@zama-fhe/relayer-sdk` 作为 ES 模块使用：

```html
<script type="module">
  import { initSDK, createInstance, SepoliaConfig } from "https://cdn.zama.ai/relayer-sdk-js/0.2.0/relayer-sdk-js.js";

  await initSDK();
  const config = { ...SepoliaConfig, network: window.ethereum };
  config.network = window.ethereum;
  const instance = await createInstance(config);
</script>
```

#### 使用 npm 包

将 `@zama-fhe/relayer-sdk` 库安装到您的项目中：

```bash
# 使用 npm
npm install @zama-fhe/relayer-sdk

# 使用 Yarn
yarn add @zama-fhe/relayer-sdk

# 使用 pnpm
pnpm add @zama-fhe/relayer-sdk
```

`@zama-fhe/relayer-sdk` 使用 ESM 格式。您需要在 package.json 中将 [type 设置为 "module"](https://nodejs.org/api/packages.html#type)。如果您的节点项目使用 `"type": "commonjs"` 或没有类型，您可以通过使用 `import { createInstance } from '@zama-fhe/relayer-sdk/web';` 来强制加载 Web 版本。

```javascript
import { initSDK, createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk";
```

### 第 2 步：初始化您的项目

要在您的项目中使用该库，您需要首先使用 `initSDK` 加载 [TFHE](https://www.npmjs.com/package/tfhe) 的 WASM。

```javascript
import { initSDK } from "@zama-fhe/relayer-sdk/bundle";

const init = async () => {
  await initSDK(); // 加载所需的 WASM
};
```

### 第 3 步：创建一个实例

加载 WASM 后，您现在可以创建一个实例。

```javascript
import { initSDK, createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/bundle";

const init = async () => {
  await initSDK(); // 加载 FHE
  const config = { ...SepoliaConfig, network: window.ethereum };
  return createInstance(config);
};

init().then((instance) => {
  console.log(instance);
});
```

您现在可以使用您的实例来[加密参数](./input.md)、执行[用户解密](./user-decryption.md)或[公共解密](./public-decryption.md)。
