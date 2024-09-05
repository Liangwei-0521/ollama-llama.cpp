# **基于Ollama实现ChatGLM3-6B大模型本地部署技术方案**

### **1、设备**

**平台：AutoDL**

```
 C:\Users\qnhl>ssh -p 24375 root@connect.westc.gpuhub.com
root@connect.westc.gpuhub.com's password:
Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.15.0-86-generic x86_64)Documentation:  https://help.ubuntu.comManagement:     https://landscape.canonical.comSupport:        https://ubuntu.com/advantageThis system has been minimized by removing packages and content that are
not required on a system that users do not log into.To restore this content, you can run the 'unminimize' command.
Last login: Wed Sep  4 15:23:05 2024 from 127.0.0.1
+--------------------------------------------------AutoDL--------------------------------------------------------+
目录说明:
╔═════════════════╦════════╦════╦═════════════════════════════════════════════════════════════════════════╗
║目录              ║名称    ║速度║说明                                                                     ║
╠═════════════════╬════════╬════╬═════════════════════════════════════════════════════════════════════════╣
║/                ║系 统 盘║一般║实例关机数据不会丢失，可存放代码等。会随保存镜像一起保存。               ║
║/root/autodl-tmp ║数 据 盘║ 快 ║实例关机数据不会丢失，可存放读写IO要求高的数据。但不会随保存镜像一起保存 ║
╚═════════════════╩════════╩════╩═════════════════════════════════════════════════════════════════════════╝
CPU ：12 核心
内存：90 GB
GPU ：NVIDIA GeForce RTX 4090, 1
存储：
  系 统 盘/               ：96% 29G/30G
  数 据 盘/root/autodl-tmp：70% 35G/50G
+----------------------------------------------------------------------------------------------------------------+
```

### **2、创建虚拟环境**

```
conda create --name ChatGLM3 python=3.11
conda activate ChatGLM3
```

### **3、安装Ollama**

```
>>> curl -fsSL https://ollama.com/install.sh | sh
>>> Installing ollama to /usr/local
>>> Downloading Linux amd64 bundle
############################################################################################################################################################## 100.0%
>>> Creating ollama user...
>>> Adding ollama user to video group...
>>> Adding current user to ollama group...
>>> Creating ollama systemd service...
WARNING: Unable to detect NVIDIA/AMD GPU. Install lspci or lshw to automatically detect and install GPU dependencies.
>>> The Ollama API is now available at 127.0.0.1:11434.
>>> Install complete. Run "ollama" from the command line.
```

* **显示安装成功，如果网络不稳定，可以多尝试上述命令，或者改变sh文件的下载镜像。**
* **后台运行ollama**

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~# nohup ollama serve > /dev/null 2>&1 &
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~# ollama ls
NAME                            ID              SIZE    MODIFIED
```

### **4、下载hugginggface 模型：ChatGLM3-6B**

* **需要注意的是：可以指定路径下载大模型权重文件。**

```
from huggingface_hub import snapshot_download
snapshot_download(repo_id="THUDM/chatglm3-6b", local_dir="./ChatGLM3/")
```

本项目存放大模型权重文件的路径：**/root/autodl-tmp/AI_Model/ChatGLM3.**

进入路径，抓取是否存在大模型ChatGLM3的重要文件，包括config.json、safetensors、tokenzier等。

### **5、拉取 llama.cpp 进行模型量化转换**

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~# git clone https://github.com/ggerganov/llama.cpp.git
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~# cd llama.cpp
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~# make 
```

