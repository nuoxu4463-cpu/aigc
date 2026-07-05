# 本地 AI 部署适配与技术选型

更新时间：2026-06-24

## 1. 结论

当前两台机器适合做“Mac 总控 + Windows CUDA worker”的轻量本地生产线。

不建议把 playbook 中的 `comfy-flux`、`comfy-hunyuan`、`comfy-wan` 原样全部部署到本地。它们可以保留为目标 Worker 名称，但在现有硬件上应先降级为：

| playbook Worker | 当前落地版本 | 部署机器 |
| --- | --- | --- |
| `comfy-flux` | SD1.5 / SD1.5-ControlNet 分镜图、关键帧验证；高质量图走 Seedream 或云端 FLUX | Windows 本地 + API |
| `comfy-hunyuan` | 粗制视频不做本地文生视频，先用图片动效、Ken Burns、转场生成 animatic | Mac |
| `comfy-wan` | 现阶段不在本地部署 Wan/Hunyuan 大视频模型；疑难镜头走商业 API 或云 GPU | API / 云 GPU |
| `ffmpeg-worker` | 拼接、转码、审片包、平台版导出 | Mac |
| `rife-worker` | AI 补帧、轻量超分 | Windows |

第一阶段目标应是：脚本生成 6-8 张镜头图，Mac 生成 15-30 秒粗制视频，Windows 做 RIFE 补帧/必要超分，Mac 产出抖音待发布包。

## 2. 机器画像

### Mac

| 项目 | 当前配置 |
| --- | --- |
| 型号 | Mac mini |
| 芯片 | Apple M4 |
| CPU/GPU | 10 核 CPU，10 核 GPU |
| 内存 | 16GB 统一内存 |
| 可用空间 | 约 152GiB |

适合：调度、文案、Excel、Dify/n8n、Python 脚本、FFmpeg/AVFoundation 视频合成、素材整理、发布前审核。

不适合：长时间跑大视频生成模型、依赖 CUDA 的 ComfyUI 视频节点。

### Windows

| 项目 | 当前配置 |
| --- | --- |
| 系统 | Windows 10 Pro 19045 |
| CPU | Intel i7-7700，4 核 8 线程 |
| 内存 | 16GB |
| GPU | NVIDIA GTX 1060 6GB |
| 驱动/CUDA | Driver 560.94，CUDA 12.6 |
| C 盘 | 约 8.4GiB 可用，不适合放模型 |
| D 盘 | 约 316GiB 可用 |
| F 盘 | 约 106GiB 可用 |

适合：CUDA 轻量推理、SD1.5、ControlNet 小工作流、RIFE 补帧、Real-ESRGAN-ncnn-vulkan。

不适合：FLUX full、SDXL 常规生产、HunyuanVideo、Wan2.2 5B/14B 本地视频生成、批量高分辨率视频生成。

## 3. 模型与工具选型

### 3.1 文案、标题、Prompt

默认部署在 Mac，Windows 仅作为备用。

| 用途 | 首选 | 备用 | 说明 |
| --- | --- | --- | --- |
| 商品长标题、关键词、Seedream prompt | `qwen2.5:7b` via Ollama | `qwen2.5:1.5b` | playbook 已采用；中文结构化 JSON 稳，7B 质量优先，1.5B 批量初筛 |
| RAG 规则检索 | 本地 Markdown + 脚本检索 | 后续接 Dify 知识库 | 当前阶段不用复杂向量库也能跑通 |
| 脚本/分镜 JSON | `qwen2.5:7b` + 人工审核 | 商业 LLM API | 内容创意和合规门禁保留人工 |

不建议把 14B/32B 以上模型放到这两台机器作为批量主力。Mac 16GB 可以尝试，但会挤占视频合成和浏览器/剪辑资源；Windows CPU 太老。

### 3.2 商品图、分镜图、关键帧

