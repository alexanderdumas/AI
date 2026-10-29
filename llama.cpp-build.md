# LLAMA.cpp 在NVIDIA GPU上编译测试
**概述:** 在边缘计算中，通常使用的C++开发环境，如果安装docker镜像或者VLLM这类很重框架不适合边缘环境中，在比较有限的资源环境下，LLAMA.cpp就是个很不错的选择。下面讲述下如何在A2000 GPU上跑Qwen3-0.6B量化模型的。

## 检查NVIDIA环境
通过命令查看CUDA版本：nvidia-smi，查看nvcc --help 是否安装

## 下载LLAMA.cpp源代码
直接克隆源码：git clone https://github.com/ggml-org/llama.cpp.git

## 编译源代码
由于仓库自带了CMakePresets.json，可以直接使用CMake工具，那么再添加一个单独的用户的文件CMakeUserPresets.json。内容如下
```json
{
  "version": 4,
  "configurePresets": [
    {
      "name": "x64-linux-gcc-cuda-debug",
      "inherits": ["base", "x64-linux-gcc", "debug"],
      "cacheVariables": {
        "GGML_CUDA": "ON",
        "LLAMA_BUILD_UI": "ON"
      }
    }
  ]
}
```

## 模型下载
下载模型Qwen3-0.6B-Q8_0.gguf，可以去huggingface或者modlescope上下载，国内用户建议去modlescope上下载，速度很快。

## 启动服务
执行：./build-x64-linux-gcc-cuda-debug/bin/llama-server -m /opt/models/gguf/qwen/Qwen3-0.6B-Q8_0.gguf --port 8080 -ngl all -fa on

## 浏览器对话
直接打开：http://127.0.0.1:8080 就可以使用对话了
