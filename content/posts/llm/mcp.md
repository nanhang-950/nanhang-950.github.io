---
title: 部署本地大模型配置MCP
date: 2026-01-16 20:51:38
categories: LLM
tags:
mathjax:
top:
---

## 前言

在一些线下断网环境下，我们如果仍然需要使用大模型。

## 部署ollama

通过 ollama 进行本地大模型部署。

### 官网下载部署

通过在官网下载相应系统的安装包安装 ollama。

官网：https://ollama.com/download

### Docker 部署

执行以下命令搜索 docker hub 上的 ollama 镜像。

```shell
❯ docker search ollama
NAME                       DESCRIPTION                                      STARS     OFFICIAL
ollama/ollama              The easiest way to get up and running with l…   1482
alpine/ollama              Minimal CPU-only Ollama Docker Image             11
dustynv/ollama             https://github.com/dusty-nv/jetson-container…   8
shinejh0528/ollama         ollama + fastAPI server                          0
tmvdl/ollama               An Ollama Docker Image                           1
eisai/ollama               ollama for Windows with CUDA support.            1
litellm/ollama             An OpenAI API compatible server for local LL…   14
dimaskiddo/ollama          Debian Based Ollama Image Repository             0
tobix99/ollama             Fork of normal ollama with this fix: https:/…   0
leopony/ollama                                                              3
soldemeyer/ollama                                                           0
dhiltgen/ollama                                                             0
zoull/ollama                                                                0
mthreads/ollama                                                             2
wanhuatong/ollama                                                           0
diliprenkila/ollama                                                         0
intoreal/ollama                                                             0
innodiskorg/ollama         For AccelBrain use. v0.1 builed from ollama …   0
vbehar/ollama                                                               0
sxk1633/ollama                                                              0
madebytimo/ollama                                                           0
saladtechnologies/ollama   Adds a salad entrypoint to the ollama image      0
olegkarenkikh/ollama                                                        0
linyinglv/ollama           Jupyter + Ollama with CUDA 12.4 base             0
ofalolu/ollama                                                              0
```

选择第一个镜像下拉：

```shell
docker pull ollama/ollama
```

创建容器：

```shell
docker run -d -p 11434:11434 --name ollama --restart always ollama/ollama:latest
```

进入容器：

```shell
docker exec -it ollama /bin/bash
```

## 模型部署

### 下拉模型

`hf.co/byteshape/Qwen3-30B-A3B-Instruct-2507-GGUF`是一个经过量化之后的模型。

文章：[一个30B的Qwen模特走进了树莓派......以及实时运行](https://byteshape.com/blogs/Qwen3-30B-A3B-Instruct-2507/)

执行以下命令部署大模型：

```shell
ollama pull hf.co/byteshape/Qwen3-30B-A3B-Instruct-2507-GGUF
ollama run hf.co/byteshape/Qwen3-30B-A3B-Instruct-2507-GGUF:FILE_NAME.gguf
```

### 修改名称

模型名字太长，我们可以修改一下。通过引用当前模型创建一个新模型来修改名字。

- 创建一个Modelfile

```shell
echo "FROM hf.co/byteshape/Qwen3-30B-A3B-Instruct-2507-GGUF:latest" > Modelfile
```

- 生成新模型

```shell
ollama create qwen3-30b -f Modelfile
```

## 配置MCP

### 配置本地模型

ollama api：

```
http://host.docker.internal:11434
```

- 测试 Chat API：

```shell
curl http://localhost:11434/api/chat `
  -H "Content-Type: application/json" `
  -d '{
    "model": "hf.co/byteshape/Qwen3-30B-A3B-Instruct-2507-GGUF",
    "messages": [
      { "role": "system", "content": "你是一个代码分析专家" },
      { "role": "user", "content": "malloc 失败后为什么必须检查返回值？" }
    ],
    "stream": false
  }'
```

- 打开 Github Copilot 聊天界面，然后选择 Ollama 模型。

<img src="/assets/image-20260205202846692.png" alt="image-20260205202846692" style="zoom:50%;" />

### 添加MCP Server

在 VSCode 上栏中运行`MCP: Open User Configuration`命令，该命令会在您的用户配置文件中打开`mcp.json`文件。然后，可以手动将服务器配置添加到该文件中。

添加配置文件内容：

```json
{
  "mcpServers": {
    "ida-pro-mcp": {
      "command": "C:\\Program Files\\Python\\python3\\python.exe",
      "args": [
        "C:\\Users\\nanhang\\AppData\\Roaming\\Python\\Python313\\site-packages\\ida_pro_mcp\\server.py"
      ],
      "timeout": 1800,
      "disabled": false,
      "env": {
        "PYTHONPATH": "C:\\Users\\nanhang\\AppData\\Local\\DBG\\WinDbgScripts"
      }
    },
    "jadx-mcp-server": {
      "command": "C:\\Program Files\\Python\\python3\\python.exe",
      "args": [
        "H:\\Tools\\AndroidTools\\jadx-mcp-server\\jadx_mcp_server.py"
      ]
    }
  }
}
```

## 测试

随便找一道逆向题测试一下效果：

题目链接：https://files.buuoj.cn/files/ee7f29503c7140ae31d8aafc1a7ba03f/attachment.tar

UPX 脱个壳，然后使用 ida-pro-mcp 分析求解：

```
## 求解CTF逆向题
分析代码逻辑，逆向找到flag
```

