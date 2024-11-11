---
title: 微软Magentic-One 智能代理
date: 2024-11-11 10:58:40 +0800
categories: 技术
tags:
  - 智能体
layout: single
toc: "true"
toc_label: 目录
toc_icon: list
toc_sticky: "true"
sidebar:
  nav: docs
---
# 和操作系统交互的方式
通过编程接口与浏览器和文件系统交互，而非直接控制鼠标和键盘。它利用Playwright等自动化工具，以编程方式执行浏览器操作，如访问URL、滚动页面、点击链接等。这种方法比模拟人工输入更可靠和高效。


## 1. 与浏览器的交互

`autogen-magentic-one` 提供了 `MultimodalWebSurfer` 类，允许代理在网页上执行搜索、访问页面、点击链接、滚动视图等操作。该类利用 Playwright 库控制浏览器行为，并通过 `MarkdownConverter` 将网页内容转换为可处理的格式。 

## 2. 与文件系统的交互

使用 Python 的标准库（如 `os` 和 `aiofiles`）与文件系统交互。例如，`LogHandler` 类继承自 `logging.FileHandler`，用于将日志记录到文件中。此外，`MultimodalWebSurfer` 类在初始化时，可以指定 `downloads_folder` 参数，设置下载文件的保存路径。 

## 3. 与终端的交互

`autogen-magentic-one` 主要通过日志记录和调试信息与终端交互。在调试模式下，代理会输出详细的日志信息，帮助开发者了解代理的内部状态和行为。此外，代理可以通过终端接受用户输入，执行相应的操作。

# 和Anthropic的Comupter Use的对比

这种方法和Anthropic的Comupter Use 的API是不一样的。Anthropic的工具调用会转换成系统调用，最后转换为鼠标和键盘的操作，对模型的要求更高一些，也更灵活一些。

哪种方式会成为主流，需要持续观察。