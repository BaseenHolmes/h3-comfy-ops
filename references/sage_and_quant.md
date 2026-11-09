# SageAttention 修复 + 量化底模对照（细节页）

> 数字全部来自 2026-11-09 本机实测；测不出的一律写「未测」。条件见 `measured_baselines.md` §8（0.35 那条 = 704×1248×73 · int4 · turbo 4 步）与 §1（0.35 那条 = 1344×768×73）。

## 1 修 sage：从「装了但不生效」到「RUN OK」（6 条命令，照抄即可）

### 1.1 现象链（按这个顺序排查，别乱试）

1. 图里挂 `PathchSageAttentionKJ` 或启动加 `--use-sage-attention` → 提交后 triton 编译失败：
   `Failed to compile. cc_cmd: [...	ritonuntimecccc.exe ... driver.c ... -lpython313] CalledProcessError`
2. 光看这个包着的错没用，**手工跑一遍 cc 命令**才看到真错：
   `driver.c:19: error: include file 'Python.h' not found`
3. 结论：不是 sage 版本问题（2.2.0 一样），是 **ComfyUI 的 embeddable python 没有开发文件**。

### 1.2 缺什么

| 缺的东西 | 正确位置（两处都要） |
|---|---|
| `python313.lib` | ① `...python_embededlibspython313.lib` ② `...python_embededLibsite-packages	ritonackends
vidialibpython313.lib` |
| 头文件 `Python.h` 等 | `...python_embededInclude`（**必须打 zip 再传**，`scp -r` 与 `Copy-Item -Recurse` 都传不进去） |

路径前缀：`D:moxingComfyUI_windows_portable_nvidiaComfyUI_windows_portablepython_embeded`（下文写成 `$EMB`，实际命令里请写全路径，别用变量——ssh 传参会丢）。

### 1.3 完整命令

~~~powershell
# 1) 下 nuget 的 CPython 3.13.14 官方开发包（约 14MB）
curl.exe -L -o C:\Users\Administrator\h3opt\python313.nupkg https://www.nuget.org/api/v2/package/python/3.13.14
# 2) nupkg 就是 zip，改扩展名后展开
ssh Administrator@192.168.0.15 powershell -NoProfile -Command "Copy-Item C:\Users\Administrator\h3opt\python313.nupkg C:\Users\Administrator\h3opt\python313.zip -Force; Expand-Archive C:\Users\Administrator\h3opt\python313.zip -DestinationPath C:\Users\Administrator\h3opt\py313 -Force"
# 3) python313.lib → 两处
ssh Administrator@192.168.0.15 cmd /c "copy /y C:\Users\Administrator\h3opt\py313\tools\libs\python313.lib D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\python_embeded\libs\python313.lib"
ssh Administrator@192.168.0.15 cmd /c "copy /y C:\Users\Administrator\h3opt\py313\tools\libs\python313.lib D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\python_embeded\Lib\site-packages\triton\backends\nvidia\lib\python313.lib"
# 4) 头文件打 zip → 传 → 解到 Include\
ssh Administrator@192.168.0.15 powershell -NoProfile -Command "Compress-Archive -Path 'C:\Users\Administrator\h3opt\py313\tools\include\*' -DestinationPath C:\Users\Administrator\h3opt\py313include.zip -Force"
ssh Administrator@192.168.0.15 powershell -NoProfile -Command "Expand-Archive C:\Users\Administrator\h3opt\py313include.zip -DestinationPath D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\python_embeded\Include -Force"
# 5) 验证（先把 exec\chk_sage2.py 传到 C:\Users\Administrator\h3opt\chk_sage2.py）
ssh Administrator@192.168.0.15 D:\moxing\ComfyUI_windows_portable_nvidia\ComfyUI_windows_portable\python_embeded\python.exe C:\Users\Administrator\h3opt\chk_sage2.py
# 6) 开 sage 并重启
ssh Administrator@192.168.0.15 schtasks /run /tn H3Comfy
~~~

### 1.4 判据

