# H3 排障手册（按报错文本可搜索）

> 用法：Ctrl+F 搜报错原文片段；搜不到就翻 §2（无报错文本的症状型）。
> 总原则：先判「在算还是卡死」（§2.1），再动刀；**跑图时不要重启、不要杀进程，先查 `/queue`**。
> 所有耗时口径是 **2026-11-09 深夜干净环境**（GPU 空闲 / IndexTTS 停 / 单 ComfyUI）；旧数字已作废。

## 1 有报错原文的

### 1.1 `Fatal Python error: Aborted`（ControlNet 在 0.35 上必崩）
- 位置：`ComfyUI\comfy\ldm\minimax\model.py:586`
- 现象：提交带 ControlNet 的图，**进程直接崩**（不是 return error），无产物；1344×768×73 与 704×480×33 都崩
- 原因：0.35.0 的 H3 实现不支持 `MiniMaxH3FunControlNetApply` 这条 patch 路径（0.44 才补上）
- 处置：**0.35 上不要跑 CN**。改起 0.44：
  ~~~text
  ssh Administrator@192.168.0.15 schtasks /end /tn H3Comfy
  ssh Administrator@192.168.0.15 C:\Users\Administrator\h3opt\start_044.bat
  curl.exe -s http://192.168.0.15:8189/system_stats
  ~~~
  复现件：`exec\CN4_int4_73.json`（0.35）；能跑的是 `exec\CN044small.json` / `CN044full.json`（0.44）
- 来源：2026-11-09 实测（本轮）

### 1.2 `RuntimeError: shape '[1, 7, 16, 16, 3, 4, 16, 16]' is invalid for input of size 22020096`
- 位置：`comfy/ldm/minimax/vae.py:308`
- 现象：把 X2 细节 VAE 塞进普通 `VAELoader`，解码阶段抛错
- 原因：X2 细节 VAE 不是常规 VAE，张量布局不同，必须走自定义节点
- 处置：删掉那个 VAELoader，改用节点 **`MiniMaxH3X2DetailedUpscale`**（输入 `images / detail_vae / detail_strength`），接在 **VAEDecodeTiled 之后、CreateVideo 之前**；`detail_vae` 选 `MiniMax-H3-X2-Detail-v1_int8_convrot.safetensors`（2.70GB）
- 来源：旧轮实测 + 本轮复核

### 1.3 `ValueError: Tensor 'diffusion_model.token_refiner.blocks.1.mlp.fc2.lora_B.weight' extends past the end of the file`
- 现象：加载 LoRA 时立刻炸，提示张量越过文件末尾
- 原因：**LoRA 文件下载不完整/被截断**（大文件长连接中断最常见）
- 判据：文件字节数与声明值不符（本轮踩过的截断件已重下）
- 处置：删掉重下；下载走 **32MB 有界 Range 串行**，不要用 `-r N-` 开放式 Range
  ~~~text
  ssh Administrator@192.168.0.15 python -c "import os;print(os.path.getsize(r'D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI\models\loras\minimax_h3_ref2v_turbo_4step_v0.1_comfyui_bf16.safetensors'))"
  ~~~
- 来源：本轮实测

### 1.4 `Unexpected architecture type in GGUF file`（**2026-11-09 已推翻：GGUF 能用**）
- 现象：用 ComfyUI-GGUF 的 `UnetLoaderGGUF` 加载 MiniMax-H3 的 GGUF 时这样报错
- 原因：`custom_nodes\ComfyUI-GGUF\loader.py` 白名单写死，**没有 minimax**；但 HF 上 `Abiray/MiniMax-H3-Pruned-GGUF` 把这个 GGUF 的元数据 `general.architecture` 写成了 **"wan"**，所以**官方那几个文件能直接加载**（不是白名单漏洞，是模型元数据换了名字）
- 处置：用 `Abiray/MiniMax-H3-Pruned-GGUF` 的文件，`models\unet\MiniMax-H3-Ref2VA-Pruned-Q4_K_M.gguf`（11.56GB）。**但 73 帧档没用**：704×1248×73、4 步 turbo 冷 75.9 s / 热 61.6 s，峰值 **23,719 MiB**，比 int4 的 43.2 s 慢 ≈43%，显存也没省 → 只在「要腾权重显存换更高分辨率/更长帧」时考虑。已有 Q3_K_M 8.90 / Q4_K_M 11.56 / Q5_K_M 14.07 / Q6_K 16.73 / Q8_0 21.58 GB，只测过 Q4_K_M。

