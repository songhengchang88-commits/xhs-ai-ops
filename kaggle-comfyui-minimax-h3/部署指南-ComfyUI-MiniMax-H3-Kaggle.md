# Kaggle T4x2 + ComfyUI + MiniMax H3 部署与生成指南

给电脑端智能体用：按这份指南在 Kaggle 上把 ComfyUI + MiniMax H3 跑起来，给五十六个民族系列做视频。

## 0. 已验证到哪一步（2026-11-12 实测）

- H3 在 Kaggle T4x2 上**已跑通**：部署单元首次运行约 10 分钟，测试视频 2 秒已生成并验看通过（雨夜街道橘猫行走、霓虹映水洼，与提示词一致，带雨声+脚步声音频）。
- 关键解法：模型装到 `/tmp/ComfyUI`。`/kaggle/working` 总共约 20GB，装不下 34.9GB 模型；`/tmp` 所在 overlay 实测可用约 1038GB。
- 用途：五十六个民族系列，每期 30 秒（3 镜头 × 10 秒），竖屏 9:16，配乐、无人物声音、无字幕。
- 参考教程（博主）：https://gaojilingjuli.blogspot.com/2026/11/ai-kaggle-gpu-comfyui.html
- 官方教程：https://docs.comfy.org/tutorials/video/minimax/minimax-h3

## 1. 准备

- **Kaggle 账号**：已注册并手机验证（账号 henry999999），每周 30 小时 GPU 额度。如需新号：注册 + 手机验证。
- **浏览器登录 Kaggle**（Google 登录即可）。
- **Notebook 文件**：本目录 `kaggle-t4-x2-comfyui-minimax-h3.ipynb`，单格 Python 脚本（463 行），直接用，不要改。
- **不要把 Kaggle 账号密码、API 密钥写进 Notebook 或本指南。**

## 2. 新建 Notebook 并导入

1. 在 Kaggle **新建一个空白 Notebook**，再把 `.ipynb` 文件内容导入。
   - 注意：往旧 Notebook 里导入 `.ipynb` 实测两次失败（编辑器报错、会话 Fatal）；新建空白再导入一次成功。
2. 设置：Settings > Accelerator > **GPU T4 x2**；**Internet 开启**。
3. 确认硬件：两张 Tesla T4，每张 14.56 GiB；内存约 33.65GB；PyTorch 2.11.0+cu130（以实际显示为准）。
4. 正确的 Notebook 地址形如：`https://www.kaggle.com/code/henry999999/kaggle-t4-x2-comfyui-minimax-h3/edit`。ID 形式的旧地址会显示 Not Found，不要用。

## 3. 运行部署单元

运行唯一的一个代码单元。脚本分 6 步：

| 步骤 | 内容 |
|---|---|
| 1/6 | 检查实际 GPU、内存和旧服务 |
| 2/6 | 更新 ComfyUI 和 H3 专用 GGUF 插件 |
| 3/6 | 安装依赖，保留 Kaggle 自带 CUDA PyTorch |
| 4/6 | 下载 H3 全套权重，支持断点续传和 SHA-256 校验 |
| 5/6 | 保存官方 H3 模板的双卡 GGUF 适配副本，启动服务 |
| 6/6 | 创建 / 复用 Cloudflare 临时访问链接 |

- 首次运行**约 10 分钟**（实测 576 秒；重启重跑约 546 秒）。等输出出现「节点和服务就绪」和 ComfyUI 访问网址。
- 脚本自带磁盘检查（模型需求量约 38GB，`/tmp` 可用约 1038GB，一次通过）。如果磁盘检查失败，停下，不要硬跑。
- 脚本本身**不自动跑生成**，只搭好服务并预设好工作流；生成在 ComfyUI 界面里手动跑（见第 6 节）。

## 4. 模型清单（5 个文件，约 34.9GB，装在 /tmp/ComfyUI）

| # | 来源仓库 | 文件 | 大小 | 作用 |
|---|---|---|---|---|
| 1 | leejet/MiniMax-H3-GGUF | minimax_h3_fl2va_pruned-Q4_K_M.gguf | 11.42 GB | 扩散模型 DiT（pruned Q4_K_M） |
| 2 | Comfy-Org/MiniMax-H3 | text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors | 15.69 GB | 文本编码器 Qwen3-VL 32B NVFP4 |
| 3 | Comfy-Org/MiniMax-H3 | vae/minimax_h3_video_vae_fp16.safetensors | 5.21 GB | 视频 VAE（fp16） |
| 4 | Comfy-Org/MiniMax-H3 | vae/minimax_h3_audio_vae_fp32.safetensors | 0.61 GB | 音频 VAE（fp32） |
| 5 | Comfy-Org/MiniMax-H3 | loras/minimax_h3_fl2v_turbo_4step_v1.0_768p_comfyui_bf16.safetensors | 1.96 GB | Turbo 4 步 LoRA |

合计约 34.89 GB。脚本自动下载并放到 `/tmp/ComfyUI/models/` 对应子目录，不用手动放。下载支持断点续传和 SHA-256 校验（3 次重试）。

**跨卡分配**：DiT / VAE → GPU0，文本编码器 → GPU1。这是跨卡分配组件，不是显存合并。

## 5. 拿到 ComfyUI 公开链接