| 场景 | 输出 | 含义 |
|---|---|---|
| 修好 | `RUN OK torch.Size([2, 1024, 8, 64]) 0.0xxxxxx` | sage 内核能跑，**到此才算修完** |
| 没修好 | `RUN ERR: ...` + traceback | 还缺文件，回头看 1.2 两处路径 |
| 启动后 | `comfy.log` 无 triton 编译段 | 参数生效 |

### 1.5 生效方式二选一

1. **启动参数（推荐）**：`C:UsersAdministratorh3optstart_comfy.bat` 里加 `--use-sage-attention`（已加好）→ 图里**不要**再挂 sage 节点，否则等于开两次
2. **图内节点**：`PathchSageAttentionKJ`，`sage_attention="auto"`（需要 KJNodes；**0.44 没装 KJNodes，只能用启动参数**）

## 2 四档量化底模对照

| 项 | int8 convrot | **int4 mixed（默认）** | w6a8 | GGUF Q4_K_M |
|---|---|---|---|---|
| 文件 | `minimax_h3_ref2va_pruned_int8_convrot.safetensors` | `MiniMax_H3_Ref2VA_pruned_mixed_int4_int8_convrot.safetensors` | `minimax_h3_ref2va_pruned_w6a8.safetensors` | `MiniMax-H3-Ref2VA-Pruned-Q4_K_M.gguf` |
| 体积 | 19.53GB | **15,093,774,276 B = 14.06GB** | **15,983,746,636 B** | **11.56GB**（同仓 Q3_K_M 8.90 / Q5_K_M 14.07 / Q6_K 16.73 / Q8_0 21.58） |
| 权重 Staged | **19,995 MB** | **14,390 MB** | 未记录 | 未记录 |
| 0.35.0 | 可用（回滚底盘） | **可用（生产默认）** | **`KeyError: 'w6a8_int8'`，不能用** | 可用（需 ComfyUI-GGUF） |
| 0.44.0 | 可用 | 可用 | **可用**（4 步 73 帧冷 68.5 s） | 未装 loader，未测 |
| 73 帧 4 步实测 | 与 int4 同档（提速不看底模） | 无 sage 72.5 s / 全局 sage 热 **43.2 s** | 0.44 冷 **68.5 s** | 冷 75.9 s / 热 **61.6 s**，峰值 **23,719 MiB** |
| 结论 | 只作回滚用 | **默认就它** | 只在 0.44 上玩，无提速证据 | **73 帧不推荐**（慢 ≈43%，峰值不降）；只在要腾权重显存换更高分辨率/更长帧时考虑 |

GGUF 为什么「白名单不支持也能加载」：`custom_nodesComfyUI-GGUFloader.py` 里确实没有 minimax，但 HF 的 `Abiray/MiniMax-H3-Pruned-GGUF` 把这个文件元数据 `general.architecture` 写成 **"wan"**，于是走了 wan 分支、正常加载。

## 3 sage 实测收益（704×1248×73 · 0.35.0）

| 档位 | 无 sage | 有 sage | 变化 | 画质 |
|---|---|---|---|---|
| 20 步 res_multistep/beta | 188.7 s | **111.9 s** | **-40.7%** | PSNR 23.0 dB（20 步累积差异，属正常），目视无差别 |
| 4 步 turbo LoRA | 72.5 s | **60.6 s** | **-16.4%** | PSNR 37.7 dB |
| 4 步 + 全局 `--use-sage-attention`（图内无节点） | — | **热 43.2 s**（冷 83.4 s） | 最快 | 同上 |

- 步数越高省得越多（20 步 -40.7%、4 步 -16.4%）：sage 优化的是注意力，步数多摊得多。
- **0.44 上是否也吃 sage：未测**（0.44 的 40.2 s 是它默认状态，没做开/关对照）。

## 4 与其它表的关系

| 想要 | 去哪 |
|---|---|
| 全部耗时/显存原始数字 | `measured_baselines.md` §1（1344×768 干净）、§3（0.44 + CN）、§4（显存驻留）、§8（2026-11-09 下午轮全表） |
| 报错原文 → 处置 | `troubleshooting.md` §1.4（GGUF）、§1.6（triton）、§1.9（w6a8）、§1.16（0.44 缺节点） |
| 改回去 | `rollback.md` §1.11（撤 sage）、§1.12（撤 GGUF/w6a8）、§1.13（启停 0.44） |