### 1.5 `"node_errors"`（POST /prompt 返回 HTTP 400）
- 现象：提交直接 400，body 里有 `node_errors`
- 原因：图结构错——参数名拼错、连线类型不对、引用了不存在的节点 id
- 处置：`GET /object_info` 查该节点真实输入名与类型（**别猜**）；最常见是 `ref_images.ref_image_0` 这类带点号的键

### 1.6 `Failed to compile. cc_cmd: [...triton\runtime\cc\cc.exe ... driver.c ... -lpython313] CalledProcessError`（**2026-11-09 已修好**）
- 现象：挂 `PathchSageAttentionKJ` 或加 `--use-sage-attention` 后，一提交就报 triton 编译失败，堆栈里有 `cc.exe` 与 `driver.c`
- 手工复现出的真错：`driver.c:19: error: include file 'Python.h' not found`
- 原因：ComfyUI 的 **embeddable python 缺开发文件**——`python_embeded\libs\python313.lib` 与 `python_embeded\Include\Python.h` 都不存在。**与 sage 版本 / whl 无关**（2.2.0 一样，换 whl 无用）
- 处置（一次修好，5 分钟）：nuget `https://www.nuget.org/api/v2/package/python/3.13.14`（约 14MB）→ 解出 `tools\libs\python313.lib` 与 `tools\include\*`；lib 放 `python_embeded\libs\` **并再拷一份到** `python_embeded\Lib\site-packages\triton\backends\nvidia\lib\`；头文件**必须打成 zip 再传**（`scp -r` 与 `Copy-Item -Recurse` 都会失败）解到 `python_embeded\Include\`。完整 6 条命令见 `sage_and_quant.md` §1
- 判据：`chk_sage2.py` 打印 **`RUN OK torch.Size([2, 1024, 8, 64])`** 才算修好；失败打 `RUN ERR: ...`
- 生效：`start_comfy.bat` 加 `--use-sage-attention`（推荐，图里不用节点），或图里插 `PathchSageAttentionKJ`（`sage_attention="auto"`）

### 1.7 `TypeError`（若干行）
- 现象：跑了才炸
- 原因：挂了 **Seed-VC 1.0.1** 自定义节点（该版本就是坏的）
- 处置：删节点；未验证过的自定义节点别进生产图

### 1.8 `patchify_video` 维度错
- 现象：分辨率设 720×1280 报维度错
- 原因：不满足 H3 的 patch 对齐
- 处置：改用 **1344×768** 或 **704×1248**

### 1.9 `KeyError: 'w6a8_int8'`（0.35 加载 w6a8 底模）
- 现象：0.35.0 加载 `minimax_h3_ref2va_pruned_w6a8.safetensors`（**15,983,746,636 B**）直接抛 `KeyError: 'w6a8_int8'`
- 原因：`comfy/quant_ops.py` 的 `QUANT_ALGOS` 只有 `mxfp8 / int8_tensorwise / convrot_w4a4 / asym_w4a8_int8`，**0.35 不认识 w6a8 这个量化格式**
- 处置：0.35 上继续用 int8 convrot / int4 mixed；要用 w6a8 就切 **0.44·8189**（`schtasks /end /tn H3Comfy` + `schtasks /run /tn H3Comfy044`），0.44 上 4 步 73 帧**冷启 68.5 s** 实测能出片

### 1.10 pip 装包失败（代理拒连 / wheel 被拒）
- 现象 A：`ProxyError` 指向 127.0.0.1:7890；现象 B：`Invalid wheel filename`
- 原因：机器上有 FlClash 监听 127.0.0.1:7890 可当代理；离线 wheel 的 build tag **必须以数字开头**
- 处置：`python_embeded\python.exe -m pip install --no-index --no-deps --force-reinstall <whl>`；`-1cu130torch210-` 合法，`-cu130torch2.10.0andhigher-` 会被拒 → 改文件名

### 1.11 `The command line is too long` / 拼接大文件失败
- 现象：`copy /b a+b+c+…` 拼 450 个分块时命令行超长
- 处置：分批拼 **60 块/批**（`exec\asm.ps1`）；下载用 `exec\dl9.ps1`（ModelScope + 32MB 有界 Range 串行）

### 1.12 下载中途挂死 / 302 跳到被墙域名
- 现象：hf-mirror 单连接跑到 50~60MB 就断；`-r N-` 开放式 Range 被 302 后挂住
- 处置：改**有界 Range**（每次 32MB 串行），走 ModelScope 同源文件，实测 **28~31 MB/s**

### 1.13 `Cannot find an overload for "Min"` / 帧数算出来被截断
- 现象：PowerShell 里 `[math]::Min` / `[math]::Max` 混算帧数、时长时绑定失败或结果少一截
- 原因：Int32 字面量与 Int64/JSON 数值混用
- 处置：`[math]::Min([int64]$a,[int64]$b)`，或写 `if`；帧数/时长一律整数算
- **注意**：4 份实测文档里**没有**这条原文，属通用 PowerShell 行为，**未在本机实测**

### 1.14 解压 .7z 没有工具
- 现象：Windows 无内置 7z
- 处置：装 7-Zip；或 `python_embeded\python.exe -m pip install py7zr`

### 1.15 curl 调不通 / 拿不到 JSON
- 判据：`curl.exe -s -m 5 http://192.168.0.15:8188/system_stats` 回 JSON 才算链路通；无输出就去掉 `-s` 或加 `-v`
- 注意：Windows 里裸 `curl` 是 `Invoke-WebRequest` 的别名（行为完全不同），**一律写 `curl.exe`**

