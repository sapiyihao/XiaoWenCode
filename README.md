# 我们的方法 · 实验工作区

> **定位**：本课题（纯 SOH 估计）的全部实验代码、中间产物与成图。
> **与文献复现的区别**：`../复现/` 是文献复现；本目录是**自研方法**。
> **三份定稿文档**（决定这里的一切参数）：
> - `../论文调研/03_标签制定方法.md` —— 标签口径
> - `../论文调研/04_特征工程方案.md` —— 通道 / 窗口 / 归一化
> - `../论文调研/05_主方法定稿.md` —— 架构 / 损失 / 训练协议

---

## 一、目录约定

```
我们的方法/
├── README.md          ← 本文件：约定、进度、产出清单
├── code/              ← 脚本，编号即执行顺序
├── out/
│   ├── tables/        ← 结果 CSV（utf-8-sig，便于 Excel 打开）
│   ├── checkpoints/   ← 模型权重（*.pt）
│   └── logs/          ← 运行日志（*.out / *.err）
└── figures/           ← 成图 PNG
```

**规则**：
1. 脚本一律**编号 + 幂等**（`--skip-existing`），中断后可续跑。
2. 所有中间产物落 `out/`，**不改原始 `../RAW_DATA/`**。
3. 每个阶段结束在本文档 §五 登记产出。

---

## 二、数据与环境

| 项 | 值 |
|---|---|
| 原始数据 | `../RAW_DATA/vin1..vin300.csv`（300 车，557.8 GiB，10 s/帧） |
| 主环境 | Python 3.13（pandas 2.2.3 / pyarrow / sklearn / matplotlib），**无 torch** |
| 训练环境 | `D:\enviroment\anaconda\envs\py312`（Python 3.12，torch 2.5.1+cu121） |
| GPU | RTX 3070 Laptop 8 GB，CUDA 可用 |
| 核数 | 16 逻辑核 |

**沙箱硬约束（踩过的坑，必须遵守）**：

| 限制 | 对策 |
|---|---|
| `multiprocessing` / joblib 线程池被拒（WinError 5） | 每个分片进程设 `OMP_NUM_THREADS=1`、`LOKY_MAX_CPU_COUNT=1` |
| `Start-Process` 起的子进程会被杀 | 分片一律用**受管后台作业**（一次 tool 调用 = 一个分片） |
| GPU 任务多进程会互撞显存 | GPU **单进程串行**，靠 batch 与 bf16 提速 |
| 并发写同一文件会写坏 | 分片用不重叠车号区间，或 `--skip-existing` |

---

## 三、脚本计划（编号即执行顺序）

| 编号 | 脚本 | 对应实验 | 主要产出 |
|---|---|---|---|
| `01_data_funnel.py` | 数据漏斗统计 | **E0** | `out/tables/funnel.csv`：车→会话→段→静置点→窗口逐层计数 |
| `02_labels.py` | SOC 三值 + SOH 标签 | E3 | `out/tables/labels.csv`、`out/tables/ocv_curve.csv` |
| `03_features.py` | 窗口切分 + 通道 + 静态标量 | E3 | `out/dataset.npz`（张量索引 + 归一化统计量） |
| `04_split.py` | VIN 级 5 折划分 | E0 | `out/tables/splits.json`、`out/tables/leak_check.csv`（泄漏量化） |
| `05_model.py` | ELAN + Mamba‑2 + 三项改进 | E2 | 模块实现（可 import） |
| `06_train.py` | 训练（全局固定超参） | E1/E2 | `out/checkpoints/*.pt`、`out/tables/train_log.csv` |
| `07_eval.py` | 评估与指标 | E1 | `out/tables/main_results.csv` |
| `08_ablation.py` | 8 组消融（C0–C7） | E2 | `out/tables/ablation.csv` |
| `09_baselines.py` | 外部基线 + 朴素/oracle 参照 | E1 | `out/tables/baselines.csv` |
| `10_robust.py` | 扰动矩阵 | E4 | `out/tables/robustness.csv` |
| `11_figures.py` | 出图 | — | `figures/*.png` |

> 编号调整以实际实现为准，但**实验与产出的对应关系不得变**。

---

## 四、开跑前的前置检查（E0 的一部分，必须先做）

| # | 检查项 | 通过标准 | 状态 |
|---:|---|---|---|
| 1 | 每车充电段时长分布 | 已知 ≥43 min 的段占比 | ⬜ |
| 2 | 静置点在 5 个温度区间的点数 | 极端温度段有点可用 | ⬜ |
| 3 | 同一会话窗口数与"独立事件数" | 两者都记账 | ⬜ |
| 4 | 会话内随机划分 vs VIN 级划分的 MAE 差 | 差距显著并记录 | ⬜ |

