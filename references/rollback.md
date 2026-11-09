# H3 回滚手册（文件路径级）

> 判据：**回滚 = 把工作流改回去 + 把文件删掉/移走**，两步都要做，只改工作流不算回滚。
> 目标机 192.168.0.15，ssh 免密可用；以下命令可直接复制粘贴。文件名含特殊字符时整条用双引号包住。
> 目标机**没有 git**：所有回滚靠「改回参数 + 删文件」，别写 `git checkout`。
> 删任何东西前先 `dir` 确认路径存在且是你要删的那个（见各节的 verify 行）。

## 0 回滚总表

| # | 改动 | 关键文件（0.35 根 = `D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI`） | 回滚动作 | 验证 |
|---|---|---|---|---|
| 1 | int4 底模（A1） | `models\unet\MiniMax_H3_Ref2VA_pruned_mixed_int4_int8_convrot.safetensors` 15,093,774,276 B | 工作流 `UNETLoader.unet_name` 改回 `minimax_h3_ref2va_pruned_int8_convrot.safetensors` | 跑通 1 条 73 帧、Staged ≈19,995 MB |
| 2 | turbo 4 步 LoRA | `models\loras\minimax_h3_ref2v_turbo_4step_v0.1_comfyui_bf16.safetensors` | 删 `LoraLoaderModelOnly`，`BasicGuider`/`BasicScheduler` 的 model 指回 UNETLoader | 图里无 LoRA 节点 |
| 3 | Pika Flex LoRA | `models\loras\pika_h3_flex_ref2va_v1_bf16.safetensors` 1,956,172,304 B | strength 设 0 或摘节点 | 图里无该 LoRA |
| 4 | 三画质 LoRA（B1） | `models\loras\h3-realism-people-t2v-i2v-r2v.safetensors` 131,229,656 / `Motion_Repair_V2.safetensors` 155,110,272 / `minimax-h3-facial-realism-closeup-cp2000.safetensors` 74,570,624 | 三个 LoRA 节点全删（或 strength 0）+ **删触发词 `r34l1sm`** | 构图不再被拉成半身特写 |
| 5 | X2 细节 VAE（B2） | `models\vae\MiniMax-H3-X2-Detail-v1_int8_convrot.safetensors` 2,834,835,196 B + `custom_nodes\ComfyUI-H3-X2-Detailed\` | 删 `MiniMaxH3X2DetailedUpscale` 节点 + 删节点目录 + 重启 | `/object_info` 无该节点名 |
| 6 | ControlNet Union | `models\model_patches\minimax_h3_fun_controlnet_union_pruned_int8_convrot.safetensors` 2,296,635,360 B | 删 `ModelPatchLoader` + `MiniMaxH3FunControlNetApply` 两个节点 | 图里无 ModelPatchLoader |
| 7 | ComfyUI 0.44（C1） | `D:\moxing\ComfyUI_044\ComfyUI_windows_portable`（4.2 GB）+ `C:\Users\Administrator\h3opt\start_044.bat` + `comfy044.log` | 杀 8189 的 python；彻底清理则删目录+bat+log+撤 8189 防火墙规则 | `netstat -ano | findstr :8189` 无输出 |
| 8 | EasyCache | 工作流内节点（`EC20c_int4_73.json` 那条） | 删节点，步数保持 20 | 耗时回到 179.6 s 档 |
| 9 | 生产图 4 步化 | `exec\PRODc.json` / `exec\PROD_int4_4step.json` | 直接用原始 `raw\prod_prompt_api.json` 提交（20 步 rm/beta、无 LoRA） | 时间戳/步数与原始一致 |
| 10 | IndexTTS 停用 | 计划任务 `IndexTTS_WebUI` / `IndexTTS_API` | `tts_on.bat` 或 `schtasks /change /enable` | 7860/7861 有监听 |
| 11 | SageAttention 启动参数（S） | `C:\Users\Administrator\h3opt\start_comfy.bat` 的 `--use-sage-attention` | 删参数 + `schtasks /run /tn H3Comfy` | 73 帧 4 步从 43.2 s 回到 72.5 s 档 |
| 12 | sage 开发文件（让 sage 能编译） | `python_embeded\libs\python313.lib`、`python_embeded\Include\Python.h` 等、`python_embeded\Lib\site-packages\triton\backends\nvidia\lib\python313.lib` | 删这三处（只影响 sage，正常出图不受影响），详见 §1.11 | 删后挂 sage 会回到 `CalledProcessError` |
| 13 | GGUF 底模（A3） | `models\unet\MiniMax-H3-Ref2VA-Pruned-Q4_K_M.gguf` 11.56GB + `custom_nodes\ComfyUI-GGUF\` | 工作流换回 `UNETLoader`；彻底清理再删 gguf 与节点目录 + 重启 | 73 帧 4 步回到 43.2 s 档（GGUF 是 61.6 s） |
| 14 | w6a8 底模（A2） | `models\unet\minimax_h3_ref2va_pruned_w6a8.safetensors` 15,983,746,636 B | 0.44 工作流换回 int4/int8；文件留着也无害（0.35 读不了） | 0.35 上加载必须先无 `KeyError: 'w6a8_int8'` |
| 15 | 0.44 计划任务（C1） | 任务 `H3Comfy044` → `C:\Users\Administrator\h3opt\start_044.bat`（8189） | `schtasks /end /tn H3Comfy044`，彻底清理加 `schtasks /delete /tn H3Comfy044 /f` | `schtasks /query /tn H3Comfy044` 报「找不到」 |

## 1 逐项精确回滚

### 1.1 底模回 int8（最快、最常做）
~~~text
ssh Administrator@192.168.0.15 cmd /c "dir D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI\models\unet"
~~~
把工作流节点 1 的 `unet_name` 改成 `minimax_h3_ref2va_pruned_int8_convrot.safetensors` 即可（**旧文件一直没动**）。
- 影响：权重 Staged 14,390 → 19,995 MB；峰值显存基本不变（峰值由 VAE 解码决定）；73 帧 4 步耗时同档（提速看步数与 LoRA，不看底模）。
- 想彻底删掉 int4 文件（省 14GB / D 盘剩 209GB）：
~~~text
ssh Administrator@192.168.0.15 cmd /c "del /q D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI\models\unet\MiniMax_H3_Ref2VA_pruned_mixed_int4_int8_convrot.safetensors"
~~~

### 1.2 去加速（回 20 步满血链路）
1. 删 `LoraLoaderModelOnly`（turbo LoRA，strength 1.0）
2. `BasicGuider.model` 与 `BasicScheduler.model` 从 LoRA 节点改回 `UNETLoader` 节点 id
3. `KSamplerSelect.sampler_name = res_multistep`，`BasicScheduler.steps = 20, scheduler = beta`，`denoise = 1.0`
- 对照：干净态 20 步 73 帧 = **179.6 s**（`exec\Q20c_int4_73.json`）

### 1.3 Pika Flex 撤回
`LoraLoaderModelOnly` 里把 `pika_h3_flex_ref2va_v1_bf16` 的 strength 设 0，或整节点删掉；采样器回 `euler/simple` + 4 步 + turbo LoRA。
- 对照：Pika 6 步 rm/beta **65.0 s** › turbo 4 步 **45.5 s**

### 1.4 画质 LoRA 撤回（B1）
删三个 `LoraLoaderModelOnly`，**并把提示词里的 `r34l1sm` 触发词删掉**（留着会改变调色）。
- 残留症状：只删节点不删触发词 → 提示词里多一个无意义 token，观感偏暗。
- 对照：B1 干净态耗时**未测**（旧值 52.7 / 53.4 s 为污染环境，作废）

### 1.5 X2 细节 VAE 撤回（B2）
~~~text
ssh Administrator@192.168.0.15 cmd /c "dir D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI\custom_nodes\ComfyUI-H3-X2-Detailed"
ssh Administrator@192.168.0.15 cmd /c "rmdir /s /q D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI\custom_nodes\ComfyUI-H3-X2-Detailed"
~~~
- 工作流侧：删 `MiniMaxH3X2DetailedUpscale` 节点，`VAEDecodeTiled` 直连 `CreateVideo`。
- **必须重启 ComfyUI**，节点才消失（`/object_info` 里搜不到 `MiniMaxH3X2DetailedUpscale` 为验证）。
- VAE 文件（2.70GB）可留可删；删：
~~~text
ssh Administrator@192.168.0.15 cmd /c "del /q D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI\models\vae\MiniMax-H3-X2-Detail-v1_int8_convrot.safetensors"
~~~

### 1.6 ControlNet 撤回
删 `ModelPatchLoader`（`model_patches\minimax_h3_fun_controlnet_union_pruned_int8_convrot.safetensors`）与 `MiniMaxH3FunControlNetApply`，model 线改回直连。
- **不要为了省事在 0.35 上留着这两个节点**：0.35 提交即 `Fatal Python error: Aborted`（`comfy/ldm/minimax/model.py:586`）
- 删模型文件（2.19GB，注意在 `model_patches` 不是 `controlnet`）：
~~~text
ssh Administrator@192.168.0.15 cmd /c "del /q D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI\models\model_patches\minimax_h3_fun_controlnet_union_pruned_int8_convrot.safetensors"
~~~

### 1.7 ComfyUI 0.44 关停 / 彻底清理（C1）
关停（保留文件，随时能再起）：
~~~text
ssh Administrator@192.168.0.15 "netstat -ano | findstr :8189"
ssh Administrator@192.168.0.15 taskkill /PID <上一步最后一列> /T /F
ssh Administrator@192.168.0.15 "netstat -ano | findstr :8189"
~~~
> 0.44 与 0.35 **目录完全独立**，杀 8189 不影响 8188；但两个同时开会把同图拖到 68~70 s（+50%）。**平时必须停。**

彻底清理（省 4.2GB + 撤共享配置）：
~~~text
ssh Administrator@192.168.0.15 cmd /c "dir D:\moxing\ComfyUI_044\ComfyUI_windows_portable"
ssh Administrator@192.168.0.15 cmd /c "rmdir /s /q D:\moxing\ComfyUI_044"
ssh Administrator@192.168.0.15 cmd /c "del /q C:\Users\Administrator\h3opt\start_044.bat C:\Users\Administrator\h3opt\comfy044.log"
ssh Administrator@192.168.0.15 netsh advfirewall firewall delete rule name="ComfyUI 8189"
~~~
- 0.44 的共享配置就在它自己的包里：`D:\moxing\ComfyUI_044\ComfyUI_windows_portable\ComfyUI\extra_model_paths.yaml`（754 字节，UTF8 无 BOM）。删了目录即一并删掉，**不要动 0.35 的** `...\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI\extra_model_paths.yaml`。
- 重装 0.44 时的坑复述：`text_encoders: text_encoders` 必须显式写，只写 `clip: clip` 会看不到 qwen3vl → CLIPLoader 列表为空。

### 1.8 EasyCache 撤回
删 `EasyCache` 节点（`exec\EC20c_int4_73.json` 里那个），步数保持 20。
- 对照：136.6 s（开）vs 179.6 s（关）；开着的代价是人群**半透明鬼影**

### 1.9 生产图回到原始 20 步
~~~text
ssh Administrator@192.168.0.15 cmd /c "dir D:\AgentHome\minimax-h3-review-20261106\raw\prod_prompt_api.json"
~~~
- 直接拿这份原样提交（20 步 res_multistep/beta、2 参考图 + 1 音频、124 帧、无 LoRA），**不要改任何节点** → 就是最原始的生产配置。
- 若已 4 步化（`exec\PRODc.json` / `PROD_int4_4step.json`），把 §8.5 那四处改回来：`unet_name`、删节点 40 LoRA、`KSamplerSelect/BasicScheduler` 回 res_multistep/beta/20、`filename_prefix` 与 seed。
- 对照：清洁态 4 步化 **157.9 s**；原始 20 步 599.8 s 为**旧污染环境**，不能当干净基线。

### 1.10 恢复 IndexTTS（要用语音时）
~~~text
ssh Administrator@192.168.0.15 C:\Users\Administrator\h3opt\tts_on.bat
ssh Administrator@192.168.0.15 schtasks /change /tn IndexTTS_WebUI /enable
ssh Administrator@192.168.0.15 schtasks /change /tn IndexTTS_API /enable
~~~
- 冷启约 55 s，常驻 **11,745 MiB**（10.3GB）→ 跑图前必须按 §2.3 再停掉，否则同图 45.5 → 68~70 s。

### 1.11 撤掉 SageAttention（保留能编译的能力）
~~~powershell
# A. 只关 sage（推荐：保留 python313.lib，随时能再开）
ssh Administrator@192.168.0.15 powershell -NoProfile -Command "Copy-Item C:\Users\Administrator\h3opt\start_comfy.bat C:\Users\Administrator\h3opt\start_comfy.bat.bak_sage -Force"
#    然后用记事本/ps1 把 --use-sage-attention 从 start_comfy.bat 删掉，再重启
ssh Administrator@192.168.0.15 schtasks /run /tn H3Comfy
# B. 连开发文件一起清（会让 sage 回到「编译失败」，其它图不受影响）
ssh Administrator@192.168.0.15 cmd /c "del \"D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\python_embeded\libs\python313.lib\""
ssh Administrator@192.168.0.15 cmd /c "del \"D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\python_embeded\Lib\site-packages\triton\backends\nvidia\lib\python313.lib\""
ssh Administrator@192.168.0.15 cmd /c "rmdir /s /q \"D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\python_embeded\Include\""
~~~
- 恢复：重新按 `sage_and_quant.md` §1 那 6 条命令做一遍即可（nuget 包还在 `C:\Users\Administrator\h3opt\py313\` 就不用重下）。
- 验证：重启后 `curl.exe -s http://192.168.0.15:8188/system_stats` 200；跑 73 帧 4 步，**43.2 s → 72.5 s** 说明 sage 确实关了。

