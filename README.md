# 前言
本文记录了使用[mozilla/source-map库](https://github.com/mozilla/source-map)来定位打包后的js代码错误，至于sourcemap文件的原理在此不再赘述，网上已经有很多很好的教程文章，比如阮一峰老师的[JavaScript Source Map 详解](https://www.ruanyifeng.com/blog/2013/01/javascript_source_map.html)与joeyguo的[脚本错误量极致优化-让脚本错误一目了然](https://github.com/joeyguo/blog/issues/14)。

本文让我们在实践中加深对sourceMap的理解吧！（建议阅读以上两篇文章后再开始实践）

流程非常简单，如下：
1. demo准备，并生成sourceMap
2. 利用[mozilla/source-map库](https://github.com/mozilla/source-map)处理指出**出错源文件、出错源文件行列数、原变量名**

demo都在如下GitHub地址：[Joeoeoe/source-map-demo](https://github.com/Joeoeoe/source-map-demo)

# demo准备，并生成sourceMap
配置一个简单的webpack:
```js
//webpack.config.js
const path = require("path");

module.exports = {
  devtool: "source-map",
  entry: "./src/index.js",
  output: {
    filename: "main.js",
    path: path.resolve(__dirname, "dist"),
  },
};
```

在**src**文件夹下创建**index.js**与**moudleA.js**
```js
// moduleA.js
const moudleA = "moduleA";

module.exports = moudleA;
```
```js
// index.js
const moudleA = require("./moduleA");
console.log(moudleA);

if (true) {
  console.log(a); // 模拟的错误
}
```
接下来用webpack打包，因为我们的webpack中配置了**devtool: "source-map"**，所以会生成**main.js**与**main.js.map**。
```js
// main.js，一串压缩后的代码，末尾指向对应的sourceMap
!function(e){var t={};function n(r){if(t[r])return t[r].exports;var o=t[r]={i:r,l:!1,exports:{}};......
//# sourceMappingURL=main.js.map
```

```js
// main.js.map sourceMap内容，让我们把他格式化出来看看吧
{"version":3,"sources":["webpack:///webpack/bootstrap","webpack:///./src/moduleA.js","webpack:///./src/index.js"]....
```
利用[json格式化工具](https://www.json.cn/)，看看sourceMap的内容：

![](https://blog-1256056666.cos.ap-guangzhou.myqcloud.com/img/20201011111116.png)

**sources**数组下是转换前的文件，其内容与**sourcesContent**下标一一对应。通过查看**sourcesContent**，可以发现**webpack/bootstrap**即webpack在浏览器端对require的模拟。其他属性在阮一峰老师的文章里有很详细的解释，此处就不再赘述了。

demo准备完后接下来可以根据sourceMap定位错误了。
# 利用[mozilla/source-map库](https://github.com/mozilla/source-map)定位错误
项目中利用sourceMap定位错误的方案一般如下（图自[脚本错误量极致优化-让脚本错误一目了然](https://github.com/joeyguo/blog/issues/14)）：
![](https://blog-1256056666.cos.ap-guangzhou.myqcloud.com/img/20201011135901.png)

为了模拟外网环境出错，把**main.js**中的**//# sourceMappingURL=main.js.map**去掉，在dist/index.html中添加错误监听:
```js
    <script>
      window.onerror = function (msg, url, row, col, error) {
        const obj = {
          msg,
          url,
          row,
          col,
        };
        console.log(obj);
      };
    </script>
```
打开html可看见：
![](https://blog-1256056666.cos.ap-guangzhou.myqcloud.com/img/20201011152125.png)

有了**row**与**col**之后接下来使用[mozilla/source-map库](https://github.com/mozilla/source-map)获取错误在源文件的位置。

**npm i source-map**安装source-map库，在项目下新建**trySourceMap.js**文件：
```js
const fs = require("fs");
const sourceMap = require("source-map"); // mozilla/source-map库
const rawSourceMap = JSON.parse(
  // 打包后的sourceMap文件
  fs.readFileSync("./dist/main.js.map").toString()
);

const errorPos = {
  // 上图中的错误位置
  line: 1,
  column: 946,
};

async function main() {
  const consumer = await new sourceMap.SourceMapConsumer(rawSourceMap); // 获取sourceMap consumer，我们可以通过传入打包后的代码位置来查询源代码的位置

  const originalPosition = consumer.originalPositionFor({ // 获取 出错代码 在 哪一个源文件及其对应位置
    line: errorPos.line,
    column: errorPos.column,
  });
  // { source: 'webpack:///src/index.js', line: 4, column: 14, name: 'a' }

  // 根据源文件名寻找对应源文件
  const sourceIndex = consumer.sources.findIndex(
    (item) => item === originalPosition.source
  );
  const sourceContent = consumer.sourcesContent[sourceIndex];
  const contentRowArr = sourceContent.split("\n"); //切分
  // [
  //  'const moudleA = require("./moduleA");\r',
  //  '\r',
  //  'if (true) {\r',
  //  '  console.log(a);\r',
  //  '}\r',
  //  ''
  // ]

  // 接下来根据行和列可获取更加具体的位置
  console.log(contentRowArr[originalPosition.line - 1]);

  consumer.destroy(); // 使用完后记得destroy
}

main();
```

![](https://blog-1256056666.cos.ap-guangzhou.myqcloud.com/img/20201011153646.png)

如此便捕捉到了出错代码在源文件的位置。

# 结语
文本记录了如何利用sourceMap定位线上代码的错误，完整demo均在此GitHub地址：[Joeoeoe/source-map-demo](https://github.com/Joeoeoe/source-map-demo)
# 参考资料
[JavaScript Source Map 详解——阮一峰](https://www.ruanyifeng.com/blog/2013/01/javascript_source_map.html)  
[Introduction to JavaScript Source Maps](https://www.html5rocks.com/en/tutorials/developertools/sourcemaps/)  
[脚本错误量极致优化-让脚本错误一目了然](https://github.com/joeyguo/blog/issues/14)  
[mozilla/source-map——Github](https://github.com/mozilla/source-map)  
[webpack是如何实现前端模块化的](https://juejin.im/post/6844903779817488392)  
