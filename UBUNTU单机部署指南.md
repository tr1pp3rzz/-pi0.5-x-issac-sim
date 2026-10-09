# Ubuntu 单机部署与运行指南（π0.5 × LIBERO × Isaac Sim 5.0）

> **适用场景**：一台**双系统**机器（Windows 11 + Ubuntu 共用一个 1TB NVMe），
> Ubuntu 分区（199.3 GB，ext4）中已安装 Isaac Sim 5.0。
>
> **本指南目标**：在同一台机器的 Ubuntu 里，同时跑通「π0.5 策略服务端」和「Isaac Sim 仿真客户端」，
> **不使用第二台机器、不使用 Windows、不需要任何网络配置**。

---

## 0. 为什么这份指南和仓库里的官方文档不一样

仓库根目录的其他文档（`WINDOWS_DOCKER_GUIDE.md`、`README.md`）描述的是
**「Windows 客户端 + 远程 Ubuntu 服务端」的跨机器架构**。你是**单机双系统**，
所以那套流程里绝大多数步骤可以直接跳过。

| 官方跨机文档 | 本指南（单机 Ubuntu） |
| --- | --- |
| Windows 客户端 + 远程 Ubuntu 服务端 | 两端都在 Ubuntu |
| 需要局域网 / Tailscale / 记服务器 IP | 用 `127.0.0.1`，**零网络配置** |
| `ufw allow 8000/tcp` 开防火墙 | **不需要** |
| 安装 WSL2 + Docker Desktop | **不需要**（Ubuntu 是真 Linux） |
| PowerShell 语法：`$env:VAR`、反引号 `` ` `` 续行 | Bash 语法：`export VAR=`、反斜杠 `\` 续行 |
| `python.bat` / `isaaclab.bat` | `python.sh` / `isaaclab.sh` |
| Windows 跑 Docker 容器 | 直接跑；**仅"导出 LIBERO 场景"这一步用 Docker** |
| 改 `OPENPI_POLICY_HOST` 指向服务器 | **保持默认 `127.0.0.1` 即可** |

> ⚠️ **最大的坑**：把官方文档里的 Windows 命令直接抄到 Ubuntu 会全部失败
> （`$env:` 不存在、`.bat` 不存在、反引号不是续行符）。

---

## 1. 本机硬件约束（已实测，务必先读）

| 项目 | 实测值 | 影响 |
| --- | --- | --- |
| GPU | RTX 5070 Laptop，**8151 MiB（≈8 GB）** | 🔴 **最大风险**：`openpi_README.md` 要求推理 **> 8 GB** |
| 架构 | compute capability 12.0（Blackwell sm_120） | 🟠 需较新的 NVIDIA 驱动 / CUDA 12.8+，需实测 |
| CPU | Ryzen 9 8945HX（16C/32T） | ✅ 充足 |
| RAM | 31.2 GB | ⚠️ 刚好在 Isaac Sim 推荐值（32 GB）下沿 |
| Ubuntu 分区 | 199.3 GB（ext4） | ⚠️ 需确认剩余空间 |

**核心矛盾**：服务端和 Isaac Sim 客户端要**抢同一张 8 GB 显卡**。
- 服务端先启动 → 观察它吃掉多少显存 → 再启动 Isaac Sim，看剩下的够不够。
- 若 OOM，见 [第 8 节 显存优化](#8-显存优化与排障)。

---

## 2. 操作清单（总览）

复制这份清单，做一步勾一步：

- [ ] **阶段 0**：Ubuntu 环境体检
- [ ] **阶段 1**：安装 `uv` + 拉取 openpi 仓库 + 装依赖
- [ ] **阶段 2**：在 Ubuntu 启动策略服务（`127.0.0.1:8000`）
- [ ] **阶段 3**：在 Isaac Sim 的 Python 里安装 `openpi-client`
- [ ] **阶段 4**：连接测试（只验证网络与协议）
- [ ] **阶段 5**：准备场景（导出 bundle → 转 USD → 稳定性验证）
- [ ] **阶段 6**：按风险从低到高运行 8 个脚本
- [ ] **阶段 7**：核对日志签名，确认「真同步 / 真异步」
- [ ] **阶段 8**：显存优化与排障

---

## 阶段 0：Ubuntu 环境体检（必做）

重启进入 Ubuntu，打开终端，执行：

```bash
echo "=== OS ==="; cat /etc/os-release | head -3
echo "=== FREE DISK ==="; df -h / /home 2>/dev/null
echo "=== GPU/DRIVER ==="; nvidia-smi
echo "=== CUDA ==="; nvcc --version 2>/dev/null | tail -2 || echo "no nvcc"
echo "=== ISAAC SIM ==="; ls -d ~/isaacsim* /opt/isaacsim* 2>/dev/null; ls ~/.local/share/ov/pkg 2>/dev/null
echo "=== UV/PY ==="; (uv --version 2>/dev/null || echo "no uv"); python3 --version
echo "=== DOCKER ==="; (docker --version 2>/dev/null || echo "no docker")
echo "=== RAM ==="; free -h
```

### 需要确认的 5 件事

| # | 检查项 | 期望值 | 不符合时怎么办 |
| --- | --- | --- | --- |
| 1 | `cat /etc/os-release` 的 VERSION | **22.04**（官方仅测过 22.04）；24.04 多半可行但属未验证 | — |
| 2 | `df -h /` 的 Avail | **≥ 60 GB**（Isaac Sim 已占一部分，openpi 环境 + 权重还要几 GB） | 清理或把仓库放另一块盘 |
| 3 | `nvidia-smi` 的 Driver Version | **≥ 570**（50 系 Blackwell 需要较新驱动） | 升级 NVIDIA 驱动 |
| 4 | Isaac Sim 安装路径 + `python.sh` 存在 | 记下路径，后文记为 `$ISAAC_ROOT` | — |
| 5 | 是否有 `uv` | 没有就装（阶段 1.1） | — |

> 📝 把 `$ISAAC_ROOT` 替换成你的实际路径，例如 `~/isaacsim` 或 `~/.local/share/ov/pkg/isaac-sim-5.0`。
> Linux 版 Isaac Sim 的启动脚本是 **`$ISAAC_ROOT/python.sh`**（不是 Windows 的 `python.bat`）。

---

## 阶段 1：安装 openpi 服务端

### 1.1 安装 `uv`（若尚未安装）

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
# 让当前 shell 立即生效
source $HOME/.local/bin/env 2>/dev/null || export PATH="$HOME/.local/bin:$PATH"
uv --version
```

