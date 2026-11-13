# 视频生成路线（拼好的，2026-11-13）

> 方法：逆向拼接（Jason 2026-11-13 定）。不找官方，找别人开源跑通的社区实现，逐件对我们的条件验，拼一条适合我们走的路。不必照搬，可自己另拼。

## 我们的条件

- 平台：Kaggle T4x2（账号 henry999999，每周 30 小时 GPU）
- 目标：每天 2 期、每期 3 镜头、每镜头 10 秒、720×1280 竖屏、I2V（首帧图保人物一致性）、配乐无人声
- 注意：WanGP 只用一块卡，T4x2 不合并显存；Kaggle 的优势是额度（30 小时/周）+ 会话最长 9 小时

## 拼好的路线

| 部件 | 选什么 | 来源 | 为什么适用我们 |
|---|---|---|---|
| 底子 | kayas881/WanGP-Kaggle（wangpp.ipynb） | GitHub 社区，19★ | 唯一现成的 Kaggle + WanGP + T4 包装；profile 4、SageAttention v1.0.6、mmgp、4 步 Lightning LoRA、ckpts 软链 /kaggle/tmp；装完 cells 1–5 必须重启内核 |
| 模型 | Wan 2.2 TI2V-5B FastWan 3 步蒸馏版 | FastVideo，Apache 2.0；GGUF Q8_0 约 5.4GB | Square-Zero-Labs 有录屏证明：唯一在免费 T4 的 WanGP 里加载到 121×704×1280 的模型 |
| 界面 | WanGP（不用 ComfyUI） | 社区实测 | 同一模型同一 T4、5 秒片段：ComfyUI 17 分钟，WanGP 3 分钟，快 5.6 倍 |
| 装机 | uv 装依赖、复用现成 torch、删 xformers | Square-Zero-Labs/Wan2GP-on-Colab（196★、81 fork、Apache-2.0） | T4 上 xformers 根本不用；uv 快得多；Drive/持久化可跨重启保留 |
| 模型存放 | Kaggle Dataset + 软链 | kelvinweijun/wan-2.2-animate-comfyui-kaggle | 会话一停就清盘，Dataset + 软链不用每次重下 10GB；Gradio API 比隧道稳 |
| 验收 | 固定一张测试图 + 固定提示词横评；I2V 必须先验证跟不跟首帧图 | rupeshofficedata/minimax-h3-comfyui-lowvram | Turbo 4 步版已被他实测证实会完全忽略输入图（8–9 分钟白跑）；FastWan 的 I2V 必须冒烟验证 |
| 量化 | GGUF Q8_0（甜点，近无损）；Q4 能用；Q3 别用 | subrit/ComfyUI-Wan2.2-workflow + 三路交叉确认 | Q8_0 是画质甜点 |

## 拼的时候扔掉的

| 扔掉的 | 原因 |
|---|---|
| MiniMax H3（33B） | T4 实测 80 分钟/10 秒且崩；已弃用（Jason 2026-11-13 定） |
| Wan 2.2 TI2V-5B 基础版（50 步） | T4 上 10 秒 720p 约 40–80 分钟，赶不上每天两期 |
| LTX-2.5 | 22B、下载 66GB，480p 10 秒约 30 分钟；太重太慢 |
| ComfyUI 路线 | T4 上比 WanGP 慢 3–6 倍 |
| 480p 当最终输出 | 民间诚实档位是 480p，但我们要 720×1280（小红书上要清晰） |
| fecsaba/wan2.2-google-colab、HaremOverlord/Wan2GP-Colab-Notebook | 0–2★、未经证实、玩具基准 |
| H3 的参考图玩法（模型本身） | 思路可借（多参考图 + 标签引用保人物），但 H3 已弃用；思路搬到 Wan 2.2 首帧图上 |

## 还剩一个缺口（只能实测补）

没有任何一家民间笔记本公布过“T4、720×1280、10 秒、FastWan 3 步”的实测墙钟时间。Kaggle 上的 FastWan 冒烟测试跑出的 720p 时间就是这个答案。冒烟顺序：先 5 秒 480p，再试 720×1280；I2V 首帧锚定保服装一致性。

## 社区来源清单

- kayas881/WanGP-Kaggle — https://github.com/kayas881/WanGP-Kaggle（无 LICENSE，注意：只自用、不转售）
- Square-Zero-Labs/Wan2GP-on-Colab — https://github.com/Square-Zero-Labs/Wan2GP-on-Colab（Apache-2.0；196★）
- kelvinweijun/wan-2.2-animate-comfyui-kaggle — https://github.com/kelvinweijun/wan-2.2-animate-comfyui-kaggle（无 LICENSE）
- rupeshofficedata/minimax-h3-comfyui-lowvram — https://github.com/rupeshofficedata/minimax-h3-comfyui-lowvram（无 LICENSE；Turbo 忽略输入图的实测记录）
- subrit/ComfyUI-Wan2.2-workflow — https://github.com/subrit/ComfyUI-Wan2.2-workflow（Unlicense）
- jigs10/ltx-2.5-kaggle-wan2gp — https://github.com/jigs10/ltx-2.5-kaggle-wan2gp（LTX-2.5 实测时间；WanGP 只用一块卡）
- FastWan 权重 — https://huggingface.co/FastVideo/FastWan2.2-TI2V-5B-FullAttn-Diffusers（Apache 2.0）
- WanGP — https://github.com/deepbeepmeep/WanGP（自定义许可：本地免费、产出可商用，不得转售 WanGP 本身）

## 状态（2026-11-13）

- Kaggle + WanGP + FastWan 冒烟测试进行中（浏览器任务），先 5 秒 480p 再试 720×1280，验证 I2V 跟图。
- Colab 路线已停（T4 连不上，用量限额），改走 Kaggle 优先。
- 旧的 Kaggle ComfyUI + H3 部署包（xhs-ai-ops 的 kaggle-comfyui-minimax-h3/）已过时，不再用。
