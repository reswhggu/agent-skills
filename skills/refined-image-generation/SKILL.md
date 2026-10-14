---
name: refined-image-generation
description: 使用开源扩散模型工具链（ComfyUI / Fooocus + Real-ESRGAN）生成高细节精细化图片。当用户要求生成、细化、重绘或放大增强图片时使用。
---

# 精细化图片生成

## 适用场景
- 根据文字描述生成高细节图片（文生图）
- 基于参考图细化/局部重绘（图生图 / inpaint）
- 放大并增强已有图片的细节（超分辨率）

## 工具链（开源项目）
| 项目 | 仓库 | 定位 |
|---|---|---|
| ComfyUI | github.com/Comfy-Org/ComfyUI | 首选：节点式工作流，ControlNet / LoRA / 分块放大，支持 API |
| Fooocus | github.com/lllyasviel/Fooocus | 快速出高质量图（内置 SDXL + 面部细节器） |
| Stable Diffusion WebUI | github.com/AUTOMATIC1111/stable-diffusion-webui | 经典界面，扩展生态丰富 |
| Real-ESRGAN | github.com/xinntao/Real-ESRGAN | 放大后的细节增强（超分辨率） |

## 一次性环境准备（NVIDIA GPU ≥ 8GB 显存；SDXL 建议 12GB+）
# ComfyUI
git clone https://github.com/comfyanonymous/ComfyUI
cd ComfyUI && pip install -r requirements.txt
python main.py --port 8188        # 访问 http://127.0.0.1:8188

# Real-ESRGAN（后处理放大）
pip install realesrgan

# 模型下载后放入对应目录（来源：Civitai / HuggingFace）
#   models/checkpoints/     底模（SDXL、Flux 等）
#   models/loras/           风格/细节 LoRA
#   models/controlnet/      ControlNet 模型
#   models/upscale_models/  放大模型（如 4x-UltraSharp）

## 精细化生成工作流
1. **提示词工程（核心）**：质量词前置（masterpiece, best quality, ultra detailed）；分层描述：主体 → 材质/纹理 → 光影 → 构图/镜头 → 风格；负面提示词：blurry, lowres, bad anatomy, extra fingers, watermark
2. **采样参数**：DPM++ 2M Karras 或 Euler a；30–50 steps；CFG 5–12（过高易过曝）
3. **结构控制**：锁定构图/姿势时用 ControlNet（canny / depth / lineart），避免重绘结构漂移
4. **细节增强**：加载 LoRA（权重 0.6–1.0）；脸/手用 ADetailer 或面部细节器节点自动修复
5. **两阶段放大（关键）**：1024×1024 出图 → hires.fix 放大 1.5–2x（denoise 0.3–0.45）→ Real-ESRGAN / 4x-UltraSharp 再放 4x（denoise 0.2–0.35）
6. **超大图**：用分块（tiling）工作流拼接，避免单次生成畸变

## 交付前检查清单
- [ ] 提示词含质量词 + 分层描述，负面提示词覆盖常见缺陷
- [ ] 最终分辨率 ≥ 1024px；更大尺寸已两阶段放大
- [ ] 关键区域（脸/手/文字）已过细节器或局部重绘
- [ ] 成品检查：无多指、无水印残留、无过度磨皮

## 常见问题速查
| 症状 | 处理 |
|---|---|
| 图糊、缺纹理 | steps 提到 40+，CFG 8–12；提示词加 highly detailed |
| 重绘后结构漂移 | ControlNet 锁结构；img2img denoise ≤ 0.5 |
| 脸/手崩坏 | ADetailer / 面部细节器节点；或局部 inpaint |
| 显存不足 | 启动加 --lowvram 或用 fp8 权重；先出小图再两阶段放大 |