### 1.2 克隆仓库（含子模块）

> ⚠️ **强烈建议放在 Ubuntu 原生 ext4 分区（如 `~/openpi`），不要放在 `/mnt/d`。**
> NTFS 不支持 Unix 权限位和符号链接，`git submodule`、`uv` 缓存、
> Python 可执行文件都可能出问题。

```bash
cd ~
git clone --recurse-submodules https://github.com/XuepengGeng/pi05-isaacsim.git openpi
cd openpi
git submodule update --init --recursive
```

如果已经克隆过，只需在仓库根目录补齐子模块：

```bash
git submodule update --init --recursive
```

### 1.3 安装 Python 依赖

```bash
cd ~/openpi
GIT_LFS_SKIP_SMUDGE=1 uv sync
GIT_LFS_SKIP_SMUDGE=1 uv pip install -e .
```

> `GIT_LFS_SKIP_SMUDGE=1` 不能省：拉取 LeRobot 依赖时需要。
> 仓库 `.python-version` 指定 Python 3.11，`uv` 会自动准备好，**不用管系统里的 Python 3.14**。

### 1.4 验证 GPU 在 JAX 里可见（可选但推荐）

```bash
uv run python -c "import jax; print(jax.devices())"
```

应当打印出 `[CudaDevice(id=0)]`。若报错或退回 CPU，说明驱动/CUDA 不匹配
（50 系 Blackwell 是重点怀疑对象）。

---

## 阶段 2：启动策略服务（Ubuntu 本机）

### 2.1 普通启动

```bash
cd ~/openpi
uv run scripts/serve_policy.py --env LIBERO --port 8000
```

说明：
- `--env LIBERO` 默认映射到 `pi05_libero` 配置和
  `gs://openpi-assets/checkpoints/pi05_libero` 权重。
- **首次运行需要联网下载权重**（缓存在 `~/.cache/openpi`，可用 `OPENPI_DATA_HOME` 改路径）。
- 服务会监听 `0.0.0.0:8000`；因为服务端和客户端在同一台机器，
  **用 `127.0.0.1` 访问即可，不需要放行防火墙**。
- **等到日志出现"模型加载完成、开始监听"再进入下一步。**

