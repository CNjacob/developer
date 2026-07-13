# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an **iOS development knowledge base** — a collection of markdown documents covering iOS底层原理 and architecture. There is no source code, build system, or test suite in this repository.

## Document Structure

| File | Topic |
|------|-------|
| `UIKit-UIView-Layer-Rendering.md` | UIView/CALayer 渲染管线、三棵树模型、响应者链、常用组件 |
| `iOS-Runtime-Deep-Dive.md` | Runtime 启动加载、类/元类结构、isa 走位、消息发送三阶段、动态特性 |
| `iOS-RunLoop-Deep-Dive.md` | RunLoop 对象模型、状态机、用户态/内核态切换、触摸事件链路 |
| `iOS-Architecture-Design.md` | MVC/MVVM/响应式框架、UIKit vs SwiftUI 架构对比、Coordinator 模式 |
| `SwiftUI-UIKit-Comparison.md` | SwiftUI 定位、与 UIKit 的关系、编程模型差异、优劣对比、混编和选型 |
| `Objective-C-Underlying-Principles.md` | ObjC 对象系统、SEL/IMP、Block 底层、KVC/KVO、ARC/weak 实现 |
| `Swift-Underlying-Principles.md` | Swift 编译流水线、类型系统、方法调度、泛型单态化、Concurrency |
| `iOS-Audio-Video-Development.md` | 音视频基础、AVFoundation 架构、音视频采集/播放/编解码、实时通信、渲染优化 |
| `iOS-Security-Mechanisms.md` | Keychain、Data Protection、Sandbox、ATS、代码签名、混淆、反调试、安全落地方案 |
| `iOS-Performance-Optimization.md` | 启动优化、包体积、内存、UI渲染、流畅度、CPU、I/O、网络与性能监控 |
| `JavaScript-TypeScript-Knowledge.md` | JavaScript 运行模型、闭包、原型、异步、模块化与 TypeScript 类型系统、严格配置 |
| `React-Native-Knowledge.md` | React/RN 渲染模型、新架构、原生模块、状态管理、性能、工程化与混合架构 |
| `Dart-Language-Knowledge.md` | Dart 类型系统、空安全、面向对象、Records、Patterns、异步、Isolate、Package 与测试 |
| `Flutter-Development-Architecture.md` | Flutter 渲染、状态、分层架构、原生通信、Add-to-App、模块化、性能与测试 |

## Guidelines for Adding New Content

- Create new `.md` files at the repository root with descriptive kebab-case names
- Use ASCII text diagrams for flows and architecture (no image dependencies)
- Follow the existing pattern: overview → detailed sections with diagrams → summary table at the end
- If extending an existing topic, prefer editing the existing file rather than creating a new one
