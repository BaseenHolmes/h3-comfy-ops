---
name: h3-comfy-ops
display_name: "H3 视频生成运维（4090 算力机）"
description: "在局域网 RTX 4090 (192.168.0.15) 上跑 ComfyUI + MiniMax H3 视频生成：选方案、调参数、排障、回滚、SageAttention 修复、量化底模（int8/int4/w6a8/GGUF）对照。用户说「跑 H3 视频/这条片子怎么调/画质糊/显存不够/ComfyUI 起不来/sage 不生效」时使用。"
version: 1.3.0
source: own
category: ops
---

# H3 ComfyUI 运维（192.168.0.15）

> 两批数字**别混用**：**深夜批** = 1344×768×73、int4、干净环境（§2.1）；**下午批** = 704×1248×73、0.35.0、开 SageAttention（§2.2）。同表内只做同条件对照，**跨分辨率/跨轮次不换算**。
> **旧耗时（60.8 / 114.6 / 206.0 / 53.0 s …）一律作废**：当时后台有 IndexTTS + 双 ComfyUI 抢显存，同图同 seed 慢 **+50%**（45.5 → 68~70 s）。对照见 `references\measured_baselines.md` §5。

## 0 每次开工先做这 4 步（顺序不能换）

~~~powershell
# ① 确认抢显存的进程没起来 —— 本机最大的一笔收益（+50% 耗时就是它）
ssh Administrator@192.168.0.15 "netstat -ano | findstr :7860"
#    常态应由计划任务 H3TtsGuard（SYSTEM，开机 + 每 2 分钟）自动压回：任务 Running 就 /end、
#    未 Disabled 就 /disable、并杀 webui.py 进程。手工只需确认 7860 无监听 + 显存 ~1.4GB。
#    守卫日志 C:\Users\Administrator\h3opt\tts_guard.log；要放行语音就在目标机建 D:\AI\tts\ALLOW_TTS（tts_on.bat 会做）
#    为什么不是"删任务"：2026-11-09 实测该任务被环境反复重建（13:06 删 → 16:18 又起 → 17:21 禁用 → 17:23 又 Running），删了会被重建，只能持续压（§4 第 2 条）
# ② 确认没有第二个 ComfyUI 抢卡（0.44 默认应停着）+ 队列空（8188 无鉴权，先确认是自己的任务）
ssh Administrator@192.168.0.15 nvidia-smi --query-gpu=memory.used,utilization.gpu --format=csv
curl.exe -s http://192.168.0.15:8188/system_stats
curl.exe -s http://192.168.0.15:8188/queue
# ③ 提交后 10 秒看一次功耗
ssh Administrator@192.168.0.15 nvidia-smi --query-gpu=utilization.gpu,power.draw --format=csv
# ④ sage 是否在跑：0.35 启动器已带 --use-sage-attention（§5 修好），图里不必再挂节点
~~~

显存判读：刚重启 ≈441 MiB；已跑过任务 ≈16 GB 是它自己缓存的 int4 权重（正常）。判干净看端口：`netstat :7860` 无监听 + 无 `webui.py` 进程。判在算：**>200W = 在算；100% 利用率 + <200W = 卡死**。

## 1 环境事实