* **显示下面内容**

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~/llama.cpp# ls
AUTHORS                        ggml                           llama-gritlm           llama-speculative
CMakeLists.txt                 gguf-py                        llama-imatrix          llama-tokenize
CMakePresets.json              grammars                       llama-infill           llama-vdot
CONTRIBUTING.md                include                        llama-llava-cli        main
LICENSE                        libllava.a                     llama-lookahead        media
Makefile                       llama-baby-llama               llama-lookup           models
Package.swift                  llama-batched                  llama-lookup-create    mypy.ini
README.md                      llama-batched-bench            llama-lookup-merge     pocs
SECURITY.md                    llama-bench                    llama-lookup-stats     poetry.lock
ci                             llama-benchmark-matmult        llama-minicpmv-cli     prompts
cmake                          llama-cli                      llama-parallel         pyproject.toml
common                         llama-convert-llama2c-to-ggml  llama-passkey          pyrightconfig.json
convert_hf_to_gguf.py          llama-cvector-generator        llama-perplexity       requirements
convert_hf_to_gguf_update.py   llama-embedding                llama-q8dot            requirements.txt
convert_llama_ggml_to_gguf.py  llama-eval-callback            llama-quantize         scripts
convert_lora_to_gguf.py        llama-export-lora              llama-quantize-stats   server
docs                           llama-gbnf-validator           llama-retrieval        spm-headers
examples                       llama-gguf                     llama-save-load-state  src
flake.lock                     llama-gguf-hash                llama-server           tests
flake.nix                      llama-gguf-split               llama-simple
```

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:/llama.cpp# cd ..
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:# cd llama.cpp/
```