### 1.12 撤掉 GGUF / w6a8 底模
~~~powershell
# 回 int4（0.35）：把工作流节点换回 UNETLoader + 原 safetensors 文件名即可，无需重启（同类节点没变）
# 彻底清理 GGUF（0.35）：
ssh Administrator@192.168.0.15 cmd /c "del \"D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI\models\unet\MiniMax-H3-Ref2VA-Pruned-Q4_K_M.gguf\""
ssh Administrator@192.168.0.15 cmd /c "rmdir /s /q \"D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\ComfyUI\custom_nodes\ComfyUI-GGUF\""
ssh Administrator@192.168.0.15 schtasks /run /tn H3Comfy
# w6a8 只需在 0.44 的工作流里换回 int4/int8；文件删不删都不影响 0.35（0.35 会报 KeyError: 'w6a8_int8'）
~~~
- 验证：0.35 重启后 4 步 73 帧回到 **43.2 s** 档（GGUF 是 61.6 s / 峰值 23,719 MiB）；峰值显存应回落。

### 1.13 启停 0.44（不动文件）
~~~powershell
ssh Administrator@192.168.0.15 schtasks /end /tn H3Comfy044     # 停 0.44
ssh Administrator@192.168.0.15 schtasks /run /tn H3Comfy        # 起 0.35（两套不能同时跑，24GB 不够）
ssh Administrator@192.168.0.15 schtasks /query /tn H3Comfy044 /fo LIST   # 确认 0.44 状态
~~~
- 判据：`netstat -ano | findstr :8189` 无输出 = 0.44 已停；`curl.exe -s http://192.168.0.15:8188/system_stats` 200 = 0.35 已起。
- `schtasks /run` 走 ssh **返回退出码 1 但任务已经起来了**（本机实测），别据退出码判断是否成功，用上面两条探活。

