---
title: 安装
icon: material-symbols-light:download-rounded
createTime: 2025/06/09 10:29:31
permalink: /zh/guide/install/
---
# 安装
本节介绍如何安装DataFlow的依赖环境。

DataFlow的文本流水线部分的依赖环境可以通过以下指令安装(要求系统已经安装conda):

```shell
conda create -n dataflow python=3.10
conda activate dataflow

git clone https://github.com/Open-DataFlow/DataFlow
cd DataFlow
pip install -e .
```

其中torch和vllm等依赖的版本可以根据CUDA版本进行调整。

如何快速上手DataFlow的文本流水线请参照Quick Start一节。