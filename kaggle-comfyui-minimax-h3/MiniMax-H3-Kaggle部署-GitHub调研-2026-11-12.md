# MiniMax H3（海螺 3.0）Kaggle 部署 — GitHub 调研 2026-11-12

Jason 指令：Kaggle T4x2 上部署 MiniMax H3；去 GitHub 找开源部署方案。之前做过：2026-11-09 抖音秋叶视频核查 + 2026-11-10/11 Colab 大视频模型调研（小薯 v2 首选 H3 W4A8 + Qwen3-VL-4B 小编码器，未验证）。

## 模型
- MiniMax H3 = Hailuo 3.0，33B 参数，2026-08-03 开源，发布当天即有 ComfyUI 原生支持（文生视频、图生视频、同 pass 立体声）。
- 许可：MiniMax H3 社区许可带地域条款，排除美国/欧盟/英国/韩国；中国境内自用无此问题。

## 官方本地部署路径（ComfyUI）
- ComfyUI ≥ 0.30.0，内置 H3 节点：MiniMaxH3ImageToVideo、MiniMaxH3ReferenceToVideo、VAEDecodeAudio、CreateVideo、SaveVideo。
- 官方工作流模板：Comfy-Org/workflow_templates 的 video_minimax_h3_t2v.json、video_minimax_h3_i2v.json（另有 api_minimax_h3_*.json 是付费 Comfy-Cloud API 节点，别混淆）。
- 权重：huggingface.co/Comfy-Org/MiniMax-H3。量化重打包：int8 convrot 扩散模型 + NVFP4 AWQ Qwen3-VL 文本编码器，磁盘约 40GB。
- 官方教程：https://docs.comfy.org/tutorials/video/minimax/minimax-h3

## T4 上已验证可行（关键证据）
- dmmdea/offload-harness《T4 closeout》（2026-08-27 实测）：H3 按官方模板接线、单张 T4 16GB partial-offload 跑通（9.8GB 显存 + 10.2GB 内存，141 个 lowvram patch，convrot_w4a4 原生算子确认）；文本编码器 qwen3vl_32b nvfp4（15.7GB）部分加载。结论：H3 达标（提示词遵循/多镜头/图生视频保真/音频设计胜过 LTX-2.5），速度慢约 2 倍。
- Sean-Michael/home-kubernetes-cloud：8GB 卡上靠 ComfyUI 动态显存/block-swap 从 64GB 系统内存流式加载（~40GB 磁盘），说明低显存也能跑、但极度依赖内存。
- 速度参考：M1 Max 上 0.4MP、3–5 秒片段要几分钟到几十分钟；T4 上只会更慢。Kaggle 每周 30 小时 GPU 额度要省着用。

## 先前结论（2026-11-09/11，仍有效）
- 「8G 显存即可跑通」偏乐观；社区实测底线 12GB，16GB 才稳。无卡本地部署不存在；真正无卡的路是 MiniMax 云端 H3 API（按秒计费）。
- H3 NF4 在 T4 上只出 288×160、22 帧——不推荐；要走 int8 convrot + turbo LoRA 路线。
- 小薯 v2 首选 H3 W4A8 + Qwen3-VL-4B 小编码器（编码器 15.7GB→5.37GB），唯一未证实是测试机 16GB 内存 vs 我们 12.67GB。

## Kaggle T4x2 部署要点
- 单张 T4 16GB，partial-offload 可跑 H3；Kaggle 磁盘须先核实能否装下约 40GB 权重，装不下就停。
- 对外访问沿用 Cloudflare Tunnel（与 FLUX 教程同法）。
- 先小步冒烟：5 秒小尺寸测试出片，再谈正式用。

## Kaggle 实测结果（2026-11-12）
- Jason 10:29 用 Google 登录 Kaggle（henry999999），notebook 已开、GPU T4x2 已勾选、联网已开、PyTorch 2.11.0+cu130、Tesla T4、内存 33.65GB。
- FLUX 博客一键脚本在 10:31 被中断（执行 56 秒，停在下载 flux1-schnell-Q4_K_S.gguf 刚开始），未继续。
- 磁盘实测：/kaggle/working 总计约 20GB，已用 815MB（只有克隆的 ComfyUI），可用约 19GB。
- H3 权重实测（HF Comfy-Org/MiniMax-H3）：扩散模型 int8 convrot 34GB、文本编码器 Qwen3-VL 32B NVFP4 15.7GB、视频 VAE int8 2.81GB、音频 VAE int8 1.99GB，最小一套约 54.5GB；单扩散模型 34GB 已超 19GB 可用。**结论：Kaggle 装不下官方量化 H3，按指令停下，未下载权重、未启动 ComfyUI、未建隧道；会话已停以省 GPU 时长，notebook 草稿已保存。**