**这个终端要一直开着**（它就是服务端）。

### 2.2 记录模式（可选，排查用）

想保存每次推理的真实输入图像与模型输出：

```bash
uv run scripts/serve_policy.py --env LIBERO --port 8000 --record
```

数据会写到 `policy_records/step_N.npy`。

### 2.3 自定义 checkpoint（可选）

```bash
uv run scripts/serve_policy.py policy:checkpoint \
  --policy.config=pi05_libero \
  --policy.dir=checkpoints/pi05_libero/my_experiment/20000
```

### 2.4 立刻观察显存占用

新开一个终端：

```bash
watch -n 1 nvidia-smi
```

记下服务端稳定后占用的显存（记为 `X` GB）。
**剩余 `8 - X` GB 就是留给 Isaac Sim 的全部预算**——这决定后面能否走通。

---

## 阶段 3：在 Isaac Sim 的 Python 中安装客户端

仓库自带的轻量客户端，**不要复制第二份**：

```bash
export ISAAC_ROOT="$HOME/isaacsim"          # ← 改成你的实际路径
export OPENPI_ROOT="$HOME/openpi"

"$ISAAC_ROOT/python.sh" -m pip install -e "$OPENPI_ROOT/packages/openpi-client"
```

> 只装轻量客户端即可，**不需要在 Isaac Sim 的 Python 里装整套 OpenPI 训练依赖**。

---

## 阶段 4：连接测试（只验证网络与协议）

```bash
"$ISAAC_ROOT/python.sh" \
  "$OPENPI_ROOT/isaac_sim/scripts/test_policy_connection.py" \
  --host 127.0.0.1 \
  --port 8000
```

- ✅ 成功 = 能收到动作，说明**协议、带宽、模型加载**都正常。
- ⚠️ 注意：该测试发送的是**模拟观测**，**不验证任务效果**，只验证连通性。
- 🚫 **不要**把地址写成 `OPENPI_POLICY_HOST=<外部IP>`；单机场景下
  `127.0.0.1` 就是正确值。历史故障记录里出现过误连 `127.0.0.1` 失败的情况——
  那是因为当时服务端在另一台机器上，**你的情况不一样**。

> 若未设置 `OPENPI_POLICY_HOST`，Isaac 脚本默认就连 `127.0.0.1`。
> 所以**单机场景你什么都不用设**。

---

## 阶段 5：准备场景（导出 bundle → 转 USD → 验证）

> 本阶段只在**第一次**做，且需要 `Docker`（用于在 LIBERO 容器里导出场景）。
> 如果 `$ISAAC_ROOT` 里已经有可用的 `libero_spatial_task2_ep0.usd`，可跳过。

### 5.1 导出 portable bundle

```bash
cd ~/openpi
docker compose -f examples/libero/compose.yml run --rm --no-deps runtime \
  bash -lc "source /.venv/bin/activate && cd /app && python examples/libero/export_scene_bundle.py \
    --task-suite-name libero_spatial --task-id 2 --episode-index 0 --seed 7 \
    --num-steps-wait 10 \
    --output-dir isaac_bridge/export/libero_spatial_task2_ep0"
```

> Docker 退出阶段可能打印 `free(): invalid pointer`——**这不代表导出失败**
> （发生在环境关闭阶段）。必须检查下面 4 项，而不是看退出码：

```bash
BUNDLE=~/openpi/isaac_bridge/export/libero_spatial_task2_ep0
test -f "$BUNDLE/scene.xml"     && echo "OK scene.xml"
test -f "$BUNDLE/manifest.json" && echo "OK manifest.json"
ls "$BUNDLE/assets" | wc -l
grep unresolved_files "$BUNDLE/manifest.json"   # 期望 "unresolved_files": []
```

### 5.2 转换为 USD（Isaac Lab）

> Linux 下 Isaac Lab 的启动脚本是 **`isaaclab.sh`**（Windows 是 `isaaclab.bat`）。

```bash
export ISAACLAB="$HOME/IsaacLab"      # ← 改成你的 Isaac Lab 安装路径
BUNDLE="$HOME/openpi/isaac_bridge/export/libero_spatial_task2_ep0"
USD="$HOME/openpi/isaac_bridge/usd_final/libero_spatial_task2_ep0.usd"

"$ISAACLAB/isaaclab.sh" -p \
  "$HOME/openpi/isaac_bridge/convert_libero_mjcf.py" \
  "$BUNDLE/scene.xml" \
  "$USD" \
  --manifest "$BUNDLE/manifest.json" \
  --headless \
  --experience isaaclab.python.kit
```