* **安装依赖**

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:# cd llama.cpp/pip install -r requirements.txt
```

发现存在转换文件代码**convert_hf_to_gguf.py**将hugging face下载的大模型

* **进行模型量化，输出转换细节**

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~/llama.cpp# python convert_hf_to_gguf.py /root/autodl-tmp/AI_Model/ChatGLM3
INFO:hf-to-gguf:Loading model: ChatGLM3
INFO:gguf.gguf_writer:gguf: This GGUF file is for Little Endian only
INFO:hf-to-gguf:Exporting model...
INFO:hf-to-gguf:gguf: loading model weight map from 'model.safetensors.index.json'
INFO:hf-to-gguf:gguf: loading model part 'model-00001-of-00007.safetensors'
INFO:hf-to-gguf:token_embd.weight,         torch.float16 --> F16, shape = {4096, 65024}
INFO:hf-to-gguf:blk.0.attn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.0.ffn_down.weight,     torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.0.ffn_up.weight,       torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.0.ffn_norm.weight,     torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.0.attn_output.weight,  torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.0.attn_qkv.bias,       torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.0.attn_qkv.weight,     torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.1.attn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.1.ffn_down.weight,     torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.1.ffn_up.weight,       torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.1.ffn_norm.weight,     torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.1.attn_output.weight,  torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.1.attn_qkv.bias,       torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.1.attn_qkv.weight,     torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.2.attn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.2.ffn_down.weight,     torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.2.ffn_up.weight,       torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.2.ffn_norm.weight,     torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.2.attn_output.weight,  torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.2.attn_qkv.bias,       torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.2.attn_qkv.weight,     torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.3.attn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.3.ffn_norm.weight,     torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.3.attn_output.weight,  torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.3.attn_qkv.bias,       torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.3.attn_qkv.weight,     torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:gguf: loading model part 'model-00002-of-00007.safetensors'
INFO:hf-to-gguf:blk.3.ffn_down.weight,     torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.3.ffn_up.weight,       torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.4.attn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.4.ffn_down.weight,     torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.4.ffn_up.weight,       torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.4.ffn_norm.weight,     torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.4.attn_output.weight,  torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.4.attn_qkv.bias,       torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.4.attn_qkv.weight,     torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.5.attn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.5.ffn_down.weight,     torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.5.ffn_up.weight,       torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.5.ffn_norm.weight,     torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.5.attn_output.weight,  torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.5.attn_qkv.bias,       torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.5.attn_qkv.weight,     torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.6.attn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.6.ffn_down.weight,     torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.6.ffn_up.weight,       torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.6.ffn_norm.weight,     torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.6.attn_output.weight,  torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.6.attn_qkv.bias,       torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.6.attn_qkv.weight,     torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.7.attn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.7.ffn_down.weight,     torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.7.ffn_up.weight,       torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.7.ffn_norm.weight,     torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.7.attn_output.weight,  torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.7.attn_qkv.bias,       torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.7.attn_qkv.weight,     torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.8.attn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:gguf: loading model part 'model-00003-of-00007.safetensors'
INFO:hf-to-gguf:blk.10.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.10.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.10.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.10.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.10.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.10.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.10.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.11.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.11.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.11.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.11.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.11.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.11.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.11.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.12.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.12.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.12.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.12.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.12.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.12.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.8.ffn_down.weight,     torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.8.ffn_up.weight,       torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.8.ffn_norm.weight,     torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.8.attn_output.weight,  torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.8.attn_qkv.bias,       torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.8.attn_qkv.weight,     torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.9.attn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.9.ffn_down.weight,     torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.9.ffn_up.weight,       torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.9.ffn_norm.weight,     torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.9.attn_output.weight,  torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.9.attn_qkv.bias,       torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.9.attn_qkv.weight,     torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:gguf: loading model part 'model-00004-of-00007.safetensors'
INFO:hf-to-gguf:blk.12.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.13.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.13.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.13.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.13.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.13.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.13.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.13.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.14.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.14.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.14.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.14.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.14.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.14.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.14.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.15.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.15.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.15.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.15.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.15.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.15.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.15.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.16.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.16.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.16.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.16.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.16.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.16.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.16.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.17.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.17.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.17.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.17.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.17.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:gguf: loading model part 'model-00005-of-00007.safetensors'
INFO:hf-to-gguf:blk.17.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.17.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.18.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.18.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.18.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.18.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.18.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.18.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.18.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.19.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.19.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.19.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.19.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.19.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.19.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.19.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.20.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.20.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.20.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.20.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.20.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.20.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.20.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.21.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.21.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.21.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.21.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.21.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.21.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.21.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.22.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:gguf: loading model part 'model-00006-of-00007.safetensors'
INFO:hf-to-gguf:blk.22.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.22.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.22.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.22.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.22.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.22.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.23.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.23.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.23.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.23.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.23.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.23.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.23.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.24.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.24.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.24.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.24.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.24.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.24.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.24.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.25.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.25.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.25.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.25.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.25.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.25.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.25.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:blk.26.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.26.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.26.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.26.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.26.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.26.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:gguf: loading model part 'model-00007-of-00007.safetensors'
INFO:hf-to-gguf:output_norm.weight,        torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.26.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.27.attn_norm.weight,   torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.27.ffn_down.weight,    torch.float16 --> F16, shape = {13696, 4096}
INFO:hf-to-gguf:blk.27.ffn_up.weight,      torch.float16 --> F16, shape = {4096, 27392}
INFO:hf-to-gguf:blk.27.ffn_norm.weight,    torch.float16 --> F32, shape = {4096}
INFO:hf-to-gguf:blk.27.attn_output.weight, torch.float16 --> F16, shape = {4096, 4096}
INFO:hf-to-gguf:blk.27.attn_qkv.bias,      torch.float16 --> F32, shape = {4608}
INFO:hf-to-gguf:blk.27.attn_qkv.weight,    torch.float16 --> F16, shape = {4096, 4608}
INFO:hf-to-gguf:output.weight,             torch.float16 --> F16, shape = {4096, 65024}
INFO:hf-to-gguf:Set meta model
INFO:hf-to-gguf:Set model parameters
INFO:hf-to-gguf:Set model tokenizer
WARNING:transformers_modules.ChatGLM3.tokenization_chatglm:Setting eos_token is not supported, use the default one.  
WARNING:transformers_modules.ChatGLM3.tokenization_chatglm:Setting pad_token is not supported, use the default one.  
WARNING:transformers_modules.ChatGLM3.tokenization_chatglm:Setting unk_token is not supported, use the default one.  
INFO:gguf.vocab:Setting special token type eos to 2
INFO:gguf.vocab:Setting special token type pad to 0
INFO:gguf.vocab:Setting chat_template to {% for message in messages %}{% if loop.first %}[gMASK]sop<|{{ message['role'] }}|>
 {{ message['content'] }}{% else %}<|{{ message['role'] }}|>
 {{ message['content'] }}{% endif %}{% endfor %}{% if add_generation_prompt %}<|assistant|>{% endif %}
INFO:hf-to-gguf:Set model quantization version
INFO:gguf.gguf_writer:Writing the following files:
INFO:gguf.gguf_writer:/root/autodl-tmp/AI_Model/ChatGLM3/chatglm3-6B-F16.gguf: n_tensors = 199, total_size = 12.5G   
Writing: 100%|███████████████████████████████████████████████████████████████| 12.5G/12.5G [00:10<00:00, 1.25Gbyte/s]INFO:hf-to-gguf:Model successfully exported to /root/autodl-tmp/AI_Model/ChatGLM3/chatglm3-6B-F16.gguf
```

