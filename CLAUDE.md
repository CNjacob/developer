# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **developer knowledge base** — a collection of markdown documents covering iOS, Android, HarmonyOS, mobile cross-platform development, Python backend development, frontend languages, compiler fundamentals, and development environment setup. There is no source code, build system, or test suite in this repository.

## Document Structure

| Directory | File | Topic |
|-----------|------|-------|
| `Mobile-Development/iOS/` | `UIKit-UIView-Layer-Rendering.md` | UIView/CALayer 渲染管线、三棵树模型、响应者链、常用组件 |
| `Mobile-Development/iOS/` | `iOS-Architecture-Design.md` | MVC/MVVM/响应式框架、UIKit vs SwiftUI 架构对比、Coordinator 模式 |
| `Mobile-Development/iOS/` | `SwiftUI-UIKit-Comparison.md` | SwiftUI 定位、与 UIKit 的关系、编程模型差异、优劣对比、混编和选型 |
| `Mobile-Development/iOS/` | `iOS-Audio-Video-Development.md` | 音视频基础、AVFoundation 架构、音视频采集/播放/编解码、实时通信、渲染优化 |
| `Mobile-Development/iOS/` | `iOS-Security-Mechanisms.md` | Keychain、Data Protection、Sandbox、ATS、代码签名、混淆、反调试、安全落地方案 |
| `Mobile-Development/iOS/` | `iOS-Performance-Optimization.md` | 启动优化、包体积、内存、UI渲染、流畅度、CPU、I/O、网络与性能监控 |
| `Mobile-Development/Android-Development/` | `Android-Environment-Setup.md` | Android Studio、SDK、模拟器、真机调试、Gradle、ADB 与常见环境问题 |
| `Mobile-Development/Android-Development/` | `Android-Core-Knowledge.md` | Android 基础组件、生命周期、UI、网络、存储、权限、架构、测试与发布 |
| `Mobile-Development/Android-Development/` | `Android-Architecture-Patterns.md` | Android 常见架构类型、目录结构、MVC、MVP、MVVM、MVI、Clean Architecture、模块化与示例代码 |
| `Mobile-Development/Android-Development/` | `Android-Java-Language-Knowledge.md` | Android Java 语法、面向对象、集合、泛型、异常、线程、Handler 与常见写法 |
| `Mobile-Development/Android-Development/` | `Android-Kotlin-Language-Knowledge.md` | Android Kotlin 语法、空安全、data class、扩展函数、Lambda、协程、Flow 与常见写法 |
| `Mobile-Development/HarmonyOS-Development/` | `HarmonyOS-Environment-Setup.md` | DevEco Studio、HarmonyOS SDK、Stage 模型项目结构、运行调试、签名与 HAP/HAR/HSP 基础 |
| `Mobile-Development/HarmonyOS-Development/` | `HarmonyOS-Core-Knowledge.md` | HarmonyOS 基础知识、UIAbility、ArkUI、状态管理、路由、权限、网络、存储与组件化 |
| `Mobile-Development/HarmonyOS-Development/` | `HarmonyOS-ArkTS-Language-Knowledge.md` | ArkTS 语法、类型、函数、类、接口、泛型、异步、ArkUI 状态装饰器与页面常用写法 |
| `Mobile-Development/HarmonyOS-Development/` | `HarmonyOS-Architecture-Design.md` | HarmonyOS 架构设计、简单分层、MVVM、MVI、Clean Architecture、HAR/HSP 模块化与示例代码 |
| `Mobile-Development/iOS/` | `iOS-Runtime-Deep-Dive.md` | Runtime 启动加载、类/元类结构、isa 走位、消息发送三阶段、动态特性 |
| `Mobile-Development/iOS/` | `iOS-RunLoop-Deep-Dive.md` | RunLoop 对象模型、状态机、用户态/内核态切换、触摸事件链路 |
| `Mobile-Development/iOS/` | `Objective-C-Underlying-Principles.md` | ObjC 对象系统、SEL/IMP、Block 底层、KVC/KVO、ARC/weak 实现 |
| `Mobile-Development/iOS/` | `Swift-Underlying-Principles.md` | Swift 基础语法、集合类型、泛型、面向协议编程、函数式编程、编译流水线、类型系统、Concurrency 和多线程 |
| `Mobile-Development/Cross-Platform-Mobile/` | `React-Native-Knowledge.md` | React/RN 渲染模型、新架构、原生模块、状态管理、性能、工程化与混合架构 |
| `Mobile-Development/Cross-Platform-Mobile/` | `Dart-Language-Knowledge.md` | Dart 类型系统、空安全、面向对象、Records、Patterns、异步、Isolate、Package 与测试 |
| `Mobile-Development/Cross-Platform-Mobile/` | `Flutter-Development-Architecture.md` | Flutter 渲染、状态、分层架构、原生通信、Add-to-App、模块化、性能与测试 |
| `Mobile-Development/Cross-Platform-Mobile/` | `Flutter-Native-Components-Integration.md` | iOS/Android 原生能力提供给 Flutter 使用、Platform Channel、PlatformView、Plugin、Pigeon 与示例代码 |
| `Python-Backend/` | `Python-Language-Knowledge.md` | Python 语言核心语法、对象模型、并发、GIL、标准库与编码习惯 |
| `Python-Backend/` | `Python-Project-Scaffold.md` | Python 项目结构、配置、日志、FastAPI 示例、数据库、测试、Docker 化 |
| `Python-Backend/` | `Python-Server-Technologies-Architecture.md` | HTTP API、数据库、缓存、消息队列、安全、可观测性、部署与架构风格 |
| `Python-Backend/` | `Python-Web-Backend-Frameworks.md` | FastAPI、Django、Flask、Starlette、gRPC、GraphQL 与框架选型 |
| `Frontend-Languages/` | `JavaScript-TypeScript-Knowledge.md` | JavaScript 运行模型、闭包、原型、异步、模块化与 TypeScript 类型系统、严格配置 |
| `Compiler-Systems/` | `LLVM-Deep-Dive.md` | LLVM 架构、IR、Clang、Pass、优化、Swift/ObjC 编译链与学习路线 |
| `Development-Environment/` | `macOS-Basic-Environment-Setup.md` | macOS 基础环境搭建、Homebrew、Oh My Zsh、nvm、rbenv、pyenv 与常用版本管理命令 |
| `Development-Environment/` | `Frontend-Development-Environment-Setup.md` | 前端开发环境搭建、Node.js、包管理器、Vite、TypeScript、代码质量、测试、调试与发布流程 |

## Guidelines for Adding New Content

- Create new `.md` files in the most relevant topic directory with descriptive kebab-case names
- Keep repository-root files limited to project-level guidance and metadata
- Use ASCII text diagrams for flows and architecture (no image dependencies)
- Follow the existing pattern: overview → detailed sections with diagrams → summary table at the end
- If extending an existing topic, prefer editing the existing file rather than creating a new one