### 1.16 `Node 'PathchSageAttentionKJ' not found` / 0.44 找不到 GGUF 加载器
- 现象：把 0.35 的图搬到 0.44·8189，要么报节点不存在，要么 `UnetLoaderGGUF` 下拉里没有
- 原因：0.44 是**独立安装**，**没有装 KJNodes**（所以没有 `PathchSageAttentionKJ`），**也没有 ComfyUI-GGUF**
- 处置：0.44 上开 sage 只能靠启动参数 `--use-sage-attention`；GGUF 只能在 0.35 上用；要跨版本用图，先把这两个节点摘掉再导

## 2 没有报错文本的症状型

### 2.1 最优先：利用率 100% 但功耗低 = 卡死（不是慢）
~~~text
ssh Administrator@192.168.0.15 nvidia-smi --query-gpu=memory.used,utilization.gpu,power.draw --format=csv
~~~
| 读数 | 判断 |
|---|---|
| 利用率 100% + 功耗 **> 200W** | 正常在算（124 帧峰值可到 425W） |
| 利用率 100% + 功耗 **< 200W**（常见 110W / 100.16W） | **换页抖动**，立刻处置，别等 |
| 显存 > 23.5 GB | 已贴顶，随时会抖 |

~~~text
ssh Administrator@192.168.0.15 "netstat -ano | findstr :8188"
ssh Administrator@192.168.0.15 taskkill /PID <上一步的PID> /T /F
ssh Administrator@192.168.0.15 C:\Users\Administrator\h3opt\start_comfy.bat
~~~
> `POST /interrupt` 对这种卡死**无效**，只能 taskkill + 重启。

### 2.2 同图同 seed 比预期慢 50%（后台进程抢显存）
- 判据：干净 **45.5 s** vs 有后台进程 **68~70 s**；空闲显存 **11,745 MiB**（应 **1,430 MiB**）
- 原因：IndexTTS 两套（10.3GB）和/或第二个 ComfyUI（0.44·8189）在抢显存
- 处置：
~~~text
ssh Administrator@192.168.0.15 C:\Users\Administrator\h3opt\tts_off.bat
ssh Administrator@192.168.0.15 schtasks /change /tn IndexTTS_WebUI /disable
ssh Administrator@192.168.0.15 schtasks /change /tn IndexTTS_API /disable
ssh Administrator@192.168.0.15 "netstat -ano | findstr :8189"
ssh Administrator@192.168.0.15 nvidia-smi --query-gpu=memory.used --format=csv
~~~

### 2.3 IndexTTS 停过又自己回来了
- 判据：`schtasks /query /tn IndexTTS_WebUI /fo LIST` 状态 Ready 且 7860/7861 有进程
- 原因：两个 **BootTrigger** 计划任务 `IndexTTS_WebUI` / `IndexTTS_API` 在重启后自动拉起
- 处置：`schtasks /change /tn IndexTTS_WebUI /disable` 和 `…/tn IndexTTS_API /disable`（**Disable 才根治，`/end` 只停当前实例**）；要恢复语音用 `tts_on.bat`（冷启约 55 s）

### 2.4 124 帧（或 >73 帧）跑不完，>30 分钟无产物
- 判据：按 §2.2 先看显存；干净态 124 帧生产链只要 **157.9 s**
- 处置：若不是后台进程，就是显存抖 → 拆段（73+51）或降帧

### 2.5 进度条卡 0%，日志无输出
- 判据：先看功耗。**<200W + 100%** → 显存抖动（§2.1）；**>200W** → 冷启动编译，带 LoRA 首次可卡 5~13 分钟
- 处置：抖就降帧/拆段；不抖就等；超 15 分钟按 §2.1 杀进程重跑