**正常摘要应类似**（这是判读成功的关键）：

```text
Applied free-body poses: 5
Repaired world-fixed joints: 4
Applied scalar joint initial states: 13
Physics scene: /physicsScene
GPU found/lost aggregate pair capacity: 8192
```

### 5.3 稳定性验证（120 帧）

```bash
"$ISAACLAB/isaaclab.sh" -p \
  "$HOME/openpi/isaac_bridge/validate_libero_usd.py" \
  "$USD" \
  --frames 120 \
  --headless
```

**期望**：表格中 table / robot base / bowl1 / bowl2 / cookies / ramekin / plate
的 120 帧位移**全部为 `0.000000000`**，且没有 PhysX aggregate capacity 报错。

> ⚠️ 必须保留整个 `usd_final/` 目录（含 `configuration/`、`materials/` 子目录）。
> **不要只复制最外层 USD**，它只有 1 KB 左右，是正常的组合入口。

---

## 阶段 6：按风险从低到高运行 8 个脚本

> 脚本都在 `~/openpi/isaac_sim/scripts/` 下。
> 标 ⚠️ 的脚本涉及 `omni` / `isaacsim` 交互，**需要在已加载场景的
> Isaac Sim Script Editor 中运行**；其余可在 Isaac Sim 的 `python.sh` 里跑。

| 步 | 脚本 | 做什么 | 默认是否动机械臂 |
| ---: | --- | --- | --- |
| 1 | `save_pi05_calibration_probe_isaacsim_5.py` | 检查场景单位、基座与末端参考位姿 | ❌ 只读 |
| 2 | `export_isaac_snapshot_script_editor.py` | 导出双相机 + 机器人快照 | ❌ 只读 |
| 3 | `read_pi05_state_isaacsim_5.py` | 检查 8 维 state 及坐标变换 | ❌ 只读 |
| 4 | `infer_pi05_once_isaacsim_5.py` | 采集观测并推理（**影子推理**，不执行动作） | ❌ 只读 |
| 5 | `test_pi05_ik_single_axis_isaacsim_5.py` | 单轴 IK 检查 | ⚠️ `EXECUTE_MOTION=False` |
| 6 | `run_pi05_closed_loop_debug_isaacsim_5.py` | 核心控制器检查 | ⚠️ `EXECUTE_CONTROL=False` |
| 7 | `run_pi05_conservative_task_isaacsim_5.py` | **同步**任务入口 | ✅ **会控制机械臂与夹爪** |
| 8 | `run_pi05_async_task_isaacsim_5.py` | **异步**任务入口 | ✅ 会控制 + 策略预取 |

**规则：每一步的结果确认后再进入下一步。**

### 6.0 启动前的环境变量（Bash 写法）

仓库里的 `set_env.example.ps1` 是 **PowerShell 版本，Ubuntu 下不能用**。
在同一个终端里手动 `export`（Isaac Sim 必须从这个终端启动）：

```bash
export OPENPI_POLICY_HOST=127.0.0.1          # 单机就是本机
export OPENPI_POLICY_PORT=8000
export OPENPI_ISAAC_SCRIPT_DIR="$HOME/openpi/isaac_sim/scripts"
export OPENPI_ISAAC_ARTIFACTS_DIR="$HOME/openpi/isaac_sim/artifacts"
# 标定参考文件（第 1 步会用到，文件需自行准备）
export OPENPI_LIBERO_REFERENCE_POSE="$HOME/openpi/data/calibration/libero_reference_pose.json"
```

> ⚠️ 环境变量**只对随后从该终端启动的进程生效**。
> 已经打开的 Isaac Sim 不会继承新变量——**改完必须重启 Isaac Sim**。

### 6.1 第 1 步的前置条件

`save_pi05_calibration_probe_isaacsim_5.py` 需要一份**与场景匹配的 LIBERO 参考位姿文件**，
默认路径 `data/calibration/libero_reference_pose.json`。该文件属于本地数据，
**缺失时探针会直接报错**，需要自行准备或用 `OPENPI_LIBERO_REFERENCE_POSE` 指定。

### 6.2 第 7 / 8 步之前必须确认的事

最后两个入口会**覆盖核心控制器的安全开关**，运行前必须确认：