| 项 | 值 |
|---|---|
| 目标机 | **192.168.0.15**（BF-202502191514），Administrator **免密 ssh**（BatchMode 可用）；操作机 192.168.0.25 无 GPU |
| GPU | RTX 4090 D，**24,564 MiB**，驱动 617.42，功耗墙 425W；磁盘 D 剩 209GB / C 剩 107.5GB，模型约 152GB；内存 65.3GB（可用约 21GB） |
| **0.35（生产）** | 根 `D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\`；端口 **8188** 无鉴权；ComfyUI **0.35.0** · Python 3.13.14 · torch **2.13.0+cu130** · triton-windows 3.8.0.post29 |
| 0.35 启动/重启 | `C:\Users\Administrator\h3opt\start_comfy.bat`（已带 `--use-sage-attention`）+ 计划任务 **H3Comfy**（InteractiveToken）；日志 `h3opt\comfy.log`，**用 `>` 覆盖写 → 重启会清掉上次崩溃日志** |
| **重启唯一可靠方式** | `schtasks /run /tn H3Comfy`（0.35）/ `H3Comfy044`（0.44）。经 ssh 用 `Start-Process` 起的进程会随 ssh 断开被带走 |
| **0.44（按需）** | ComfyUI **0.44.0**，根 `D:\moxing\ComfyUI_044\ComfyUI_windows_portable`；端口 **8189**；torch 2.14.1+cu130；`h3opt\start_044.bat` + 计划任务 **H3Comfy044**，日志 `h3opt\comfy044.log`；**默认停着**；独立 models 目录，**无 KJNodes**（没有 `PathchSageAttentionKJ`，**sage 在 0.44 上装不起来**，见 §5）；**已装 ComfyUI-GGUF**（`UnetLoaderGGUF` 可用）；自带原生 `MiniMaxH3SigmaShift` / `MiniMaxH3FunControlNetApply` / `GetVideoComponents`；**默认开 Comfy 模型编译器**（`--disable-comfy-compiler` 可关，开时日志有 `Comfy model compiler graph breaks: 22, rogues: 1`） |
| models 根 | `...\ComfyUI\models\`：`unet\` `vae\` `loras\` `text_encoders\` `model_patches\`（ControlNet Union 在这） |
| 输入 / 输出 | 参考图 `...\ComfyUI\input\`；产物 `...\ComfyUI\output\`（正式）与 `...\output\h3opt_ab\`（实验） |
| IndexTTS | webui 7860 / tts_server 7861 + BootTrigger 任务 `IndexTTS_WebUI` / `IndexTTS_API`。**会被环境反复重建**（不是普通"重启自启"）→ 已由守卫任务 **H3TtsGuard** 每 2 分钟压回（§4 第 2 条）；开工只需确认 7860 无监听 |
| SageAttention | **已修好可用**（2.2.0 + `python313.lib` + `Python.h`，§5）；0.35 走启动参数，图里也可插 `PathchSageAttentionKJ("auto")` |

**底模四档**（详细字节数/文件名见 `references\sage_and_quant.md` §2）：int4 `MiniMax_H3_Ref2VA_pruned_mixed_int4_int8_convrot.safetensors` **14.06GB 默认** ｜ int8 `minimax_h3_ref2va_pruned_int8_convrot.safetensors` 19.53GB 回滚档 ｜ w6a8 `minimax_h3_ref2va_pruned_w6a8.safetensors` **15,983,746,636 B 仅 0.44** ｜ GGUF `MiniMax-H3-Ref2VA-Pruned-Q4_K_M.gguf` 11.56GB（73 帧慢 43%）。LoRA 在 `models\loras\`：turbo 4step（4/8/12/20 步都靠它）、`pika_h3_flex_ref2va_v1_bf16` 1.82GB、画质三件套 `h3-realism-people`（触发词 `r34l1sm`）/`Motion_Repair_V2`/`minimax-h3-facial-realism`；细节 VAE `vae\MiniMax-H3-X2-Detail-v1_int8_convrot.safetensors` 2.70GB；CN `model_patches\...controlnet_union_pruned_int8_convrot.safetensors` 2.19GB。

## 2 决策矩阵（先查表，再动手）

### 2.1 主矩阵 · 1344×768×73 · int4 · 干净环境（2026-11-09 深夜实测）

统一条件：int4、1344×768（除注明）、24fps、`VAEDecodeTiled`、`CreateVideo(24)→SaveVideo`、`BasicGuider`（等价 CFG=1）。body 全在 `D:\AgentHome\minimax-h3-review-20261106\exec\`。

| 场景 | LoRA | 步数 / 采样器 | 分辨率 × 帧 | 实测耗时 | 现成 body |
|---|---|---|---|---|---|
| 冒烟验图（新图先跑） | turbo | 4 步 euler/simple | 512×512×33 | **未测**（把现成 body 的 width/height/length 改小） | `Q4_int4_73.json` |
| **单人室内光足** | turbo | **4 步 euler/simple** | 1344×768×73 | **45.5 / 45.6 s** | `B035_2.json` |
| **双人对话背景干净** | turbo | **4 步 res_multistep/beta** | 1344×768×73 | **45.8 s**（脸崩上 8 步 **83.4 s**） | `Q4_int4_73.json` → `L8e.json` |
| **夜景 / 人群 / 多人脸** | **Pika Flex** | **6 步 rm/beta** | 1344×768×73 | **65.0 s**（≈turbo 8 步观感，省 18 s）；再细 12 步 **115.1 s** | `PK6_rm_int4_73.json` / `Q12c_int4_73.json` |
| **长视频 >124 帧** | turbo | 4 步 euler/simple | 73 + 51… | 73 帧 **45.5 s**；51 帧**未测**；124 帧生产链 **157.9 s**（无 sage）/ **103.2 s**（开 sage，§2.3） | `S1_seg73.json`+`S2_seg51.json` / `PRODc.json` |
| **带参考图/音频** | turbo | 4 步 euler/simple | 1344×768×**124** + 2 图 + 1 音频 | **157.9 s**（未开 sage；同条件开 sage 未复测整链） | `PRODc.json` |
| **要控构图（ControlNet）** | turbo + CN Union | 4 步 rm/beta | 704×1248×73（**只能 0.44**） | **64.0 s**（同 seed 无 CN **39.7 s**，**+61%**）→ 能跑但**控制力不可信** | `CN704_044.json` / `CN704_044_nocn.json` |
| 要 2K 输出 | turbo | 4 步 euler/simple | 1152×2048×33（0.44） | **67.2 s**（X2 细节 VAE 那条**未测**） | `V2K_044.json` / `B2b_x2_33.json` |
| 要新能力（Sol-H3-VAE / context windows / latent upscaler） | — | — | — | **未测**（需 0.44） | — |

### 2.2 竖屏 + SageAttention · 704×1248×73 · 0.35.0（2026-11-09 下午实测）

| 场景 | 配置 | 耗时 | 备注 |
|---|---|---|---|
| 高质量满血 | 20 步 rm/beta，无 sage | **188.7 s** | 对照基线 |
| ↳ 开 sage | 同上 + `PathchSageAttentionKJ` | **111.9 s（-40.7%）** | PSNR **23.0 dB**（20 步属正常累积差异），目视无差别 |
| 快出片 | 4 步 turbo LoRA，无 sage | **72.5 s** | 对照基线 |
| ↳ 开 sage | 同上 + KJ 节点 | **60.6 s（-16.4%）** | PSNR **37.7 dB** |
| **最快（首选）** | 4 步 turbo LoRA + 全局 `--use-sage-attention`（图里无节点） | 冷 **83.4 s** / 热 **43.2 s** | 优先用这条：省时间也省得改图 |
| GGUF 档 | Q4_K_M + 全局 sage | 冷 **75.9 s** / 热 **61.6 s**，峰值 **23,719 MiB** | 比 int4 4 步热跑慢约 **43%**，峰值**没降** → 73 帧不推荐 |
| 0.44 全部档位 | 4 / 8 / 12 / 20 步；另测 1152×2048×33 与 704×1248×51（4 步） | **40.2 / 67.6 / 84.1 / 195.8 s**；**67.2 / 32.6 s** | 比 0.35 略快，**没有实质提速**（20 步首含冷启） |
| w6a8（0.44） | 4 步 73 帧 | 冷 **68.5 s** | 0.35 加载直接报 `KeyError: 'w6a8_int8'` |

- **sage 免费提速，默认就该开**：省 16~40%，PSNR 37.7 dB（4 步）/ 23.0 dB（20 步），目视无差别。
- **GGUF 只在「要腾权重显存换更高分辨率/更长帧」时考虑**：73 帧档慢 43%，峰值显存不降（瓶颈是激活值不是权重）。
- **节点缓存会造假耗时**：同图同 seed 重跑 **0.1 s** 返回，测速必须换 `noise_seed` 或 `filename_prefix`。

### 2.3 0.35 vs 0.44 同图终测（2026-11-09 17:00，int4 + turbo4step，每例单独提交）

| 用例 | 0.35 + sage | 0.44 关编译器 | 0.44 开编译器（默认） |
|---|---|---|---|
| 704×1248×73 4 步 | **42.8 s** | 69.6 s（热）/ 77.7 s（冷） | **40.2 s**（冷 50.4 s，含编译） |
| 1344×768×73 4 步 | 55.1 s | 54.7 s | 52.2 s |
| 1344×768×73 20 步 rm/beta | **151.2 s** | 192.2 s | 181.6 s |
| **1344×768×124 帧 4 步** | **103.2 s**（峰值 23,331MiB，**不 OOM**） | — | — |
| GGUF Q4_K_M 1344×768 4 步 | 86.8 s（+58% vs int4） | 83.4 s（+52%） | — |
| w6a8 1344×768 4 步 | 加载即 `KeyError: 'w6a8_int8'` | 55.9 s（与 int4 同速，无收益） | — |

- **结论：不迁移 0.44**。0.35+sage 与 0.44+编译器打平（704 42.8 vs 40.2 s），20 步反而 0.35 快 17%（151.2 vs 181.6 s）；0.44 还要吃 sage 不可用 + 双目录运维。
- 0.44 关编译器时 704×1248 比 1344×768 还慢（69.6 vs 54.7 s），开编译器恢复（40.2 vs 52.2 s）——eager 路径该尺寸劣化，冷热两次一致。
- **124 帧可达**：4 步 + sage + int4 下 1344×768×124 帧 103.2 s / 23,331MiB，生产 20 步基线 599.8 s → **5.8×**。§8「>124 帧才拆段」仍然成立，但 124 帧本身不必拆。

## 3 怎么跑一次生成

`exec\` 里新一批 body 是**完整 HTTP body**（顶层 `prompt` + `client_id`），直接 `--data-binary @文件`：

~~~powershell
# 1) 提交（换 body 文件即换方案；提交前换 noise_seed，否则命中缓存 0.1 s 返回）
curl.exe -s -X POST http://192.168.0.15:8188/prompt -H "Content-Type: application/json" --data-binary "@D:\AgentHome\minimax-h3-review-20261106\exec\B035_2.json"
# → {"prompt_id":"xxxx","number":1,"node_errors":{}}
# 2) 轮询（status.completed 才有 outputs；长时间不动按 §4 判卡死）
curl.exe -s "http://192.168.0.15:8188/history/<prompt_id>"
# 3) 取产物（subfolder = filename_prefix 斜杠前那半段）
curl.exe -s -o .\out.mp4 "http://192.168.0.15:8188/view?filename=B035_2_00001_.mp4&subfolder=h3opt_ab&type=output"
~~~

- 产物落 `...\ComfyUI\output\h3opt_ab\<prefix>_00001_.mp4`。
- 老 body（`PROD_int4_4step.json`、`A1_*.json`、`B1_*.json`）是裸节点字典，自己包一层 `@{prompt=$wf; client_id='dsh-h3'}`；新 body 不用包。
- 其它 API：`GET /object_info`（查真实参数名，别猜）、`POST /free`（`{"unload_models":true,"free_memory":true}`）、`POST /interrupt`（对卡死无效）。

## 4 必知的坑（现象 → 判据 → 处置）

> **远程写 .ps1 的三条硬约束**（全是本轮实测踩出来的）：
> ① **内容必须纯 ASCII**（注释也别写中文）：UTF-8 中文经 ssh 落地后被 PS 5.1 按 ANSI 读，多字节序列会吞掉换行，报 `缺少语句块` / `意外的标记 '}'` 这类**假语法错**，报错行号还指在下一行。
> ② **反斜杠要双写**：PowerShell 双引号串里写 `C:\\Users\\...`（到远端才是单反斜杠）；写单引号串反斜杠会被 ssh 吃掉，报"找不到批处理文件"。能用正斜杠就用 `C:/Users/...`。
> ③ **ssh 里 `Start-Process` 起的进程随连接断开被杀**：要常驻必须注册计划任务。

| 坑 | 现象 | 判据 | 处置 |
|---|---|---|---|
| **后台进程抢显存（第一坑）** | 同图同 seed 慢 50%：68~70 s vs 45.5 s；>73 帧跑不完 | 空闲显存 **11,745 MiB**（应 1,430） | ①确认 7860 无监听 ②`nvidia-smi` 看 used 是否 ~1.4GB ③0.44 按 PID 杀（§6） |
| **IndexTTS 会被环境反复重建** | 停一阵又占 10.3GB；**删任务、`/change /disable` 都会被撤销**（实测 13:06 删 → 16:18 又起 → 17:21 禁用 → 17:23 又 Running，任务 XML `<Date>` 三度变化） | `schtasks /query /tn IndexTTS_WebUI /fo LIST` 能查到 + 7860 有监听 | **常态靠守卫** `H3TtsGuard`（`C:\Users\Administrator\h3opt\tts_guard.ps1`，每 2 分钟：Running→`/end`、未 Disabled→`/disable`、杀 `webui.py`；有动作写 `tts_guard.log`）。要放行语音 → 建 `D:\AI\tts\ALLOW_TTS`（`tts_on.bat`）。撤守卫 → `schtasks /delete /tn H3TtsGuard /f`。`/end` 只杀 cmd 外壳 → `netstat -ano \| findstr :7860` 取 PID 再 `taskkill /PID <pid> /T /F` |
| **节点缓存 → 假耗时** | 复跑 **0.1 s** 返回，误判「提速」 | 与首跑同 seed 比 | 每次实测**换 seed 或 filename_prefix**；偏差幅度未测 |
| **`start_comfy.bat` 覆盖日志** | 重启后 `comfy.log` 只剩这一次，崩溃栈丢了 | 文件里没有上次报错 | 先 `copy C:\Users\Administrator\h3opt\comfy.log C:\Users\Administrator\h3opt\comfy_old.log` |
| **100% 利用率 + <200W = 卡死** | 不报错，进度不动 | `nvidia-smi --query-gpu=utilization.gpu,power.draw --format=csv` → 100% 且 <200W | `netstat -ano \| findstr :8188` 取 PID → `taskkill /PID <pid> /T /F` → `schtasks /run /tn H3Comfy` |
| **重启只能靠计划任务** | ssh 里 `Start-Process` 起的进程，ssh 一断就没了 | `schtasks /query /tn H3Comfy /fo LIST` 存在且 InteractiveToken | `schtasks /run /tn H3Comfy`（0.35）/ `H3Comfy044`（0.44） |
| **ControlNet 在 0.35 必崩** | 提交即崩，无产物（两种分辨率都崩） | `Fatal Python error: Aborted`，栈里 `ComfyUI\comfy\ldm\minimax\model.py:586` | 0.35 **别跑 CN**；改 0.44·8189（`CN4_int4_73.json` 是 0.35 崩溃复现件） |
| **CN 在 0.44 只能"扰动"不能"控制"** | 喂控图，出片跟不住控图的运动/构图 | ①二轮逐帧实测：修好类型后能跑，strength 1.0 整片发红发糊、0.2/0.5 与无 CN 肉眼无差、三档都看不到白/红条被跟随；同 seed PSNR **21.5 dB**（有变化≠有控制）②耗时 70.4 s vs 无 CN 69.6 s ③一轮人像控图 PSNR 仅 9.36 dB、+61% 耗时 | 判 **CN Union 不可用于 H3 ref2va**；要控运动用 `MiniMaxH3AddGuide`（首帧/参考图）或时间线分段 |
| **CN 逐帧喂视频报类型错** | `prompt_outputs_failed_validation` / node `MiniMaxH3FunControlNetApply` | `return_type_mismatch: control_video, received_type(VIDEO) mismatch input_type(IMAGE)` | `LoadVideo` 输出是 VIDEO，中间必须插 **`GetVideoComponents`**（out: IMAGE,AUDIO,FLOAT,COMBO,COMBO），再把 `control_video` 指到它的 IMAGE 输出 |
| **X2 细节 VAE 不能直连 VAELoader** | `RuntimeError: shape '[1, 7, 16, 16, 3, 4, 16, 16]' is invalid for input of size 22020096`（`comfy/ldm/minimax/vae.py:308`） | — | 必须用节点 **`MiniMaxH3X2DetailedUpscale`**（images / detail_vae / detail_strength），接 VAEDecodeTiled 后、CreateVideo 前 |
| **LoRA 文件截断** | `ValueError: Tensor 'diffusion_model.token_refiner.blocks.1.mlp.fc2.lora_B.weight' extends past the end of the file` | 字节数对不上声明值 | 重下（32MB 有界 Range），别拿半截文件凑 |
| **「GGUF 不支持 H3」是假象** | ComfyUI-GGUF 白名单确实没有 minimax | — | HF `Abiray/MiniMax-H3-Pruned-GGUF` 把 `general.architecture` 写成 **"wan"** 骗过白名单，能加载；73 帧仍比 int4 慢 43% |
| **w6a8 只有 0.44 认** | 0.35 加载报 `KeyError: 'w6a8_int8'` | 0.35 的 `QUANT_ALGOS` 无 w6a8 | 要试就切 0.44（4 步 73 帧冷启 68.5 s） |
| **EasyCache 出鬼影** | 人群出现半透明重影 | 20 步 + `EasyCache 0.2`：179.6 → 136.6 s（1.31×） | 复杂场景**别用**，只留给简单场景 |
| **重启前先 netstat 取 PID** | `Get-NetTCPConnection` 在该机取不到 OwningProcess | — | `netstat -ano \| findstr :8188`，最后一列是 PID |
| **PowerShell `[math]::Min` int64 坑** | 算帧数/时长时绑不上重载或结果少一截 | Int32 字面量与 Int64 混用 | `[math]::Min([int64]$a,[int64]$b)` 或写 `if`；**文档无原文，属通用行为，未在本机实测** |
| **7z / 大文件** | 解 `*.7z` 没程序；`copy /b` 450 块命令行太长；下载中途挂死、`-r N-` 被 302 跳墙 | — | `pip install py7zr` 或装 7-Zip；`copy /b` 改 60 块/批（`exec\asm.ps1`）；下载走 ModelScope **32MB 有界 Range 串行**（`exec\dl9.ps1`）；Windows 写 `curl.exe` |

其它症状（无报错型、接缝、换脸、假提速…）见 `references\troubleshooting.md`。**保命三条**：跑图时别重启别杀进程，先看 `/queue`；新图先 512×512×33 验结构；改动前先备份或记原值。

## 5 SageAttention（2026-11-09 修好，已默认开）

> **只对 0.35 有效。0.44 装不上**：把 0.35 的 `sageattention`(2.2.0, cp39-abi3 wheel) 复制到 0.44（torch **2.14.1+cu130**）后，`--use-sage-attention` 每步刷 `[ERROR] Error running sage attention: 0 active drivers ([]). There should only be one., using pytorch attention instead.` 然后回退 pytorch attention，4 步 73 帧卡在 `0/4 Model Initializing` >5 min。已把 0.44 的 `triton`/`sageattention` 改名 `_disabled_*` 停用；0.44 想快只能靠默认的 Comfy 模型编译器（704 尺寸 -42%，1344 尺寸只 -5%）。

**症状**：挂 sage 节点或加 `--use-sage-attention` 后 triton 编译失败 `Failed to compile. cc_cmd: [...triton\runtime\cc\cc.exe ... driver.c ... -lpython313] CalledProcessError`；手工复现真错 = `driver.c:19: error: include file 'Python.h' not found`。
**根因**：ComfyUI 的 embeddable python 缺开发文件——`python_embeded\libs\python313.lib` 与 `python_embeded\Include\Python.h` 都不存在（换 sage whl 无用）。
**修法**（完整 6 条命令见 `references\sage_and_quant.md` §1，可直接复制）：nuget 3.13.14 的 `tools\libs\python313.lib` 放 `python_embeded\libs\` **并再拷一份到** `python_embeded\Lib\site-packages\triton\backends\nvidia\lib\`；`tools\include\*` 头文件**必须先打 zip 再传**（`scp -r` / `Copy-Item -Recurse` 都会失败）解到 `python_embeded\Include\`。

~~~powershell
# 验证：必须先看到 RUN OK 才算修好（先把 exec\chk_sage2.py 传到 C:\Users\Administrator\h3opt\）
ssh Administrator@192.168.0.15 D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\python_embeded\python.exe C:\Users\Administrator\h3opt\chk_sage2.py
# 期望：RUN OK torch.Size([2, 1024, 8, 64]) 0.0xxxxxx   失败：RUN ERR: ... + traceback
~~~

**生效二选一**：① 启动加 `--use-sage-attention`（已写进 `start_comfy.bat`，推荐，图里免节点）；② 图里插 `PathchSageAttentionKJ`，`sage_attention="auto"`（要有 KJNodes，0.44 没有）。
**收益**：20 步 188.7 → **111.9 s（-40.7%，PSNR 23.0 dB）**；4 步 72.5 → **60.6 s（-16.4%，PSNR 37.7 dB）**；全局参数热跑 **43.2 s**（冷 83.4 s）。
**回滚**：删 `start_comfy.bat` 里的参数并重启，或删图内节点；`python313.lib` / `Include\` 留着无害。

## 6 三版本：切换与回滚

| 版本 | 开关（改哪里） | 回滚动作 | 重启 |
|---|---|---|---|
| **A1 · int4 底模（默认已开）** | `UNETLoader.unet_name` = `MiniMax_H3_Ref2VA_pruned_mixed_int4_int8_convrot.safetensors` | 改回 `unet\minimax_h3_ref2va_pruned_int8_convrot.safetensors`（旧文件没删） | 不用 |
| **A2 · 量化替代档**（w6a8 仅 0.44 / GGUF 仅 0.35） | w6a8：`UNETLoader.unet_name` = `minimax_h3_ref2va_pruned_w6a8.safetensors`；GGUF：装 `ComfyUI-GGUF` + `UnetLoaderGGUF` 选 `MiniMax-H3-Ref2VA-Pruned-Q4_K_M.gguf` | 换回 `UNETLoader` + int4/int8 safetensors（0.35 加载 w6a8 报 `KeyError: 'w6a8_int8'`） | GGUF 要，w6a8 不用 |
| **S · SageAttention** | `start_comfy.bat` 带 `--use-sage-attention`（已开） | 删参数重启；或删图内 `PathchSageAttentionKJ` | 要 |
| **B1 · 画质 LoRA** | turbo(1.0) → realism(0.6) → motion(0.5) → facial(0.4)，`BasicGuider.model` 指最末一级 | 删节点，`BasicGuider.model` 指回 turbo 输出（生产图节点 40，`["40",0]`） | 不用 |
| **B2 · 细节 VAE（2K）** | 节点 `MiniMaxH3X2DetailedUpscale` 接 VAEDecodeTiled 之后 | 删节点 + 删 `custom_nodes\ComfyUI-H3-X2-Detailed\` | 要 |
| **C1 · 0.44 并存（已装）** | `schtasks /run /tn H3Comfy044` → 8189；只给 CN / w6a8 / 新节点用 | `schtasks /end /tn H3Comfy044` + `schtasks /run /tn H3Comfy` 回 0.35 | 要 |

~~~powershell
ssh Administrator@192.168.0.15 schtasks /end /tn H3Comfy ; ssh Administrator@192.168.0.15 schtasks /run /tn H3Comfy044   # → 0.44·8189
ssh Administrator@192.168.0.15 schtasks /end /tn H3Comfy044 ; ssh Administrator@192.168.0.15 schtasks /run /tn H3Comfy   # → 0.35·8188
~~~

**0.44 没有实质提速**（4 步 40.2 s vs 0.35 开 sage 43.2 s；20 步 195.8 vs 188.7 s），价值是 **CN 能跑通 + 新节点生态**；两套**不能同时跑**（24GB 不够）。文件路径级回滚见 `references\rollback.md`。

## 7 画质 vs 速度：4 步不是万能

夜景+人群、**1344×768×73**、int4、干净环境（未开 sage 批次）：

| 配置 | 耗时 | 相对 4 步 | 观感 / 用途 |
|---|---|---|---|
| turbo 4 步 euler/simple · res_multistep/beta | **45.5 / 45.8 s** | 1.00× | 简单场景（单人/室内/光足）就它，采样器无实质差 |
| Pika Flex 4 步 euler/simple | **47.3 s** | 1.04× | 背景糊，**别用** |
| **Pika Flex 6 步 rm/beta** | **65.0 s** | 1.43× | **复杂场面首选**（≈turbo 8 步观感；6 步 euler 69.3 s 更慢不选） |
| turbo 8 步 rm/beta | **83.4 s** | 1.83× | 脸仍略「蜡」，人群可辨 |
| turbo 12 步 rm/beta | **115.1 s** | 2.53× | 细节自然，接近满血 |
| turbo 20 步 rm/beta | **179.6 s** | 3.95× | 最好，日常不必要 |
| 20 步 + EasyCache 0.2 | **136.6 s** | 3.00× | 快 1.31×，但人群**半透明鬼影** |

同一批在 **704×1248×73 + sage** 下：20 步 **188.7 → 111.9 s**，4 步 **72.5 → 60.6 s**（全局参数热跑 **43.2 s**）。线性模型 **≈8.3 s/步 + 12 s 固定开销**（1344×768/73）。结论：**简单场景 4 步 + sage；复杂场面 Pika 6 步（65 s）；要细节 12 步（115 s）**；EasyCache 只给简单场景。

## 8 长视频 >124 帧：拆段（唯一可行解）

| 项 | 做法 |
|---|---|
| 段长 | 单段上限 124 帧；实操 **73 + 51**，再长就 73+51+51… |
| 第二段接法 | seg1 末帧（`ffmpeg -y -sseof -0.08 -i seg1.mp4 -frames:v 1 seg1_last.png`）→ `LoadImage` → `MiniMaxH3AddGuide(frame_idx=0)` → `BasicGuider.conditioning` |
| 提示词 | seg2 前半句**原样重复** seg1 场景描述，后半句只写「动作升级」；**别重新描述长相**（会换脸） |
| seed | **每段换一个**（同 seed 会让第二段重复第一段动作） |
| 拼接 | `ffmpeg -f concat -safe 0 -i list.txt -c copy final.mp4`；亮度差一档先 `ffmpeg -vf eq=brightness=0.02 -c:a copy` 修再拼 |

价值是「单段跑不动时能跑 / 能接戏 / 长度不受限」，**不是提速**。干净态拆段耗时**未测**（旧 105.7 / 157.6 s 含污染作废）；按帧数推算 51 帧 ≈35 s；0.44 上 704×1248×51 帧 4 步实测 **32.6 s**。

## 9 未测清单（别当已验证用）

**已补测（2026-11-09 17:00，别再当未测）**：CN 逐帧（→ 判死，见 §4）、124 帧纯文生（103.2 s 通过）、0.44 吃不吃 sage（→ 装不上）、GGUF/w6a8 在 0.35 与 0.44 的同图对照。

**仍未测**：X2 细节 VAE 的干净态 2K 耗时 / 1344×768 的 51 帧干净态 / 512×512×33 干净态 / B1 画质 LoRA 干净态耗时 / GGUF 其余量化档（Q3_K_M、Q5_K_M、Q6_K、Q8_0）/ w6a8 画质 PSNR / 除 sage 外的启动参数组合。全表见 `references\measured_baselines.md` §7、§8。

## 10 references

- `references\measured_baselines.md` —— 引用具体耗时/显存/字节数；§5 旧数字为何作废，§8 下午轮全表
- `references\troubleshooting.md` —— 有报错搜原文；§2 是「无报错但不对劲」
- `references\rollback.md` —— 退回任何改动；§2 一键回旧稳定态，§3 回滚后 4 条验证
- `references\sage_and_quant.md` —— 修 sage 的完整命令、int8/int4/w6a8/GGUF 四档量化对照与取舍
- 原始文档（数字来源，**旧耗时已作废**）：`D:\AgentHome\minimax-h3-review-20261106\00_README.md`、`03_已执行改动与验证回滚记录.md`、`guides\07`~`13`