### 2.6 8188 连不上
~~~text
ssh Administrator@192.168.0.15 schtasks /run /tn H3Comfy
curl.exe -s http://192.168.0.15:8188/system_stats
~~~
- 等 **12~40 s** 应返回 200；若任务退出码 1（§2.7），改用 `start_comfy.bat` 后台起

### 2.7 `schtasks /run` 返回成功但任务退出码 1
- 现象：ssh 非交互会话里跑，命令成功，任务历史 **0x1**
- 原因：任务登录态是「只使用交互式」
- 处置：用 `C:\Users\Administrator\h3opt\start_comfy.bat` 后台起，或去本机控制台点

### 2.8 `Get-NetTCPConnection` 取不到 OwningProcess
- 处置：用 `netstat -ano | findstr :8188`，最后一列就是 PID

### 2.9 出片有接缝 / 第二段亮度差一档
- 处置：两段共用同一场景描述；seg2 首帧用 seg1 末帧过 `MiniMaxH3AddGuide(frame_idx=0)`；亮度差用 `ffmpeg -i seg2.mp4 -vf eq=brightness=0.02 -c:a copy seg2_fix.mp4` 修再拼

### 2.10 第二段换了脸
- 原因：第二段提示词重新描述了长相
- 处置：第二段只写「同一场景 + 动作升级」，身份靠 AddGuide 图片锚定

### 2.11 耗时看起来异常短（假提速）
- 原因：同参数同 seed 复跑命中 ComfyUI 节点缓存
- 判据：与首次同 seed 对比；偏差幅度**未测**
- 处置：每次实测换 `RandomNoise.noise_seed`

### 2.12 画面糊 / 人群融化（不是故障）
- 判据：4 步蒸馏在**复杂场景**才暴露：人群糊成色块、人脸融化、招牌不可读
- 处置：简单场景保持 4 步（**45.5 s**）；复杂场面换 **Pika Flex 6 步（65.0 s）**，或 turbo 8/12 步（83.4 / 115.1 s）

### 2.13 人群出现半透明鬼影
- 原因：开了 **EasyCache 0.2**（20 步 179.6 → 136.6 s，1.31×）
- 处置：复杂人群场景关掉 EasyCache；只给简单场景用

### 2.14 ControlNet 上了但看不出效果
- 现象：0.44 上 CN 能加载、能出图，但输出完全跟随提示词；喂人像控图，出片人物/场景与控图不符
- 判据（2026-11-09 下午实测）：704×1248×73 有 CN **64.0 s** vs 同 seed 无 CN **39.7 s**（**+61%**）；两者 **PSNR y 仅 9.36 dB**（内容完全不同）→ 不是「没生效」，是**控制映射不可信**
- 原因：用**单张静帧**当 `control_video`，没有逐帧控制序列；且 CN 与 H3 的对应关系未验证
- 处置：状态「能跑通、**不作生产依赖**」；要用必须先自己验证控图映射；要真控制需准备逐帧序列，**未测**。0.35 上同图**硬崩**（`Fatal Python error: Aborted` @ `model.py:586`），只能用 0.44

### 2.15 B1 三 LoRA 之后构图变了（人变半身特写）
- 原因：`h3-realism-people` 会把人自动拉近
- 处置：全景/群戏降强度或摘掉 realism LoRA；要保构图只挂 turbo LoRA

## 3 ssh / PowerShell / 批处理 环境坑

| 现象 | 原因 | 处置 |
|---|---|---|
| 远程复杂 PowerShell 被引号截断 | ssh 传参层丢引号 | 写 `.bat` 用 `scp` 传过去再 `cmd /c` 执行 |
| 命令被安全层拦 | 拼接字符串 / `-EncodedCommand` | 命令**字面量整条**写死，禁止拼接 |
| 本地 `pwsh` 不在 PATH | 版本目录未入 PATH | 写 `powershell -NoProfile -File xxx.ps1` |
| 目标机没有 `git` | 环境是目录拷贝 | 一切回滚靠文件备份，别写 `git checkout` |

## 4 提交前自检（30 秒，能省半小时）

1. `curl.exe -s http://192.168.0.15:8188/queue` → running / pending 都空
2. `nvidia-smi` 空闲显存 ≈**1,430 MiB**（TTS 已停、0.44 没开）
3. body 是完整 HTTP body（顶层有 `prompt` + `client_id`）→ 直接 `--data-binary @file`
4. `RandomNoise.noise_seed` 已换新值
5. `SaveVideo.filename_prefix` 用 `h3opt_ab/` 前缀 + 可辨名字
6. 提交前 `POST /free` 一次（`{"unload_models":true,"free_memory":true}`），关掉目标机上的浏览器
7. 首跑 10 秒后看功耗：>200W 才算真在算（§2.1）