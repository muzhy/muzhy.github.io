+++
isCJKLanguage = true
title = "本地部署DeepSeek-Coder V2并接入到VS Code"
description = "使用ollama在本地部署DeepSeek-Coder V2并通过Continue插件接入到VS Code"
keywords = ["DeepSeek", "VS Code", "Continue"]
date = 2025-03-09T16:11:17+08:00
authors = ["木章永"]
tags = ["DeepSeek", "VS Code", "Continue"]
categories = ["DeepSeek"]
cover = '/images/code_ai_agent.png'
draft = true
+++

# 什么是 DeepSeek-Coder V2
DeepSeek-Coder-V2是DeepSeek团队推出的基于MoE架构的智能代码模型，支持338中编程语言，几乎覆盖所有主流和小众编程语言，一次能处理长达128K的代码文件。

Github 开源仓库地址：[https://github.com/deepseek-ai/DeepSeek-Coder-V2](https://github.com/deepseek-ai/DeepSeek-Coder-V2)

用过DeepSeek很多，但是已经有了DeepSeek-r1，为什么还要DeepSeek-Coder 呢？

原因当然是本地部署满血版DeepSeek-r1的成本太高，蒸馏版的DeepSeek-r1在代码辅助方面的功能相比DeepSeek-Coder 优势并不明显（具体看多少参数的蒸馏版），而DeepSeek-Coder-V2的部署成本更低。

针对代码辅助的场景，通过减少模型能力以降低部署成本并进行针对性优化至少在目前的阶段是比价合理的做法。

![](/images/DeepSeek-Coder-V2性能比较.png)
DeepSeek-Coder-V2于其他模型能力对比，可以看见性能是可以挤进第一梯队的

# 本地部署DeepSeek-Coder-V2

## 安装ollama
要部署DeepSeek-Coder-V2可以通过[ollama](https://ollama.com/)进行安装
从官网下载 https://ollama.com/download 对应操作系统的安装包后双击安装即可。

安装ollama之后，可以在命令行执行
```PowerShell
ollama 
```
检查是否安装完成

以Windows为例，安装ollama之后，下载的模型文件默认是存放到`C:\Users\%UserName%\.ollama`
如果需要修改模型文件的位置，需要添加环境变量`OLLAMA_MODELS`，将路径设置为要保存模型的路径。

![](/images/配置OLLAMA环境bianl.png)

修改环境变量之后需要重新打开命令行窗口，否则环境环境变量可能不生效。可以在命令行窗口执行
```PowerShell
$Env:OLLAMA_MODELS
```
检查修改的环境变量是否生效

## 下载DeepSeek-Coder-V2模型
![](/images/DeepSeek-coder-v2模型版本.png)
deepseek-coder-v2有16b和236b两个版本，对于我羸弱的PC而言，只能跑得动16b的。

在命令行执行
```PowerShell
ollama pull deepseek-coder-v2
```
下载模型文件，如果需要下载236b版本的执行
```PoserShell
ollama pull deepseek-coder-v2:236b
```

也可以执行`ollama run deepseek-coder-v2`下载模型并启动，不过个人更喜欢分步骤操作。

下载完成后可以运行
```PowerShell
ollama ls
```
查看已经下载了的模型文件

## 运行DeepSeek-Coder-V2
执行
```PowerShell
ollama run deepseek-coder-v2:16b
```
运行deepseek-coder-v2:16b，下载236b版本的根据执行`ollama ls`后列出来的模型名修改命令

运行DeepSeek-Coder-V2最好是有8G的显存，如果显存不够的话，可能会导致需要使用CPU运行模型进行推理，用CPU运行的话速度会慢很多
启动模型之后，可以执行`ollama ps`查看正在运行的模型
```PowerShell
> ollama ps
NAME                     ID              SIZE     PROCESSOR          UNTIL             
deepseek-coder-v2:16b    63fb193b3a9b    10 GB    29%/71% CPU/GPU    4 minutes from now
```
受限于我本机羸弱的性能，只有8G 的显存，模型并不能完全装入到显存中，不过就个人使用体验感觉已经够用了。

执行了`ollama run`之后其实已经可以在命令行发送消息使用deepseek-coder-v2了
```
>>> who are you
 I am DeepSeek Coder, an intelligent assistant developed by China's DeepSeek
company, designed to provide services such as information retrieval,
conversational interaction, and answering questions through natural language
processing and machine learning technologies.
```


# 接入VS Code

使用`ollama`运行模型，默认会监听`11434`端口，本机程序可以通过`11434`端口调用模型API。
要在VS Code中使用deepseek-coder-v2就需要访问这个端口，如果需要在其他机器访问则需要把端口暴露出去，不在本文的内容之内。

VS Code上有很多插件可以使用，如`Continue`或`CodeGPT`，我是使用`Continue`接入大模型的

安装`Continue`之后，需要修改`.continue/config.json`文件接入部署在本地的大模型
```json
{
  "models": [
    {
      "title": "DeepSeek Coder v2",
      "model": "deepseek-coder-v2:16b",
      "provider": "ollama",
      "apiBase": "http://localhost:11434"
    }
  ],
  "tabAutocompleteModel": {
    "title": "deepseek-coder-v2",
    "provider": "ollama",
    "model": "deepseek-coder-v2:16b"
  },
  "contextProviders": [
    {
      "name": "codebase",
      "params": {}
    }
  ],
  ...
}
```
需要在`models`中添加部署的deepseek-coder-v2的配置，代码自动补全需要添加`tabAutocompleteModel`

配置完成之后就可以在VS Code中使用了。

代码补全功能在写代码的时候会提示，如果需要让AI添加/修改代码，可以选中所需要的代码，然后按`ctrl+I`，在聊天框中输入想要AI做的工作后就可以了。

更多关于`Continue`的操作可以查看[官方文档](https://docs.continue.dev/customize/overview)