## GGUF 小版能否塞进 19GB（2026-11-12 补查）
- Abiray pruned GGUF：Q3_K_M 8.9GB、Q4_K_M 11.6GB、Q5_K_M 14.1GB、Q6_K 16.7GB、Q8_0 21.6GB；Q4_K_M 为 16GB 显卡推荐档。
- udero turbo-baked GGUF（8 步蒸馏已融合）：Q4_K_M 11.42GB、Q5_0 13.92GB。
- 编码器：32B NVFP4 15.7GB 太大；小版 Qwen3-VL-4B INT4 约 5.37GB；32B GGUF Q2_K 8.49GB。
- VAE：视频 VAE int8 2.81GB + 音频 VAE 约 0.56GB ≈ 3.4GB。
- 核算（可用 19GB）：Q4_K_M（11.6）+ 4B 编码器（5.4）+ VAE（3.4）+ ComfyUI（约 1）≈ 21GB，超了；只有 Q3_K_M（8.9）+ 4B 编码器（5.4）+ VAE（3.4）≈ 18.7GB 能勉强塞进，但 Q3_K_M 是最低画质实验档、分辨率很低，不值得为我们 720P 需求部署。
- HyperFlow H3 Flash（9.1GB Q4_K_M）未在 16GB 显存上实测过、首次下载约 90GB，不适合。
- **结论：Kaggle 19GB 装不下可用的 H3；要么查 Kaggle 有无其他可写磁盘，要么换磁盘大的平台（Colab 约 100GB），要么改 Wan 2.2 5B（能塞进 19GB）。**

## 参考仓库
- Comfy-Org/ComfyUI（内置 H3 节点）、Comfy-Org/workflow_templates（官方 H3 模板）、Comfy-Org/MiniMax-H3（HF 权重）
- ai-models-lab/minimax-h3（H3 资源站 + ComfyUI 工作流生成器 + ComfyUI-MiniMaxH3 节点）
- dmmdea/offload-harness（T4 实测记录）
- artokun/comfyui-mcp（minimax-h3-video skill，本地权重路径说明）
- RouterGlock/minimax-h3-comfyui-video（LM Studio 插件，fl2va GGUF + 8 步 turbo LoRA）

## 部署结果（2026-11-12 11:12，博主 notebook 实测成功）

- **结论：H3 在 Kaggle T4x2 上跑通了。** 用 Jason 给的博主 notebook（kaggle-t4-x2-comfyui-minimax-h3.ipynb，单格脚本）部署成功，测试视频已生成并验看。
- **磁盘问题的解法：装到 /tmp/ComfyUI。** /tmp 所在 overlay 实测可用约 1038GB，/kaggle/working 的 20GB 上限不再是瓶颈。脚本自带磁盘检查（需约 38GB），/tmp 一次通过。
- **权重（5 文件，约 34.9GB）：** pruned Q4_K_M GGUF 11.4GB（leejet/MiniMax-H3-GGUF）+ 32B NVFP4 编码器 15.7GB + fp16 视频 VAE 5.2GB + 音频 VAE 0.6GB + Turbo 4 步 LoRA 2.0GB。文本编码器放 GPU1、DiT/VAE 放 GPU0（跨卡分配，非显存合并）。
- **测试视频：** MiniMax_H3_Kaggle_T2V 工作流，2 秒、0.10MP、16:9、32 帧、4 步 Turbo；约 5 分钟生成（加载模型约 3 分钟）。画面：雨夜街道橘猫行走、霓虹映水洼，与提示词一致；带雨声+脚步声音频。
- **唯一一次修复：** 把 .ipynb 导入原 notebook 两次失败（编辑器「Oops something went wrong」，会话 Fatal）；改新建空白 notebook 导入成功（新 notebook notebookac34fb9fa0，已重命名 kaggle-t4-x2-comfyui-minimax-h3）。
- **公开链接：** https://evaluated-well-recycling-firefox.trycloudflare.com（临时 trycloudflare，会话停即失效）。导入文件自带的博主旧链接 bingo-little-everyone-thoroughly.trycloudflare.com 不是本次的，不要用。
- **会话保持运行。** Kaggle 会话约 20 分钟，GPU 已用 00:29/30 小时。T2V 与 I2V 工作流均已保存在 ComfyUI 工作流列表中。
- **注意：** 博主脚本本身不自动跑生成，只搭好服务并预设好工作流；测试生成是在 ComfyUI 界面里手动跑的。脚本开头自述「没有实测当前 Kaggle 的生成速度和峰值内存」，本次实测补上了这块：低分辨率 2 秒约 5 分钟。