**第 1 项是硬门槛**：256 帧 = 2560 s ≈ 42.7 min，短于此的段切不出窗口。

---

## 五、进度与产出登记

| 阶段 | 状态 | 产出文件 | 备注 |
|---|---|---|---|
| 目录与约定 | ✅ 完成 | 本 README | 2026-11-20 |
| E0 漏斗与划分 | ⬜ 未开始 | — | 先做 §四 四项检查 |
| E3 标签与特征消融 | ⬜ 未开始 | — | 依赖 E0 |
| E1 主实验 | ⬜ 未开始 | — | 依赖 E3 定稿标签 |
| E2 架构消融 | ⬜ 未开始 | — | 最耗时 |
| E4 鲁棒性 | ⬜ 未开始 | — | 依赖已训 checkpoint |
| E5 泛化与效率 | ⬜ 未开始 | — | — |

---

## 六、判据（跑之前写死，事后不改）

| 项 | 阈值 | 依据 |
|---|---|---|
| SOH MAE | **≤ 1.5%** | 04 论文 1.11% / 我们复现 1.292% |
| 相对最好基线 | ≥10% 改善且 Wilcoxon p<0.05 | 防"打平就宣称更好" |
| 三改进组合 | ≥10% 相对改善，时延 < 2× | 否则如实写"中性" |
| 达到 oracle 的比例 | ≥40% | 05 我们只到 39% |
| 10 mV 电压噪声 | MAE 退化 ≤2% | 02 的 +0.030 pp 作下界 |
| 50% 丢帧 | MAE 退化 ≤20% | — |
| 种子稳定性 | 5 个种子方向不变 | 04 的种子敏感度 ±0.096 pp |

---

---

## 七、Git 版本控制约定

**远端**：`https://github.com/sapiyihao/XiaoWenCode.git`（仓库根 = 本目录 `我们的方法/`）

### 7.1 什么进 git，什么不进

| 进 git | 不进 git（已被 `.gitignore` 排除） |
|---|---|
| `code/**/*.py` | `out/checkpoints/`、`out/logs/`、`*.pt` |
| `README.md`、文档 | `*.npz`、`*.npy`、`*.parquet`（数据集张量） |
| **`out/tables/*.csv`（结果留痕）** | 原始 `../RAW_DATA/`（在仓库外） |
| `figures/**/*.png` | `__pycache__/`、`.venv/` |

**判据**：**代码 + 小体积结果 + 图**入库；**大数据 + 权重 + 日志**不入库（可重算，且会撑爆仓库）。

### 7.2 提交规范

```
<类型>: <一句话说明>

类型：feat 新功能 / fix 修复 / exp 实验配置 / data 数据脚本 / docs 文档 / refactor 重构
```

示例：
- `feat: 实现一维 ELAN + Mamba-2 主干`
- `exp: E2 消融 C0–C3 配置`
- `data: 标签流水线（三值 SOC + PCHIP）`

**粒度**：一个逻辑改动一次提交；**不要一次提交几百个文件**。

### 7.3 禁忌（对应"不要大范围回写"）

| ❌ 禁止 | 原因 |
|---|---|
| `git push --force` / `-f` | 会覆盖远端历史 |
| `git reset --hard` 到已推送的提交 | 本地与远端失联，需强推才能恢复 |
| `git rebase` 已推送的公共提交 | 重写历史 |
| `git commit --amend` 已推送的提交 | 同上 |
| 一次性 `git add -A` 扫全仓 | 容易把大文件/临时文件带进去 |

**允许的回退方式**：
- 撤销某次提交的改动 → `git revert <hash>`（生成新提交，不重写历史）
- 丢掉未提交的改动 → `git checkout -- <file>` 或 `git restore <file>`
- 只重写**尚未推送**的本地提交 → 允许，但推送前必须确认

### 7.4 本机特例（必须用这份配置）

本机 git 默认 **schannel 后端 TLS 握手失败**（`SEC_E_NO_CREDENTIALS`），已在本仓库本地配置改用 openssl：

```bash
git config http.sslBackend openssl
```

该设置**只作用于本仓库**，不影响全局配置。换机器克隆后如遇同样报错，照此设置一次即可。

### 7.5 首次推送需要凭据

远端为公开仓库，**读取可匿名，写入需 GitHub 凭据**（PAT 或 Git Credential Manager）。首次 `git push` 会要求认证，认证一次后由凭据管理器记住。

---

*本目录的所有参数以 `../论文调研/03、04、05` 三份定稿为准；如需变更，先改定稿再改代码。*