| 用途 | 首选 | 备用 | 部署 |
| --- | --- | --- | --- |
| 商品三图生产 | Seedream API | SD1.5 仅做 POC | API + Mac 调度 |
| 白底商品保持 | 程序抠图后原商品回贴 | 不交给扩散模型重绘 | Mac/Windows |
| 分镜草图 | SD1.5 img2img / txt2img | Seedream / FLUX API | Windows ComfyUI |
| 构图控制 | SD1.5 ControlNet Canny/Depth/OpenPose | 手工分镜图 | Windows ComfyUI |

关键原则：商品包装、商标、中文标签不要交给扩散模型重绘。商品图生产应采用“生成背景 + 商品抠图回贴 + 程序加真实文案”的结构。

Windows 推荐模型：

| 模型 | 用途 | 原因 |
| --- | --- | --- |
| `v1-5-pruned-emaonly.safetensors` | SD1.5 基线验证 | playbook 已记录 SHA-256，可复现 |
| SD1.5 Inpainting | 局部修补、背景扩展 | 6GB 显存可承受 |
| SD1.5 ControlNet Canny/Depth/OpenPose | 控制构图和姿态 | 轻量、生态成熟 |
| RMBG / rembg 轻量模型 | 商品抠图 | 比扩散模型更适合保持商品真实性 |

暂不推荐本地部署：

| 模型 | 原因 |
| --- | --- |
| FLUX.1-schnell | 权重和显存压力大，GTX 1060 6GB 不适合；可作为云 GPU/API 路由 |
| FLUX.1-dev 系列 | 默认非商业许可，商业生产前需要额外授权 |
| SDXL | 6GB 显存会非常吃紧，生产效率低 |

### 3.3 粗制视频与 AI 补帧

当前工作流应把“本地视频生成”拆成两段：先用静态图生成粗制视频，再做补帧/转码。

| 阶段 | 首选工具 | 部署机器 | 说明 |
| --- | --- | --- | --- |
| 粗制视频 | Python/Pillow + AVFoundation/FFmpeg | Mac | 当前项目已有实现；稳定、可控、成本低 |
| 镜头动效 | Ken Burns、推拉摇移、遮罩、字幕 | Mac | 用确定性脚本做 animatic |
| 补帧 | RIFE / Flowframes / rife-ncnn-vulkan | Windows | GTX 1060 6GB 适合做补帧，不适合做大模型视频生成 |
| 超分 | Real-ESRGAN-ncnn-vulkan | Windows | 只对选中镜头做，避免批量浪费 |
| 最终合成 | FFmpeg 或 macOS AVFoundation | Mac | 输出无水印母版和抖音版 |

暂不建议在本地部署 HunyuanVideo-1.5 或 Wan2.2。现有 Windows 显存和内存都偏小，Mac 没有 CUDA；强行部署会把 MVP 变成环境折腾。

### 3.4 发布前处理

| 用途 | 工具 | 部署 |
| --- | --- | --- |
| 抖音 9:16 母版 | FFmpeg / AVFoundation | Mac |
| 字幕和贴片 | Python/Pillow 或剪映专业版 | Mac |
| 人工终审 | 剪映专业版 / QuickTime / Finder | Mac |
| 自动发布 | 暂不做 | 使用官方后台，人工确认 |

## 4. 推荐部署路径

### Mac

```text
/Users/xunuo/Documents/aigc
  storyboard/       分镜图与关键帧
  video/            粗制视频、帧序列、母版
  tools/            Python/Swift/调度脚本
  docs/             部署与工作流文档
```

Mac 侧建议补齐：

| 组件 | 用途 |
| --- | --- |
| FFmpeg | 转码、抽帧、响度、平台版本 |
| Node.js | 运行 playbook 里的 Excel/Dify 服务 |
| Ollama | qwen2.5 文案与 prompt |
| Python venv | 视频脚本、表格、文件调度 |

### Windows

不要把模型放 C 盘。本次实际部署使用：

```text
D:\aigc-ai
  ComfyUI\
  models\
    checkpoints\
    controlnet\
    loras\
    upscale_models\
  input\
  output\
  workflows\
  logs\
```

旧目录 `D:\ai` 已存在历史文件，本次没有改动。后续建议统一使用 `D:\aigc-ai`，避免旧环境污染新工作流。

