# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **developer knowledge base** — a collection of markdown documents covering iOS, mobile cross-platform development, Python backend development, frontend languages, compiler fundamentals, and development environment setup. There is no source code, build system, or test suite in this repository.

## Document Structure

| Directory | File | Topic |
|-----------|------|-------|
| `iOS-Development/` | `UIKit-UIView-Layer-Rendering.md` | UIView/CALayer 渲染管线、三棵树模型、响应者链、常用组件 |
| `iOS-Development/` | `iOS-Architecture-Design.md` | MVC/MVVM/响应式框架、UIKit vs SwiftUI 架构对比、Coordinator 模式 |
| `iOS-Development/` | `SwiftUI-UIKit-Comparison.md` | SwiftUI 定位、与 UIKit 的关系、编程模型差异、优劣对比、混编和选型 |
| `iOS-Development/` | `iOS-Audio-Video-Development.md` | 音视频基础、AVFoundation 架构、音视频采集/播放/编解码、实时通信、渲染优化 |
| `iOS-Development/` | `iOS-Security-Mechanisms.md` | Keychain、Data Protection、Sandbox、ATS、代码签名、混淆、反调试、安全落地方案 |
| `iOS-Development/` | `iOS-Performance-Optimization.md` | 启动优化、包体积、内存、UI渲染、流畅度、CPU、I/O、网络与性能监控 |
| `iOS-Underlying-Principles/` | `iOS-Runtime-Deep-Dive.md` | Runtime 启动加载、类/元类结构、isa 走位、消息发送三阶段、动态特性 |
| `iOS-Underlying-Principles/` | `iOS-RunLoop-Deep-Dive.md` | RunLoop 对象模型、状态机、用户态/内核态切换、触摸事件链路 |
| `iOS-Underlying-Principles/` | `Objective-C-Underlying-Principles.md` | ObjC 对象系统、SEL/IMP、Block 底层、KVC/KVO、ARC/weak 实现 |
| `iOS-Underlying-Principles/` | `Swift-Underlying-Principles.md` | Swift 基础语法、集合类型、泛型、面向协议编程、函数式编程、编译流水线、类型系统、Concurrency 和多线程 |
| `Cross-Platform-Mobile/` | `React-Native-Knowledge.md` | React/RN 渲染模型、新架构、原生模块、状态管理、性能、工程化与混合架构 |
| `Cross-Platform-Mobile/` | `Dart-Language-Knowledge.md` | Dart 类型系统、空安全、面向对象、Records、Patterns、异步、Isolate、Package 与测试 |
| `Cross-Platform-Mobile/` | `Flutter-Development-Architecture.md` | Flutter 渲染、状态、分层架构、原生通信、Add-to-App、模块化、性能与测试 |
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