输出上述内容，则转换成功。

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~/autodl-tmp/AI_Model/ChatGLM3# ls
MODEL_LICENSE                     model-00005-of-00007.safetensors  pytorch_model-00005-of-00007.bin
README.md                         model-00006-of-00007.safetensors  pytorch_model-00006-of-00007.bin
chatglm3-6B-F16.gguf              model-00007-of-00007.safetensors  pytorch_model-00007-of-00007.bin
config.json                       model.safetensors.index.json      pytorch_model.bin.index.json
configuration_chatglm.py          modeling_chatglm.py               quantization.py
model-00001-of-00007.safetensors  pytorch_model-00001-of-00007.bin  special_tokens_map.json
model-00002-of-00007.safetensors  pytorch_model-00002-of-00007.bin  tokenization_chatglm.py
model-00003-of-00007.safetensors  pytorch_model-00003-of-00007.bin  tokenizer.model
model-00004-of-00007.safetensors  pytorch_model-00004-of-00007.bin  tokenizer_config.json
```

量化模型权重文件chatglm3-6B-F16.gguf  存在，验证模型量化成功。

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~/autodl-tmp/AI_Model/ChatGLM3# ls
MODEL_LICENSE                     model-00005-of-00007.safetensors  pytorch_model-00005-of-00007.bin
README.md                         model-00006-of-00007.safetensors  pytorch_model-00006-of-00007.bin
chatglm3-6B-F16.gguf              model-00007-of-00007.safetensors  pytorch_model-00007-of-00007.bin
config.json                       model.safetensors.index.json      pytorch_model.bin.index.json
configuration_chatglm.py          modeling_chatglm.py               quantization.py
model-00001-of-00007.safetensors  pytorch_model-00001-of-00007.bin  special_tokens_map.json
model-00002-of-00007.safetensors  pytorch_model-00002-of-00007.bin  tokenization_chatglm.py
model-00003-of-00007.safetensors  pytorch_model-00003-of-00007.bin  tokenizer.model
model-00004-of-00007.safetensors  pytorch_model-00004-of-00007.bin  tokenizer_config.json
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~/autodl-tmp/AI_Model/ChatGLM3#
```