- [ ] 场景 Prim 路径正确（见 [附录 B](#附录-b关键配置常量)）
- [ ] 两路相机正常，不是黑帧
- [ ] 坐标标定完成（左右镜像已修正）
- [ ] 工作空间与 `z=0.915 m` 桌面净空正确
- [ ] **知道急停方式**

### 6.3 两个关键开关（改一行即可回退对照）

在入口脚本里改：

```python
TASK_ARM_CONTROL_METHOD = "robosuite_osc"   # 默认；另一选择 "lula_ik"
TASK_ARM_ACTION_LIMIT_MODE = "robosuite"    # 默认；可选 "isaac_safe" / "unbounded"
```

| 开关 | 取值 | 含义 |
| --- | --- | --- |
| `TASK_ARM_CONTROL_METHOD` | `robosuite_osc` | 移植 robosuite 1.4.1 `OSC_POSE` 力矩控制（每个 PhysX step 重算 7 轴力矩） |
| | `lula_ik` | Lula IK 关节位置目标 + Isaac position drive（原始方案） |
| `TASK_ARM_ACTION_LIMIT_MODE` | `isaac_safe` | 逐维 `[-1,1]` + 额外 5 mm / 1° 模长限制（最保守） |
| | `robosuite` | 只保留 robosuite 官方逐维 `[-1,1]` |
| | `unbounded` | 全部撤掉，**仅用于对照实验**（可能单步发出厘米级目标） |

> 两个开关都**不会**关闭工作空间、IK、关节、力矩、夹爪接触等安全门。

---

## 阶段 7：核对日志签名（非常重要）

### 7.1 同步入口的期望签名

```text
RUNTIME_ENTRY_NAME=run_pi05_conservative_task_isaacsim_5.py
RUNTIME_ENTRY_MODE=synchronous
CONTROL_PROFILE=conservative_task_rollout
POLICY_ENDPOINT=127.0.0.1:8000
REPLAN_STEPS=5, MAX_CONTROL_STEPS=<与入口配置一致>
ASYNC_POLICY_INFERENCE=False
ARM_ACTION_DEBUG_SCALE=<与入口配置一致>
ARM_ACTION_LIMIT_MODE=<isaac_safe|robosuite|unbounded>
```

### 7.2 异步入口的期望签名

```text
RUNTIME_ENTRY_NAME=run_pi05_async_task_isaacsim_5.py
RUNTIME_ENTRY_MODE=asynchronous
CONTROL_PROFILE=asynchronous_task_rollout
ASYNC_POLICY_INFERENCE=True
```

### 7.3 判读规则

> **只要上述任何一项不符，就不要把这次结果记作「同步基线」。**

历史教训：曾经出现「同步入口实际上是异步入口的副本」，
导致 12 份日志全部是异步 profile，**所谓「同步也失败」是无效结论**。

另外：
- **同步推理期间 Timeline 暂停是刻意行为**（等模型返回）；异步模式才会继续播放。
- 任务结束时脚本会统一暂停 Timeline，**暂停按钮跳变 ≠ 仿真器故障**。

---

## 阶段 8：显存优化与排障

### 8.1 8 GB 显存不够时的处理顺序

| 顺序 | 手段 | 说明 |
| ---: | --- | --- |
| 1 | 先测服务端单独占用 | `nvidia-smi` 看 π0.5 推理实际吃多少 |
| 2 | Isaac Sim 用 **headless** | 转换/验证脚本已经用 `--headless`；交互脚本无法 headless |
| 3 | 降低渲染开销 | 关光追 / 降 RTX 采样 / 降分辨率 |
| 4 | 让核显接管显示输出 | Ryzen 9 8945HX 自带 Radeon 核显；把桌面输出接到核显可给独显腾出 1~2 GB |
| 5 | 分时运行 | 先只跑服务端（用快照推理验证协议），再单独跑 Isaac Sim |
| 6 | **换机器 / 云 GPU** | 到这个地步说明 8 GB 真的不够，这是文档推荐的分工方式 |

### 8.2 Blackwell（sm_120）风险

RTX 50 系是新架构，**较老的预编译 CUDA 轮子可能没有 sm_120 支持**：

- 阶段 1.4 的 `jax.devices()` 是**必做的第一道验证**。
- 若报 `no kernel image is available` 之类错误 → 需要升级到
  支持 CUDA 12.8+ 的 jaxlib / PyTorch 版本。

### 8.3 其他常见故障

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `No module named openpi_client` | 装错了 Python 环境 | 必须用 `$ISAAC_ROOT/python.sh` 安装 |
| Prim 不存在 / 相机全黑 | 没打开匹配的 USD，或 Prim 路径错 | 先跑第 2 步导出快照确认 |
| 脚本仍用旧目录 | 环境变量没被继承 | 在**启动 Isaac Sim 之前** export，然后重启 |
| 一按 Play 场景爆炸 | 世界固定关节锚点错误 | 见 `LIBERO_MJCF_TO_USD_排障与修复说明.md` §8 |
| `foundLostAggregatePairsCapacity` | 物体重叠或容量不足 | 先修位姿，再调容量（不能只调容量） |
| 观测左右镜像 | 相机未翻转 | 确认 `FLIP_AGENT_HORIZONTAL=True`、`FLIP_WRIST_HORIZONTAL=True` |
| 策略找错物体 | 指令文本不匹配 | 必须逐字使用 [附录 B](#附录-b关键配置常量) 的 prompt |
| `uv sync` 依赖冲突 | 缓存/版本问题 | `rm -rf .venv && uv sync`；必要时 `uv self update` |

---

## 附录 A：环境变量对照表（Windows → Ubuntu）

| Windows PowerShell（官方文档） | Ubuntu Bash（本指南） |
| --- | --- |
| `$env:OPENPI_POLICY_HOST = "IP"` | `export OPENPI_POLICY_HOST=127.0.0.1` |
| `$env:OPENPI_POLICY_PORT = "8000"` | `export OPENPI_POLICY_PORT=8000` |
| `. .\isaac_sim\set_env.example.ps1` | 手动 `export`（`.ps1` 在 Linux 不可用） |
| `` 反引号续行 `` ` `` | `\` 反斜杠续行 |
| `& "$IsaacRoot\python.bat" script.py` | `"$ISAAC_ROOT/python.sh" script.py` |
| `& "$isaacLab\isaaclab.bat" -p ...` | `"$ISAACLAB/isaaclab.sh" -p ...` |
| `${PWD}` | `$PWD` |

| 变量 | 本机建议值 | 作用 |
| --- | --- | --- |
| `OPENPI_POLICY_HOST` | `127.0.0.1` | 策略服务地址（单机） |
| `OPENPI_POLICY_PORT` | `8000` | 策略服务端口 |
| `OPENPI_ISAAC_SCRIPT_DIR` | `~/openpi/isaac_sim/scripts` | Script Editor 回退目录 |
| `OPENPI_ISAAC_ARTIFACTS_DIR` | `~/openpi/isaac_sim/artifacts` | 日志/图片/NPZ 输出 |
| `OPENPI_LIBERO_REFERENCE_POSE` | `~/openpi/data/calibration/libero_reference_pose.json` | 标定参考位姿 |
| `OPENPI_DATA_HOME` | 默认 `~/.cache/openpi` | 权重缓存位置 |

---

## 附录 B：关键配置常量

**当前场景对应 `libero_spatial` 的 task 2，语言指令必须逐字一致：**

```text
pick up the black bowl from table center and place it on the plate
```

> 🚫 不要改用 "between the plate and the ramekin" 之类的其他任务文本，否则策略会去找错误的碗。

| 配置 | 值 |
| --- | --- |
| 机器人根 Prim | `/scene/robot0_base/robot0_base` |
| 世界固定关节 | `/scene/joints/rootJoint_robot0_base` |
| 末端刚体 | `/scene/robot0_base/robot0_right_hand` |
| 主相机（`observation/image`） | `/scene/cameras/agentview` |
| 腕部相机（`observation/wrist_image`） | `/scene/robot0_base/robot0_right_hand/robot0_eye_in_hand` |
| 桌面高度 | `z = 0.900 m` |
| EEF 最低目标（含 15 mm 间隙） | `z = 0.915 m` |
| 世界旋转 | `QUAT_LIBERO_FROM_ISAAC_WXYZ = [1, 0, 0, 0]` |
| 世界平移 | `TRANSLATION_LIBERO_FROM_ISAAC = [0, 0, 0]` |
| 局部工具点偏移 | `[0, 0, 0.09649957] m`（随手掌旋转，非世界平移） |
| 观测尺寸 | 两路均 `224 × 224 × 3 uint8` |
| 状态 | 8 维：末端位置 3 + 轴角姿态 3 + 两个手指关节位置 |

> ⚠️ 更换 USD 场景 / 任务 / 桌子几何后，**上表中的高度、偏移、标定都必须重新测量**，
> 不能沿用这些常数。`isaac_sim/calibration/cameras_v001.json` 也只是当前场景的参考标定。

---

## 附录 C：已知限制（务必知晓）

1. **无逐回合确定性重置（episode reset）** —— P0 未实现。
   控制器只使用 Stage 当前状态并稳定 1 秒，重复试验的起点不一致。
2. **无任务成功自动判定** —— P0 未实现。
   summary 中明确写入 `task_success_automatically_evaluated=false`，
   可能出现"已完成任务却被记为失败"。
3. **MuJoCo 与 PhysX 存在系统性差异**（接触/摩擦/驱动/渲染），
   **场景转换成功和协议连通都不能证明行为等价**。
4. **因此：不要直接用 Isaac Sim 的运行结果计算与 LIBERO 可比的成功率。**
5. 修改光照、相机、控制模式或动作缩放后，应记录配置并重新做对照实验。

> 📌 文档的观点很明确：**当前最优先的工程项不是放宽阈值，而是补齐
> episode reset 和自动成功判定**——否则成功率没有可比性。

---

## 附录 D：命令速查

```bash
# ── 服务端 ──────────────────────────────────────────────
cd ~/openpi
uv run scripts/serve_policy.py --env LIBERO --port 8000
uv run scripts/serve_policy.py --env LIBERO --port 8000 --record

# ── 客户端安装（一次性）────────────────────────────────
"$ISAAC_ROOT/python.sh" -m pip install -e "$HOME/openpi/packages/openpi-client"

# ── 连接测试 ────────────────────────────────────────────
"$ISAAC_ROOT/python.sh" "$HOME/openpi/isaac_sim/scripts/test_policy_connection.py" \
  --host 127.0.0.1 --port 8000

# ── 显存观察 ────────────────────────────────────────────
watch -n 1 nvidia-smi

# ── 导出场景（需要 Docker）──────────────────────────────
cd ~/openpi
docker compose -f examples/libero/compose.yml run --rm --no-deps runtime \
  bash -lc "source /.venv/bin/activate && cd /app && python examples/libero/export_scene_bundle.py \
    --task-suite-name libero_spatial --task-id 2 --episode-index 0 --seed 7 \
    --num-steps-wait 10 --output-dir isaac_bridge/export/libero_spatial_task2_ep0"

# ── 转换 USD ────────────────────────────────────────────
"$ISAACLAB/isaaclab.sh" -p "$HOME/openpi/isaac_bridge/convert_libero_mjcf.py" \
  "$HOME/openpi/isaac_bridge/export/libero_spatial_task2_ep0/scene.xml" \
  "$HOME/openpi/isaac_bridge/usd_final/libero_spatial_task2_ep0.usd" \
  --manifest "$HOME/openpi/isaac_bridge/export/libero_spatial_task2_ep0/manifest.json" \
  --headless --experience isaaclab.python.kit

# ── 稳定性验证 ──────────────────────────────────────────
"$ISAACLAB/isaaclab.sh" -p "$HOME/openpi/isaac_bridge/validate_libero_usd.py" \
  "$HOME/openpi/isaac_bridge/usd_final/libero_spatial_task2_ep0.usd" \
  --frames 120 --headless
```

---

## 相关文档

- [项目首页 README](README.md) —— 全局架构与快速开始
- [Isaac Sim 集成说明](README%20(1).md) —— 脚本、Prim、控制开关详解
- [MJCF → USD 桥接](README%20(2).md) —— 转换工具与最小流程
- [MJCF → USD 排障与修复说明](LIBERO_MJCF_TO_USD_排障与修复说明.md) —— 场景爆炸等 10 类问题根因
- [π0.5 LIBERO 失败排查报告](PI05_LIBERO_FAILURE_AUDIT_20260901.md) —— 阈值、同步/异步与已知限制
- [OpenPI 中文说明](openpi_README.md) —— 模型、权重、显存要求、PyTorch 支持
- [Windows Docker 跨机指南](WINDOWS_DOCKER_GUIDE.md) —— **跨机器**方案（本机不适用，可作对照）
- [远程推理](remote_inference.md) —— WebSocket 协议与客户端调用示例
