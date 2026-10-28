# 某校园跑步类 Android 应用的逆向分析

> 一个经过商用加固的校园跑步类 App：代码的存在形态、Native 层密钥管理、自研双签名协议设计，以及 12 条服务端加固建议。

![platform](https://img.shields.io/badge/platform-Android%20%7C%20arm64--v8a-blue)
![focus](https://img.shields.io/badge/focus-app%20hardening%20%2B%20server--side%20trust-informational)
[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

---

## 目录

```
REPORT.md                 主报告（建议从这里开始）
docs/architecture.md      分层与请求链路
LICENSE                   CC BY-NC-SA 4.0
```

## 概要

样本是一个国内校园跑步与打卡类应用（7.x 系列，arm64，数百 MB）。它用了商用加固，业务逻辑只在运行时内存中存在。这份笔记记录的是**加固到底改变了什么、密钥被放在了哪里、网络层的签名协议怎么设计的**，以及从防守视角出发、不依赖修改客户端的 12 条建议。

核心判断只有一句：

> **这类应用的安全性不取决于客户端把代码和密钥藏得多深，而取决于服务端愿不愿意对客户端上报的内容做独立复核。**

## 结论速览

| 观察点 | 结论 |
|---|---|
| 静态反编译的产出 | 只有少量壳类，业务代码不可见 |
| 业务代码的存在形态 | 存在于进程内存，磁盘不留完整副本 |
| 敏感常量的存放方式 | 静态为密文，运行时在内存中被还原 |
| 加密层的实现归属 | 主要交给第三方商用安全 SDK，自研部分集中在签名 |
| 签名方案 | 两级：会话级（头）与业务体级（体），均为加盐摘要 |
| 传输保护 | 请求与响应共用一种不透明的信封结构，客户端无法解开自己发出的信封 |
| 最大的设计隐患 | 一处配置开关能让加密层静默退化为明文 |
| 最有效的防线 | 服务端对内容做交叉校验（见报告第 5 节） |

## 三点普遍规律

1. **加固产品对"被观察"的敏感度，高于"被读取"** —— 防线常落在"是否有人尝试接近"，而不是"被拿走了多少"。
2. **完整性校验需要双向确认** —— "看起来完整"与"能被格式自身的冗余字段验证通过"是两件事，只有后者构成完整性。
3. **混合架构下，JVM 侧工具只是起点** —— 跨平台层的逻辑不在这条工具链上，"反编译完了"往往刚开始。

## English abstract

Notes on a hardened Android app from China's campus-running vertical (arm64, commercial packer, hybrid native + cross-platform UI).

The write-up covers what the packer structurally changes, why moving secrets into a native library only relocates the trust boundary, and how a self-built "dual signature + opaque envelope" protocol is structured — followed by **12 server-side hardening recommendations** that require no client changes.

The through-line: *client-side obfuscation buys cost, not trust; trust comes from the server re-deriving what the client claims.*

## 引用

```
@misc{campus-run-app-re-notes,
  title  = {某校园跑步类 Android 应用的逆向分析},
  year   = {2026},
  note   = {代码存在形态 / Native 密钥管理 / 双签名协议 / 服务端加固建议}
}
```

## 免责声明

仅供学习与研究。请勿将其中任何信息用于未经授权的访问、数据获取、成绩篡改，或其他违反法律法规与平台用户协议的行为。若你是该应用的开发者并希望就披露范围沟通，请开 Issue。