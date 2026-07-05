# 本地部署状态

更新时间：2026-07-05

## 当前结论

- ComfyUI 不是 Docker 部署；当前记录和已验证部署都是 Windows 原生 Python venv 版。
- Windows 部署路径：
  `D:\aigc-ai\ComfyUI\repo`
- Windows 启动方式：
  计划任务 `AIGC-ComfyUI` 调用 `D:\aigc-ai\start-comfyui.cmd`
- Mac 本机未安装 Docker CLI，未发现本机 Docker 版 ComfyUI 部署记录。
- 2026-07-05 在线检查：`192.168.0.102` 的 SSH 端口拒绝连接，`8188` ComfyUI HTTP 端口不可达。需要先确认 Windows 主机已开机、IP 仍为 `192.168.0.102`，以及 OpenSSH/网络可用。

## Mac

已完成：

- 克隆私有 playbook 到 `/Users/xunuo/Documents/aigc/project-docs`。
- 安装 playbook Node 依赖。
- 安装项目级 FFmpeg static：
  `/Users/xunuo/Documents/aigc/project-docs/node_modules/ffmpeg-static/ffmpeg`
- 安装 Ollama macOS 包与 `qwen2.5:1.5b` 模型。

注意：

- Ollama 模型已下载，但当前推理时 `llama-server process has terminated: signal: killed`，需要后续单独排查 macOS 内存/启动方式。短期可先用远端 API 或后续再修复 Ollama。

## Windows

部署根目录：

```text
D:\aigc-ai
```

已完成：

- 创建干净目录结构：
  - `D:\aigc-ai\ComfyUI`
  - `D:\aigc-ai\models`
  - `D:\aigc-ai\input`
  - `D:\aigc-ai\output`
  - `D:\aigc-ai\workflows`
  - `D:\aigc-ai\tools`
  - `D:\aigc-ai\logs`
- 安装 MinGit：
  `D:\aigc-ai\tools\MinGit\cmd\git.exe`
- 安装 Python 3.10：
  `D:\aigc-ai\tools\Python310\python.exe`
- 安装 ComfyUI：
  `D:\aigc-ai\ComfyUI\repo`
- 创建 ComfyUI venv：
  `D:\aigc-ai\ComfyUI\venv`
- 安装 PyTorch CUDA：
  `torch 2.5.1+cu121`
- CUDA 验证通过：
  `NVIDIA GeForce GTX 1060 6GB`
- 安装 ComfyUI requirements。
- 导入 playbook workflow：
  - `D:\aigc-ai\workflows\product-main-sd15-img2img.json`
  - `D:\aigc-ai\workflows\product-main-sd15-img2img-api.json`
- 注册 ComfyUI 可视化 workflow：
  - `workflows/aigc/product-main-sd15-img2img.json`
  - `workflows/aigc/product-main-sd15-img2img-api.json`
- 安装 SD1.5 checkpoint：
  `D:\aigc-ai\models\checkpoints\v1-5-pruned-emaonly.safetensors`
- 安装 RIFE：
  `D:\aigc-ai\tools\rife-ncnn-vulkan-20221029-windows\rife-ncnn-vulkan.exe`
- 安装 Real-ESRGAN：
  `D:\aigc-ai\tools\realesrgan-ncnn-vulkan-v0.2.0-windows\realesrgan-ncnn-vulkan.exe`
- 放行 Windows 防火墙端口：
  `AIGC-ComfyUI-8188`
- 创建后台启动任务：
  `AIGC-ComfyUI`

ComfyUI 当前状态：

```text
http://192.168.0.102:8188
```

历史已验证 `/system_stats` 可访问，返回：

- ComfyUI `0.26.0`
- Python `3.10.11`
- PyTorch `2.5.1+cu121`
- GPU `cuda:0 NVIDIA GeForce GTX 1060 6GB`
- CheckpointLoaderSimple 可识别：
  `v1-5-pruned-emaonly.safetensors`

已验证：

- SD1.5 checkpoint SHA256：
  `6ce0161689b3853acaa03779ec93eafe75a02f4ced659bee03f50797806fa2fa`
- ComfyUI API 出图成功，测试输出：
  `D:\aigc-ai\output\product-main\695123659466_sd15_v01_00002_.png`
- Mac 侧预览图：
  `/Users/xunuo/Documents/aigc/output-preview/695123659466_sd15_v01_00001_.png`

当前限制：

- 测试输入图是占位图：
  `D:\aigc-ai\input\product_695123659466_white.png`
  后续生产要替换成真实商品图，或在 ComfyUI `LoadImage` 节点里选择新图。
- 旧目录 `D:\ai` 已存在大量历史 AI 文件，本次部署没有改动它；新环境统一放在 `D:\aigc-ai`。

## 常用命令

### 检查 ComfyUI

```bash
curl http://192.168.0.102:8188/system_stats
```

### Windows 本机检查

```powershell
Invoke-RestMethod http://127.0.0.1:8188/system_stats
```

### 打开可视化 workflow

在浏览器打开：

```text
http://192.168.0.102:8188
```

从 ComfyUI 的工作流/文件加载入口选择：

```text
workflows/aigc/product-main-sd15-img2img.json
```

这份是可视化节点画布。`product-main-sd15-img2img-api.json` 是脚本/API 调用用的版本。

### 后台启动 ComfyUI

Windows 上推荐使用计划任务启动：

```powershell
schtasks /Run /TN AIGC-ComfyUI
```

计划任务调用的脚本：

```text
D:\aigc-ai\start-comfyui.cmd
```

脚本内容等价于：

```bat
cd /d D:\aigc-ai\ComfyUI\repo
D:\aigc-ai\ComfyUI\venv\Scripts\python.exe main.py --listen 0.0.0.0 --port 8188 --lowvram --input-directory D:\aigc-ai\input --output-directory D:\aigc-ai\output --extra-model-paths-config D:\aigc-ai\extra_model_paths.yaml
```

如需手动前台启动：

```powershell
cd D:\aigc-ai\ComfyUI\repo
D:\aigc-ai\ComfyUI\venv\Scripts\python.exe main.py --listen 0.0.0.0 --port 8188 --lowvram --input-directory D:\aigc-ai\input --output-directory D:\aigc-ai\output --extra-model-paths-config D:\aigc-ai\extra_model_paths.yaml
```

### Docker 说明

当前没有 ComfyUI Docker 部署记录。不要用 `docker compose up` 作为本项目的默认启动方式，除非后续重新做 Docker 化部署并更新本文档。

### RIFE 补帧

```powershell
D:\aigc-ai\tools\rife-ncnn-vulkan-20221029-windows\rife-ncnn-vulkan.exe -i input_frames -o output_frames
```

### Real-ESRGAN 超分

```powershell
D:\aigc-ai\tools\realesrgan-ncnn-vulkan-v0.2.0-windows\realesrgan-ncnn-vulkan.exe -i input.png -o output.png -s 2 -n realesrgan-x4plus
```