Windows 侧建议部署：

| 组件 | 用途 |
| --- | --- |
| ComfyUI portable / 手动安装版 | SD1.5、ControlNet、API worker |
| PyTorch CUDA | 使用 GTX 1060 |
| RIFE / Flowframes | AI 补帧 |
| Real-ESRGAN-ncnn-vulkan | 轻量超分 |
| Python 3.10/3.11 | ComfyUI 和 worker 脚本 |
| Git | 拉取 ComfyUI 和节点 |

ComfyUI 启动参数建议：

```powershell
python main.py --listen 0.0.0.0 --port 8188 --lowvram
```

只在局域网内访问，不暴露公网。Mac 调用：

```text
http://192.168.0.102:8188
```

当前已落地：

| 组件 | 状态 |
| --- | --- |
| ComfyUI | 已部署到 `D:\aigc-ai\ComfyUI\repo`，不是 Docker 部署 |
| SD1.5 checkpoint | 已安装并通过 SHA256 校验 |
| 可视化 workflow | 已注册到 `workflows/aigc/product-main-sd15-img2img.json` |
| API workflow | 已注册到 `workflows/aigc/product-main-sd15-img2img-api.json` |
| 后台启动 | 已创建 Windows 计划任务 `AIGC-ComfyUI`，调用 `D:\aigc-ai\start-comfyui.cmd` |
| 冒烟测试 | 已成功生成 `D:\aigc-ai\output\product-main\695123659466_sd15_v01_00002_.png` |

## 5. MVP 路线

### 第 1 步：先跑通无大模型闭环

1. Mac 读取结构化脚本，生成 6-8 个 `shot_id`。
2. 使用现有分镜图或 Seedream API 生成镜头图。
3. Mac 生成 15-30 秒粗制视频。
4. Windows 对粗制视频做 RIFE 2x 补帧。
5. Mac 输出抖音 1080x1920 待发布包。

### 第 2 步：接入 Windows ComfyUI

1. 安装 SD1.5 baseline。
2. 导入 playbook 的 SD1.5 img2img API 工作流。
3. Mac 通过 HTTP 调 Windows ComfyUI。
4. 每次生成记录模型、seed、workflow、耗时、输出 hash。

### 第 3 步：加入商品图生产

1. Mac/Ollama 从 Excel 生成长标题、关键词和三路 Seedream prompt。
2. Seedream 生成背景/营销图。
3. 程序化回贴真实商品图和真实文字。
4. 人工审核商品描述和平台合规。

### 第 4 步：升级路由

只有当 MVP 连续稳定后，再引入：

| 目标 | 建议 |
| --- | --- |
| 高质量分镜 | 云 GPU FLUX.1-schnell 或商业图像 API |
| 本地高质量视频 | 另配 24GB+ NVIDIA GPU 工作站 |
| Hunyuan/Wan worker | 云 GPU 或新工作站 |
| 批量队列 | FastAPI + Redis/Celery + PostgreSQL |

## 6. 不建议清单

现阶段不要做这些事：

- 不在 Windows C 盘安装模型或生成缓存。
- 不在 GTX 1060 6GB 上硬跑 Wan/Hunyuan/FLUX full。
- 不把商品包装文字交给 SD1.5/SDXL/FLUX 重绘。
- 不在四 Worker MVP 跑通前采购多台 GPU。
- 不建设自动发布、自动评论、自动私信。
- 不同时安装大量 ComfyUI 自定义节点。

## 7. 当前优先级

优先级从高到低：

1. Mac 安装 FFmpeg、Node.js、Ollama，并把现有视频脚本整理成 `ffmpeg-worker`。
2. Windows 清理 C 盘到至少 20GB 可用，把 AI 环境固定到 D 盘。
3. Windows 安装 ComfyUI CUDA 版，先跑 SD1.5 baseline。
4. Windows 安装 RIFE/Flowframes，验证 1080x1920 粗制视频补帧。
5. Mac 写一个统一任务 JSON，记录 `content_id`、`shot_id`、`version_id`、模型、seed、耗时和输出文件。