### **6、ollama部署ChatGLM3大模型**

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~/autodl-tmp/AI_Model/ChatGLM3# ollama create chatglm3-6B-F16.gguf
Error: open /root/autodl-tmp/AI_Model/ChatGLM3/Modelfile: no such file or directory
```

* **说明需要存在Modelfile文件，现在进行编辑。**

```
touch Modelfile
nano Modelfile
vi Modelfile
------------------ 输入下面这句命令--------------------------
FROM /root/autodl-tmp/AI_Model/ChatGLM3/chatglm3-6B-F16.gguf
```

* **返回终端，执行下面命令**

`(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~/autodl-tmp/AI_Model/ChatGLM3# ollama create chatglm3-6B-F16.gguf -f Modelfile transferring model data 100% using existing layer sha256:ea80f4702bcadb6bac526043e28ff6a6f3638cb597d44434755cf5a67c21174f creating new layer sha256:755c9c8d22e43623484a3773b1f2374c44b350b0856a0746b83b63b92f67ee3b writing manifest success`

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~# ollama ls
NAME                            ID              SIZE    MODIFIED
chatglm3-6B-F16.gguf:latest     a86622a8e7f0    12 GB   43 minutes ago运行大模型
```

显示成功！

* **使用ollama run + 模型名称 实现运行本地大模型。**

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:~/autodl-tmp/AI_Model/ChatGLM3# ollama run chatglm3-6B-F16.gguf:latest
```

* **进入模型终端，测试prompt样例**

```
>>> 介绍强化学习的基本概念和应用领域吧。

### 强化学习
强化学习是机器学习的一个子领域，它关注的是如何基于与环境的互动来学习决策策略。它的核心思想是通过不断尝试新的行动-反馈 pairs（  
动作-奖励）来优化策略。强化学习的主要任务是在给定环境的条件下，找到能够最大化累积奖励的决策序列。

强化学习的基本概念包括：状态、动作、奖励和策略。**状态**是指当前的环境 condition，**动作**是指可以采取的操作或行为，**奖励   
**是对某个动作的发生给予的反馈，**策略**则是在某个状态下选择某个动作的概率分布。

强化学习的应用领域非常广泛，其中最典型的应用是游戏控制，比如国际象棋、围棋、扑克等。除此之外，强化学习还被广泛应用于自动驾驶   
、机器人控制、推荐系统、金融投资等领域。
```

### **7、ollama webUI 可视化**

考虑到使用web界面访问本地大模型会更直观。现介绍基于autodl服务器实现方法。

参考链接：https://github.com/open-webui/open-webui

* **安装web u**i

```
PS C:\Users\qnhl> docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
Unable to find image 'ghcr.io/open-webui/open-webui:main' locally
main: Pulling from open-webui/open-webui
e4fff0779e6d: Pull complete
d97016d0706d: Pull complete
53db1713e5d9: Pull complete
a8cd795d9ccb: Pull complete
de3ba92de392: Pull complete
6f4d87c224b0: Pull complete
4f4fb700ef54: Pull complete
dd92a6022ddb: Pull complete
bbbfed48a772: Pull complete
a825beebdb5b: Pull complete
fd0f6bc6022b: Pull complete                                                                                             4b10ca2c003a: Pull complete
a3ad9497b5bd: Pull complete
51f9a5aab456: Pull complete
b2d0f2e14def: Pull complete
de00304a38e7: Pull complete
Digest: sha256:739f63c9adbd6b40e6d4c99c6acc3ddf991bd181953fae4e05df14401d900af7
Status: Downloaded newer image for ghcr.io/open-webui/open-webui:main
087904ef657088c4689a32c2de6f1015c36c8d8de55dc1e276c0d61592579d21
```

* **展示效果**
![initial](https://github.com/user-attachments/assets/9af77eb2-f4fd-4c69-bae3-557dc5ebc082)


* **外部连接**
  在ollama web ui中选择**外部连接**，****为宿主机的端口号，即web ui 直接访问服务器宿主机的服务端口.
  目的 ip port 从平台中获取，”自定义服务“处。

```
https://u292800-8d7c-f5d28469.westc.gpuhub.com:****
```

* **映射容器端口6006**

可以理解为将ollama的服务映射到服务器容器实例可对外暴露的端口，以6006为例。前提：通过SSH穿隧暴露使得外网可以访问容器实例6006端口。

同时，将服务器的ollama服务映射容器端口6006

```
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:# export OLLAMA_HOST=0.0.0.0:6006
(ChatGLM3) root@autodl-container-0282478d7c-f5d28469:# nohup ollama serve &
```

* **运行效果**

![show](https://github.com/user-attachments/assets/d01a4c3d-d72d-4973-a47b-447dbbc2b633)

