---
title: categories
date: 2021-05-26 16:51:07
categories:
  - 研发流程
tags:
  - devops
---

## 概述

Sonar 免费的社区版本不支持 Obj-C /Swift  

sonar -scanner 支持导入 Infer、SwiftLint、Lizard 生成的报告

### 静态分析的三大工具

Clang、Infer 和 OCLint

开源项目基于这 3 个工具做增强

### 开源项目

最开始是 [octo-technology/sonar-objective-c](https://github.com/octo-technology/sonar-objective-c) ，在 2015年不再维护。

[Backelite/sonar-objective-c](https://github.com/Backelite/sonar-objective-c) fork 过来维护到 2018 年，另起新项目[Idean/sonar-swift](https://github.com/Idean/sonar-swift)

[tal-tech/sonar-swift](https://github.com/tal-tech/sonar-swift) 基于[Idean/sonar-swift](https://github.com/Idean/sonar-swift) ，增加了 facebook infer 的结果导入，及项目中文化

后者：

- 下载 jar 包放到 SonarQube 的`extensions/plugins` 目录
- 下载 `run-sonar-swift.sh`
- 在待分析项目的目录下运行 `run-sonar-swift.sh`，会调用 `xcodebuild`，所以需要在 Mac 下运行



所需流程如下：

- 执行 xcodebuild，并将 log 记录输出到文件
- 使用 xcpretty 将log日志输出为 json 格式
- 执行 infer 分析，配置并忽略第三方的代码目录，导入刚刚的 json 编译日志
- 执行 swiftlint ，并将结果输出到文件（如果有用到 swift 语言。但此工具只是代码格式的检查，不含语法或逻辑检查）
- 使用 lizard 以 xml 格式输出
  执行 SonarScanner ，并填写报告路径



## 软件介绍

### Infer

Infer 是由Facebook公司推出的静态代码扫描工具，支持 C/C++/Java 语言的扫描。

链接：https://github.com/facebook/infer

Infer 调用 gcc/clang/javac 等编译命令，捕捉项目编译的日志，见 https://fbinfer.com/docs/analyzing-apps-or-projects

所以同样需要 `xcodebuild ` 



### Darling

类似 wine，Darling 让 macOS 软件运行在 Linux 上。安装 XCode？

https://www.darlinghq.org/



### SwiftLint

Github 提供安装包，MacOS 的 pkg 和 Linux 的，解压即可运行  

```
# 安装 swift 编译器
dnf install swift-lang
# 按 GitHub 说明，指定 libsourcekitdInProc.so 所在路径
export LINUX_SOURCEKIT_LIB_PATH=/usr/libexec/swift/lib
```

编写 main.swift

```
import Foundation

var a=1/0
```

使用 SwiftLint 分析

```
swiftlint lint
```

结论：SwiftLint 只管代码风格，并不检查语法错误（除以0）



### OCLint

工作流程：

- 项目编译

- xcpretty 工具处理生成 compile_commands.json。里面列出每个源文件编译时所用的命令
- oclint-json-compilation-database 基于 compile_commands.json，对每个源文件执行的静态代码分析



## 实操步骤

检查 SDK 是否正常
```
xcodebuild -showsdks
# 如果提示错误，要指定 xcode 的位置
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer/
```

安装 xcpretty

```
sudo gem install xcpretty
```


 安装 infer
```
brew install infer
```


注意事项

- 使用本机的 apple id 编译（涉及到保存在本机的 keychain）
- 要修改 project.pbxproj 里的所有 DEVELOPMENT_TEAM 和 PRODUCT_BUNDLE_IDENTIFIER 对应的值（可以先在 XCode 操作一遍，获取 DEVELOPMENT_TEAM 即团队 ID。后者 bundle id 可以加个随机数）
- 使用本机终端执行一次下面的 xcodebuild ，会弹框允许访问登录钥匙串（输入账号的登录密码确认），选择“总是允许”。或者打开系统的`钥匙串访问`，左边点击`登录`，右边展开`Apple Development: APPLE_ID`，双击`专用密钥`，弹框切换到`访问控制 `，勾选`允许所有应用访问此项目`
- 使用远程shell登录Mac时，默认是没有账户的，但是访问钥匙串要求必须有用户身份，所以 build 步骤前添加一步解锁钥匙串

下面编译

```
# 如果是 shell 远程登录运行，需要解锁钥匙串
security unlock-keychain

# 使用 xcodebuild 编译，签名出错可忽略？
xcodebuild clean build -workspace InteractiveBroadcast.xcworkspace -scheme InteractiveBroadcast -destination 'generic/platform=iOS' COMPILER_INDEX_STORE_ENABLE=NO | tee xcodebuild.log

# 使用 xcpretty 整理 xcodebuild 的日志
LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8 xcpretty -r json-compilation-database -o compile_commands.json < xcodebuild.log > /dev/null

# 使用 infer 从 compile_commands.json 分析，--skip-analysis-in-path 是忽略扫描目录
infer run --skip-analysis-in-path Pods --compilation-database compile_commands.json

# 结果导入到 sonarqube
sonar-scanner -Dsonar.host.url=http://192.168.1.193:9000 -Dsonar.projectKey=Anxin_IAC_Client/ios-client -Dsonar.sources=. -Dsonar.swift.infer.report=infer-out/report.json
```



### OCLint 安装及单独使用

```
brew install oclint

# // 以下2步与 Infer 一样的
# 使用 xcodebuild 编译，签名出错可忽略？
xcodebuild clean build -workspace InteractiveBroadcast.xcworkspace -scheme InteractiveBroadcast -destination 'generic/platform=iOS' COMPILER_INDEX_STORE_ENABLE=NO | tee xcodebuild.log

# oclint-xcodebuild 已不维护，官方同样推荐使用 xcpretty 整理日志
# 使用 xcpretty 整理 xcodebuild 的日志
xcpretty -r json-compilation-database -o compile_commands.json < xcodebuild.log > /dev/null

# OCLint 分析，结果导出为 report.html
oclint-json-compilation-database -e Pods -- -report-type html -o report.html -rc LONG_LINE=9999 -max-priority-1=9999 -max-priority-2=9999 -max-priority-3=9999
```



## 其它

```
gem list
sudo gem install -n /usr/local/bin cocoapods
rm -fr Podfile.lock Pods
pod install

```

