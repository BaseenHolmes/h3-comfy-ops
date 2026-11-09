# h3-comfy-ops · Agent Skill

在局域网 **RTX 4090（24GB）** 上跑 **ComfyUI + MiniMax H3** 视频生成的运维技能：选方案、调参数、排障、回滚，含 **SageAttention 修复**、量化底模（int8 / int4 / w6a8 / GGUF）对照与**本机实测矩阵**。

> 数据来自一台真实机器（RTX 4090D 24GB / Windows / ComfyUI 0.35.0 / torch 2.13.0+cu130 / Python 3.13.14 embeddable）的实测，不是论文与二手教程。

## 这个 skill 能回答什么

- 4 步 / 6 步 / 12 步 / 20 步怎么选，各自多少秒、峰值多少 GB
- 1344×768 跑 124 帧会不会 OOM（答案：不会，103.2s / 23.3GB）
- SageAttention 装不上怎么办（embeddable Python 缺 `python313.lib` + `Include\Python.h`，补上即可，20 步 -40.7%）
- GGUF / w6a8 / ComfyUI 0.44 到底值不值得迁（答案：都不值，附实测）
- ControlNet Union 控运动为什么没用（两轮实测判死）
- 显存被后台进程抢走 50% 怎么定位（含 Windows 计划任务"自己复活"的对抗方案）
- 每个改动的回滚步骤

## 安装（挑一种）

**A. 直接让 agent 自己装（最省事）**：把本仓库链接丢给你的 agent，说「安装这个 skill」：
```
https://github.com/BaseenHolmes/h3-comfy-ops
```

**B. git clone 到对应 agent 的技能目录**（Windows 路径，Linux/macOS 换成 `~/<agent>/skills`）：
```powershell
# Codex CLI
git clone https://github.com/BaseenHolmes/h3-comfy-ops "$env:USERPROFILE\.codex\skills\h3-comfy-ops"
# Claude Code
git clone https://github.com/BaseenHolmes/h3-comfy-ops "$env:USERPROFILE\.claude\skills\h3-comfy-ops"
# DSH / 跨 agent 通用目录
git clone https://github.com/BaseenHolmes/h3-comfy-ops "$env:USERPROFILE\.agents\skills\h3-comfy-ops"
# WorkBuddy
git clone https://github.com/BaseenHolmes/h3-comfy-ops "$env:USERPROFILE\.workbuddy\skills\h3-comfy-ops"
```

**C. 只要文件**：
```powershell
curl.exe -L -o SKILL.md https://raw.githubusercontent.com/BaseenHolmes/h3-comfy-ops/main/SKILL.md
```

**D. 用 skills CLI（如果装了）**：`npx skills add BaseenHolmes/h3-comfy-ops`

## 前置条件

跑 SKILL.md 里的命令需要：

1. 目标算力机在**同一局域网**，且已配置 **SSH 免密**（`ssh Administrator@192.168.0.15` 能直接进）
2. ComfyUI 在 `http://192.168.0.15:8188`（无鉴权，脚本直接用）
3. 机器上有 H3 模型与 turbo LoRA

> IP 是作者内网的，换成你自己的；SKILL.md 里所有命令都是通用的 SSH + ComfyUI HTTP，不依赖特定主机名。

## 内容

| 文件 | 作用 |
|---|---|
| `SKILL.md` | 主文件：环境事实、决策矩阵、坑表、修复步骤、skill 边界 |
| `references/measured_baselines.md` | 全部实测数字（0.35/0.44 对照、CN 强度扫描、步数阶梯、量化对照） |
| `references/sage_and_quant.md` | SageAttention 修复全过程 + 四档量化底模的字节数/速度/显存 |
| `references/troubleshooting.md` | 报错 → 判据 → 处置（含日志原文） |
| `references/rollback.md` | 每个改动的回滚步骤 |

## 版本

- **v1.3.0**（2026-11-09）：0.35 vs 0.44 同图终测、124 帧、CN 逐帧判死、IndexTTS 守卫、SSH/编码三坑
- v1.2.0：0.44 并存、SageAttention 修复、GGUF/w6a8 判决
- v1.1.0：int4 底模、画质 LoRA、X2 细节 VAE、Pika Flex

## 边界

- 只覆盖 **Windows + NVIDIA + ComfyUI** 的 H3 部署；Linux 上路径与脚本要改
- 数字是 **4090D 24GB** 上的，4090 满血/5090 会有差异，但排序结论通常一致
- 涉及模型下载的部分只给来源与体积，不提供模型文件
