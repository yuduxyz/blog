Visual Studio Code（VS Code）是由微软维护的一个开源项目，通过 Electron 打包运行。核心代码在 [microsoft/monaco-editor](https://github.com/Microsoft/monaco-editor) 中，这是一个运行在浏览器上的编辑器项目。

通过切换开发人员工具，可以看到，VS Code 实际上就是一个运行在浏览器中的应用:

![image](https://user-images.githubusercontent.com/9818716/62685520-668aa880-b9f5-11e9-913a-50db41fb79e4.png)

更多和 VS Code 相关的趣事可以在[维护 VS Code 开源项目背后的那些事情](https://www.zhihu.com/question/36292298/answer/160028010)一文中看到。

下面介绍如何在 VS Code 中开发一款快速拷贝模版文件的插件。

## 环境准备

安装 [Yeoman](https://yeoman.io/) 和 [VS Code Extension Generator](https://github.com/Microsoft/vscode-generator-code)：

```
npm install -g yo generator-code
```

其中，Yeoman 是一款通用的脚手架生成器，可以用于搭建任意语言的项目（JavaScript、Java、Python、C# 等等）。

用于构建 JavasScript Web 应用时，Yeoman 工作流由三部分组成——脚手架工具 Yo、构建工具 Gulp / Grunt 等、包管理器 npm / Bower。

Yeoman 的典型使用场景有：
- 快速创建新项目
- 创建项目中的一个新部分，如一个带有单元测试的 controller
- 创建新的模块或者包
- 启动新的服务
- 保证编码规范、最佳实践和代码风格
- ...

VS Code Extension Generator 就是微软通过 Yeoman 搭建的一个插件项目生成器。

## 从脚手架生成初始项目

运行 `yo code`，即可通过插件项目生成器创建一个新的项目：

![yo code](https://user-images.githubusercontent.com/9818716/62768739-1af5fe80-baca-11e9-8f36-6b6c091762d7.png)

这里，我们选择使用 TypeScript 创建一个新的插件。

![](https://user-images.githubusercontent.com/9818716/62695621-4b299880-ba09-11e9-83af-fe63ed0a7ffa.png)

填写相关配置项后，将会创建模版文件，并自动安装依赖。

## 开发

在生成的项目目录下打开 VS Code，按下 `F5`，将会在一个新的扩展开发宿主（Extension Host）窗口中运行插件。

在命令面板（win: `Ctrl+Shift+p`, mac: `Command+p`）中输入 `Hello World` 命令，将会出现 `Hello World` 弹窗。

![](https://user-images.githubusercontent.com/9818716/62702572-aebbc200-ba19-11e9-86ca-0420a6160ae6.png)

这样，我们成功的创建了一个简单的 VS Code 插件，它在输入 `Hello World` 命令后会出现 `Hello World` 消息弹窗。

项目的目录结构为：

```
.
├── .vscode
│   ├── launch.json     // 插件加载和调试的配置
│   └── tasks.json      // 配置TypeScript编译任务
├── README.md           // 一个友好的插件文档
├── src
│   └── extension.ts    // 插件源代码
├── package.json        // 插件配置清单
├── tsconfig.json       // TypeScript配置
```

如果要进一步开发，主要需要关注两个文件：

### `extension.ts`

这是插件的入口文件。

```javascript
// 'vscode' 模块包含了 VS Code extensibility API
import * as vscode from 'vscode';

// 一旦插件激活，vscode 会立刻调用下述方法
export function activate(context: vscode.ExtensionContext) {

    // 下面的代码只会在插件激活时执行一次
    console.log('Congratulations, your extension "hello" is now active!');

    // 入口命令在 package.json 文件中定义，调用 registerCommand 方法注册命令的回调
    // registerCommand 中的参数必须与 package.json 中的 command 保持一致
    let disposable = vscode.commands.registerCommand('extension.sayHello', () => {
        // 每次命令执行时都会调用这里的代码
        // ...
        // 给用户显示一个消息提示
        vscode.window.showInformationMessage('Hello World!');
    });

    context.subscriptions.push(disposable);
}
```

插件的开发主要就是在 `package.json` 定义启动命令，然后在入口文件中对命令注册响应事件。

通过 VS Code 提供的 API，我们可以实现一系列功能，具体可以参考[文档](https://liiked.github.io/VS-Code-Extension-Doc-ZH/#/extension-capabilities/README)。主要包括：
- 核心功能，如注册命令、配置、快捷键绑定等等
- 主题设置
- 声明式添加语言特性
- 编程式添加语言特性
- 扩展控制台
- 调试功能

我们下面将实现的插件就是要扩展控制台功能，自定义资源管理器侧边栏的菜单行为。

### `package.json`

插件的命令在 `package.json` 的 `contributes` 字段中定义：

```
"contributes": {
  "commands": [{
    "command": "extension.helloWorld",
    "title": "Hello World"
  }]
},
```

`extension.helloWorld` 中的 `extension` 是命名空间，也可以替换成其他名字。title 中的命名最终会出现在命令选择面板中。

此外，对于通过命令面板调用的命令，还需要在 `package.json` 中定义好激活事件：

```
"activationEvents": [
  "onCommand:extension.helloWorld"
]
```

当用户在命令面板中第一次调用 `extension.helloWorld` 时，插件就会自动激活，`extension.ts` 中的 `registerCommand` 会将 `extension.helloWorld` 绑定到正确的处理函数上。

所以，要想通过命令启动一个插件，需要做好三件事：

1. 在 `package.json` 中定义命令 `commands`
2. 在 `package.json` 中定义激活事件 `activationEvents`
3. 在 `extension.ts` 中注册事件处理函数 `registerCommand`

## 调试

其实就是 VS Code 内置调试功能的使用，在代码序号左侧点击，设下断点。按 `F5` 进入调试模式后，即可开始断点调试：

![debug](https://user-images.githubusercontent.com/9818716/62768369-488e7800-bac9-11e9-9e4f-edfe0a62df5c.png)

## 实现模版文件的拷贝



## 发布