- 部署单元跑完会打印 **ComfyUI 访问网址**（形如 `https://xxx.trycloudflare.com`）。
- 这是 Cloudflare 临时隧道链接，**每次会话都不一样，会话停即失效**。不要用别人旧的链接（博主文件里自带的旧链接已失效）。
- 在浏览器打开该链接进入 ComfyUI。
- 注意：如果打开链接看到 ComfyUI 默认工作流报「缺少模型配置文件」，不用慌——那是浏览器本地默认的文生图工作流（它要的 SD1.5 检查点本来就没装），**不影响 H3 生成**。加载下面的 H3 工作流即可；看右上角队列面板确认生成状态。

## 6. 生成视频（以蒙古族第 2 期为例）

- **工作流**：在 ComfyUI 的 Workflows 里加载 `MiniMax_H3_Kaggle_T2V`（文生视频；另有 `MiniMax_H3_Kaggle_I2V` 图生视频）。两个工作流都已保存在 ComfyUI 工作流列表中。
- **参数（每镜头）**：时长 **10 秒**、**0.4MP**（约 480p）、竖屏 **9:16**、**Turbo 4 步**。
  - 注意：脚本预设是 2 秒 / 0.1MP 的冒烟测试值，正式生成要在工作流里改成 10 秒 / 0.4MP / 9:16 / 4 步。
- **提示词**：见本目录 `视频提示词-蒙古族-草原童年-2026-11-12.md`，3 个镜头提示词全文照抄进提示词框；人物锚点三镜头逐字重复，不要用相对指代。
- **依次跑 3 个镜头**：先跑完一个再排下一个，队列里一次一个任务，不要同时排 3 个。
- **速度实测**：0.4MP × 10 秒一个镜头约 1.5 小时（采样每步约 21 分钟，4 步）。一期 3 镜头约 4.5 小时。每天用 ComfyUI 4–5 小时（不到 5 小时），每周 30 小时额度要省着用。
- **下载**：每个镜头跑完后在 ComfyUI 界面下载，或用 API（见第 7 节）。
- **拼接**：3 段硬切拼成 30 秒（ffmpeg concat），不加转场特效。
- **自审**：五项一票否决（人物走样、动作穿帮、AI 味重、配音口型不对、节奏拖沓），全片通看；不过重做，两次不过换做法。
- **音乐**：每镜头由 H3 生成同风格配乐（马头琴、温暖怀旧），无人物声音、无字幕。

## 7. 查状态（不用进界面也能查）

用 curl 直接请求隧道地址（不要带浏览器 Sec-Fetch 头）：

- 队列：`GET <隧道地址>/queue`（看 `queue_running` / `queue_pending`）
- 历史：`GET <隧道地址>/history`（已完成的文件名）
- 服务：`GET <隧道地址>/system_stats`

注意：隧道对跨站 iframe 请求会返回 403，页面里嵌不进去；直接用链接在浏览器打开，或用 curl 查 API。

## 8. 排错

- **会话被顶掉**：不要在第二台设备上打开同一个 Notebook——会把正在跑的会话顶掉，生成丢失。只用一台设备操作。
- **隧道 429（Tunnelstore request failed）**：Cloudflare 临时隧道限流/断了。重跑部署单元拿一个新链接；不要反复刷旧链接。隧道链接每次会话都不一样。
- **隧道打不开（连接失败）**：隧道可能已断。重跑部署单元拿新链接。如果会话也停了，先重启会话再重跑。
- **会话自己停了（Draft Session off）**：Kaggle 会话是临时的，自己会停。点 Start session 重启会话，重跑部署单元（约 10 分钟）。脚本会逐文件校验 SHA，已有的模型直接复用、缺的才下载；如果 `/tmp` 被清空，5 个模型（约 34.9GB）要重新下载，耗时更长。
- **/tmp 磁盘不够**：不要装到 `/kaggle/working`（总共约 20GB，装不下）。模型必须装 `/tmp/ComfyUI`（overlay 约 1038GB）。脚本自带磁盘检查，失败会直接停，不要绕过。
- **导入 Notebook 失败**：不要往旧 Notebook 里导入 `.ipynb`（实测两次失败）。新建空白 Notebook 再导入。
- **打开链接报缺少模型配置**：见第 5 节，是默认工作流的问题，不影响 H3 生成；加载 `MiniMax_H3_Kaggle_T2V` 工作流即可。

## 9. 许可与注意事项

- MiniMax H3 社区许可带地域条款，排除美国/欧盟/英国/韩国；中国境内自用无此问题。
- Kaggle 每周 30 小时 GPU 额度；每天用 4–5 小时（不到 5 小时）。
- 会话是临时的：停了之后隧道失效，`/tmp` 可能被清空；每天开工先重启会话、重跑部署单元（约 10 分钟），再生成。
- 不要把 Kaggle 账号密码、API 密钥写进 Notebook 或本指南。

## 10. 本目录文件清单

| 文件 | 说明 |
|---|---|
| 部署指南-ComfyUI-MiniMax-H3-Kaggle.md | 本文件 |
| kaggle-t4-x2-comfyui-minimax-h3.ipynb | 部署脚本，单格，463 行 |
| 视频提示词-蒙古族-草原童年-2026-11-12.md | 第 2 期（蒙古族）3 镜头提示词全文 |
| 五十六个民族清单-2026-11-12.md | 56 期顺序与制作约定 |
| MiniMax-H3-Kaggle部署-GitHub调研-2026-11-12.md | 调研与实测记录 |

参考：博主教程 https://gaojilingjuli.blogspot.com/2026/11/ai-kaggle-gpu-comfyui.html ；官方教程 https://docs.comfy.org/tutorials/video/minimax/minimax-h3 ；权重 https://huggingface.co/Comfy-Org/MiniMax-H3
