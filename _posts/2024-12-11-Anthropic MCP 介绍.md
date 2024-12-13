---
title: Anthropic MCP 介绍
date: 2024-12-10 21:42:39 +0800
categories: 技术
tags:
  - AI
layout: single
toc: "true"
toc_label: 目录
toc_icon: list
toc_sticky: "true"
sidebar:
  nav: docs
comments: "true"
---
## MCP时序图


![](https://d2m4tio3tm4t0x.cloudfront.net/2024/12/e13f142ee2cbde7d77c63f9b6e6f6e5b.svg)
## MCP调试
### uvx命令的MCP Server无法启动

在Mac OS中，通过命令安装UV
```bash
curl -LsSf [https://astral.sh/uv/install.sh](https://astral.sh/uv/install.sh) | sh
```

安装的路径为：/Users/{username}/.local/bin  ，Claude MCP Server运行的Path环境变量中，不包含这个路径。所以需要在MCP配置文件中，uvx命令需要写成绝对路径。

这里注意，为了安全和隔离性考虑，Claude APP运行在一个独立的容器或沙盒环境中，这个环境和你的实际终端环境是分开的。

### Log日志的路径

~/Library/logs/Claude/mcp*.log