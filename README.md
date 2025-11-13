# flutter_shorebird_demo

flutter热修框架shorebird的测试



## 文档

接入流程:

https://docs.shorebird.dev/code-push/initialize/

管理后台:

https://console.shorebird.dev/orgs/36172/apps

版本限制:

https://docs.shorebird.dev/getting-started/flutter-version/

中文博客:

https://zhuanlan.zhihu.com/p/27590130553

https://blog.csdn.net/wang_yong_hui_1234/article/details/137987794



## 步骤

1. Sign up

选择Microsoft注册

2. Install

```
curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash
```

3. 检测

```
shorebird doctor
```

4. 登录

```
shorebird login
```

5. 进入已存在的flutter项目

```
cd my_shorebird_app
```

6. 初始化

```
shorebird init
```

7. 打包

> 可指定flutter版本，应用的版本名、版本号
>
> 例：版本号：1.0.1+3

```
shorebird release android --flutter-version 3.35.1 --artifact apk -v -- --build-name=1.0.1 --build-number=3
```

8. 预览已上传的包

```
shorebird preview
```

9. 打patch

> 此时可以修改代码，然后打热修补丁

```
shorebird patch android
```

成功后，app下次启动即可看到更新，目前手机需连代理才能正常下载



## deepseek对shorebird的专业解释

### 苹果审核机制

- **“Native 代码”指什么**：通常指那些**编译后直接在本机操作系统上运行的代码**（如用 Objective-C、Swift 编写的部分）。这些代码决定了应用的核心功能和行为。
- **“不能修改”的真实含义**：苹果并非禁止所有更新，而是禁止**在应用通过审核上架后，通过服务器动态下载并执行能改变应用原始目的或核心行为的代码**（即可执行代码）。这确保了用户设备上运行的 App，其核心功能是经过苹果审核的版本。



苹果官方开发者协议的3.3.2节明确规定，应用程序**禁止下载或安装任何可执行代码** 。作为例外，允许使用**解释型代码**（如JavaScript），但前提是**所有脚本、代码和解释器都必须打包在应用内，不能下载**。唯一的特例是使用苹果内置的 WebKit 框架或 JavascriptCore 下载和运行的脚本，但这些脚本**不能通过提供与提交到 App Store 的预期和宣传用途不一致的特性或功能，来改变应用程序的主要用途** 。



主要限制和允许的例外情况：

| **苹果的态度**   | **情况**                       | **关键点/目的**                                              |
| :--------------- | :----------------------------- | :----------------------------------------------------------- |
| ❌ **明确禁止**   | **禁止：下载可执行代码**       | 防止未经审核改变App核心功能、规避安全与隐私审查。            |
| ⚠️ **有条件允许** | **限制：下载解释型代码(如JS)** | 代码**不能**改变App**主要用途** (如将阅读类App突然改成游戏)；通常需通过**WebKit**等系统框架运行 。 |
| ✅ **允许**       | **允许：内置脚本解释器**       | 所有脚本、代码和解释器都必须**打包在最初提交审核的App内** 。 |
| ❌ **明令禁止**   | **常见违规案例**               | 使用**JSPatch**、**[Rollout.io](https://rollout.io/)**等能修改原生方法、修复Bug的热更新框架 。 |



### shorebird

Shorebird能够确保不更新Native代码，核心在于它的技术方案将更新范围**精准地限制在了Dart代码层面**，并且针对不同移动平台的政策，采用了不同的合规技术实现。



在两个平台上的核心机制：

| 维度       | Android 平台                                                    | iOS 平台                                                      |
|:---------|:--------------------------------------------------------------|:------------------------------------------------------------|
| **技术原理** | 下发二进制的`dlc.vmcode`补丁文件，Dart VM在运行时直接加载，实现**AOT（预编译）**方式的代码替换。 | 通过定制化的Dart VM解释器来运行更改的Dart代码，未更改的部分依然以AOT模式运行。              |
| **核心机制** | **二进制替换**                                                     | **解释执行**                                                    |
| **性能表现** | 由于全程是AOT执行，**性能几乎没有损耗**。                                      | 未更改的代码保持AOT高性能，**更改的代码因解释执行，会有一定的性能损耗**。                    |
| **合规依据** | Google Play政策允许在虚拟机中运行的代码进行更新。                                | App Store禁止下载任何可执行代码。Shorebird通过不下发机器码，并将补丁代码交由解释器执行来规避此限制。 |



### 深入理解Shorebird的工作方式

Shorebird如何实现“不碰Native代码”，以下几个关键点：

1. **构建阶段的分叉与嵌入**：当你使用 `shorebird build` 命令构建应用时，Shorebird实际上使用了一个自己**分叉（Fork）并定制过的Flutter引擎**。这个引擎内嵌了负责更新检查、补丁下载与验证的关键组件，为后续的热更新打下了基础。
2. **精准的更新范围限制**：Shorebird的设计初衷就是只更新Dart代码。这意味着：
   - **无法修改原生代码（Objective-C/Swift, Kotlin/Java）**：任何对原生插件或平台自身代码的修改，Shorebird都无法通过热更新实现。
   - **无法更新或添加插件**：因为插件的增减或变更涉及到了原生代码，所以也必须通过应用商店发布新版本。
3. **补丁的生成与应用**：当你修改Dart代码后，使用 `shorebird patch` 命令，Shorebird会将你的新代码与线上版本进行对比，生成一个最小差异的二进制补丁文件（在Android上就是`.dlc.vmcode`文件）。应用在启动时，Shorebird的运行时加载器会指导Dart VM去加载这个补丁，从而让新代码生效。