## 2 一键回到「旧稳定态」（保守顺序）

1. 停 0.44：`netstat -ano | findstr :8189` → `taskkill /PID x /T /F`
2. 停 IndexTTS：`C:\Users\Administrator\h3opt\tts_off.bat` + 两个计划任务 `/disable`
3. 重启 0.35 清显存：取 PID → `taskkill /PID x /T /F` → `schtasks /run /tn H3Comfy`（**别用 `start_comfy.bat` 直接起**，经 ssh 起的进程会随 ssh 断开被带走）
4. 工作流改回：int8 底模 + 无 LoRA + 20 步 res_multistep/beta（§1.1 + §1.2）
5. 提交前跑 §4 的 4 条验证

## 3 回滚后必须验证（4 条，30 秒）

1. `curl.exe -s http://192.168.0.15:8188/system_stats` → 200 且 `comfyui_version = 0.35.0`
2. `ssh Administrator@192.168.0.15 nvidia-smi --query-gpu=memory.used --format=csv` → 空闲 ≈**1,430 MiB**
3. `curl.exe -s http://192.168.0.15:8188/object_info` 搜不到已删节点名（如 `MiniMaxH3X2DetailedUpscale`）
4. 跑 1 条 73 帧冒烟（`exec\B035_2.json` 改 seed + prefix）→ **45.5±1 s** 才算回滚到位；跑到 68~70 s 说明还有后台进程

## 4 不要做的事

- **不要删 0.35 的 `extra_model_paths.yaml`**：0.35 自己也要用（若被改坏，CLIPLoader 列表会空）
- 不要因为 `schtasks /run /tn H3Comfy` 退出码 1 就以为没起来：ssh 非交互下**它照样能把任务拉起来**，退出码 0x1 是假阴性，用 `/system_stats` 确认
- 不要用 `Start-Process` 经 ssh 起 ComfyUI 当「重启」（ssh 一断进程就没了），一律走计划任务
- 不要用 `Get-NetTCPConnection` 取 PID（该机取不到 OwningProcess），一律 `netstat -ano | findstr :8188`
- 不要在跑图时杀进程：先 `GET /queue` 确认没在跑
- 不要删 `models\unet` 下的 int8 底模（那是回滚底盘，体积大但删了就没退路）