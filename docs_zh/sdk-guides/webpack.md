# 常见的 webpack 错误

本文档提供了开发过程中遇到的常见 Webpack 错误的解决方案。请按照以下步骤解决每个问题。

## 无法解析 'tfhe_bg.wasm'

**错误消息：** `Module not found: Error: Can't resolve 'tfhe_bg.wasm'`

**原因：** 在代码库中，有一个 `new URL('tfhe_bg.wasm')`，它会触发 Webpack 的解析。

**可能的解决方案：** 您可以通过在 `webpack.config.js` 中添加一个解析配置来为此文件添加一个回退：

```javascript
resolve: {
  fallback: {
    'tfhe_bg.wasm': require.resolve('tfhe/tfhe_bg.wasm'),
  },
},
```

## Buffer 未定义

**错误消息：** `ReferenceError: Buffer is not defined`

**原因：** 当在浏览器环境中使用 Node.js `Buffer` 对象时，会出现此错误，因为它在浏览器中不是本机可用的。

**可能的解决方案：** 要解决此问题，您需要为 Node.js 核心模块提供浏览器兼容的回退。安装必要的 browserified npm 包并配置 Webpack 以使用这些回退。

```javascript
resolve: {
  fallback: {
    buffer: require.resolve('buffer/'),
    crypto: require.resolve('crypto-browserify'),
    stream: require.resolve('stream-browserify'),
    path: require.resolve('path-browserify'),
  },
},
```

## 导入 ESM 版本时出现问题

**错误消息：** 导入 ESM 版本时出现问题

**原因：** 对于像 Webpack 或 Rollup 这样的打包器，导入将被替换为 `package.json` 的 `"browser"` 字段中提到的版本。这可能会导致类型问题。

**可能的解决方案：**

- 如果您遇到类型问题，可以使用 TypeScript 5 的这个 [tsconfig.json](https://github.com/zama-ai/fhevm-react-template/blob/main/tsconfig.json)。
- 如果您遇到任何其他问题，可以强制导入浏览器包。

## 使用捆绑版本

**错误消息：** 捆绑库时出现问题，尤其是在 SSR 框架中。

**原因：** 库可能无法与某些框架正确捆绑，从而导致在构建或运行时过程中出错。

**可能的解决方案：** 使用 `@zama-fhe/relayer-sdk/bundle` 提供的[预捆绑版本](./webapp.md)。使用 `<script>` 标签嵌入库并如下所示进行初始化：

```javascript
const start = async () => {
  await window.fhevm.initSDK(); // 加载所需的 wasm
  const config = { ...SepoliaConfig, network: window.ethereum };
  config.network = window.ethereum;
  const instance = window.fhevm.createInstance(config).then((instance) => {
    console.log(instance);
  });
};
```
