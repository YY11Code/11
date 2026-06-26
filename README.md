# 11
11# 单 checkpoint 失败 interface 分析

找出 **DockQ 低** 的 antibody–antigen interface，算抗原/CDR 叠合 RMSD，并导出 PyMOL 图。

---

## 怎么跑

```bash
cd /lustre/grp/cmclab/share/zhengyi/xfold-train-new_backup/foldbench/analysis_failures-single-ckpt

sbatch run.sh \
  --predict-dir /path/to/latest_step_32000_epoch_32_...
```

示例（step 32000 checkpoint）：

```bash
sbatch run.sh \
  --predict-dir /lustre/grp/cmclab/share/zhengyi/xfold-train-new_backup/foldbench/output_20260611_140322/latest_step_32000_epoch_32_20260401_171045
```

`--predict-dir` 填**有 `pdb_top1_detail.csv` 的那一层 checkpoint 目录**（先跑过 `hw_pipeline-cleand-abag.sh` 评估）。

日志：`logs/failure_analysis_foldbench_<jobid>.out`

`run.sh` 默认提交 **dcu** 分区、**独占整节点**（128 CPU + ~1TB 内存）。Step 3 PyMOL 并行默认封顶 **8**；进程池崩溃后会自动串行重试失败条目。

---

## 跑完得到什么

**「失败 interface」**（下文简称 失败条目）：该 checkpoint 在 `pdb_top1_detail.csv` 里，**top-1 预测的 DockQ < 0.23** 的 antibody–antigen interface。  
例如某次跑完共 **56 条**；它们写入 `failed_interfaces.csv`，也是后续所有导出的**母集**。

RMSD **> 2 Å** 只用于 **PDF 页筛选**和**柱状图**，**不**用来决定谁进失败列表。

```
<predict-dir>/analysis_failures/
│
├── failed_interfaces.csv
│   └── 失败条目主表（N 行，N = DockQ<0.23 的 interface 数）
│       列含：interface_name, dockq_score, antigen_rmsd, cdr1/2/3_rmsd
│           
│
├── category_summary.pdf
│   └── 5 根柱状图：DockQ 失败总数 + 抗原/CDR1/2/3 中 RMSD>2Å 各多少条
│
├── pse_dockq/                         # DockQ 视角（与 foldbench 评估一致）
│   │   对齐：pred 整体按 OST 变换矩阵叠到 GT（来自 detail/*_structure_ost.json）
│   │   高亮：GT 抗体蓝 / 抗原青，pred 抗体红 / 抗原橙；其余灰
│   ├── 8ivx-assembly1_A_B.pse         # PyMOL session，用 PyMOL 打开可交互
│   ├── 8bf4-assembly1_A_B.pse         # … 共 N 个，与 CSV 行一一对应
│   ├── …
│   └── all.pdf                        # N 页；每页一条快照；页标 interface 名 + DockQ
│
├── pse_antigen_rmsd_gt2/              # 抗原 RMSD 视角
│   │   对齐：pred 抗原链 CA → GT 抗原链 CA（PyMOL align）
│   │   高亮：GT 抗原青 / pred 抗原橙；其余灰
│   ├── *.pse                            # N 个（每条失败条目都有，不论 RMSD 大小）
│   └── all.pdf                          # M₁ 页；仅 antigen_rmsd > 2Å；页标 interface 名 + DockQ + Ag RMSD
│
├── pse_cdr1_rmsd_gt2/                 # CDR1 RMSD 视角
│   │   对齐：pred 抗体链 CA → GT CDR1 残基 CA（IMGT 标注 CDR1）
│   │   高亮：GT CDR1 蓝 / pred CDR1 红；其余灰
│   ├── *.pse                            # N 个
│   └── all.pdf                          # M₂ 页；仅 cdr1_rmsd > 2Å；页标 interface 名 + DockQ + CDR1 RMSD
│
├── pse_cdr2_rmsd_gt2/                 # CDR2 RMSD 视角
│   │   对齐：pred 抗体链 CA → GT CDR2 残基 CA（IMGT 标注 CDR2）
│   │   高亮：GT CDR2 蓝 / pred CDR2 红；其余灰
│   ├── *.pse                            # N 个
│   └── all.pdf                          # M₃ 页；仅 cdr2_rmsd > 2Å；页标 interface 名 + DockQ + CDR2 RMSD
│
└── pse_cdr3_rmsd_gt2/                 # CDR3 RMSD 视角
    │   对齐：pred 抗体链 CA → GT CDR3 残基 CA（IMGT 标注 CDR3）
    │   高亮：GT CDR3 蓝 / pred CDR3 红；其余灰
    ├── *.pse                            # N 个
    └── all.pdf                          # M₄ 页；仅 cdr3_rmsd > 2Å；页标 interface 名 + DockQ + CDR3 RMSD
```

**文件命名**：CSV 里 `8ivx-assembly1|A|B` → 各目录下 `8ivx-assembly1_A_B.pse` / 对应 PDF 页。

**`.pse` 与 `all.pdf` 的区别**：

| | `.pse` | `all.pdf` |
|---|--------|-----------|
| 用途 | PyMOL 交互查看、改视角 | 快速翻页浏览，无需开 PyMOL |
| 条数 | 五个目录均为 **N 个** | **N** 或 **M₁–M₄**（见下表） |
| 标签 | 场景内 3D 文字（DockQ / RMSD） | 页左上角 2D  overlay（interface 名 + 数值） |

目录名里的 `rmsd_gt2` 只约束 **`all.pdf` 收哪些页**，不约束 `.pse` 是否导出。

---

## 四步在干什么（一句话版）

| 步骤 | 做什么 |
|------|--------|
| 1 | 从评估结果里挑出 **DockQ < 0.23** 的失败 interface |
| 2 | 算 **抗原、CDR1/2/3** 的叠合 RMSD，写入 CSV |
| 3 | 按类别导出 **每条 `.pse` + 目录内 `all.pdf`**（多线程） |
| 4 | 画 **柱状图** `category_summary.pdf` |

默认阈值：DockQ < **0.23**（定义失败），RMSD > **2 Å**（仅 PDF / 柱状图）。

---

## 五个文件夹分别看什么

**`.pse` 规则（五个目录相同）**：每条失败条目各有一个 `.pse`（共 N 个，N = `failed_interfaces.csv` 行数）。

**`all.pdf` 规则（五个目录不同）**：见下表第二列。

| 文件夹 | `all.pdf` 收录条件 | 视角 / 对齐 |
|--------|-------------------|-------------|
| `pse_dockq/` | 全部 N 条失败条目 | DockQ：pred 按 OST 变换叠到 GT，看 interface |
| `pse_antigen_rmsd_gt2/` | 抗原 RMSD > 2 Å 的子集（M₁ 条） | 只对齐抗原链，高亮抗原 |
| `pse_cdr1_rmsd_gt2/` | CDR1 RMSD > 2 Å 的子集（M₂ 条） | 只对齐 CDR1 |
| `pse_cdr2_rmsd_gt2/` | CDR2 RMSD > 2 Å 的子集（M₃ 条） | 只对齐 CDR2 |
| `pse_cdr3_rmsd_gt2/` | CDR3 RMSD > 2 Å 的子集（M₄ 条） | 只对齐 CDR3 |

举例 N = 56：五个目录各有 **56 个** `.pse`；`pse_dockq/all.pdf` 为 **56 页**；若 M₁ = 21，则 `pse_antigen_rmsd_gt2/all.pdf` 为 **21 页**。

同一条失败条目可同时出现在多个 `all.pdf` 里（例如抗原与 CDR3 的 RMSD 均 > 2 Å）。

**颜色（各文件夹统一）：** GT 抗体蓝 / 抗原青，Pred 抗体红 / 抗原橙，其余灰色。  
**图上数字：** `.pse` 内为 DockQ / RMSD；`all.pdf` 页左上角另有 interface 名 + 数值。

---

## 柱状图怎么读

`category_summary.pdf` 共 **5 根柱**：DockQ、Ag、CDR1、CDR2、CDR3。

- 柱高 = **条数**
- 柱顶如 `21 (37.5%)` = 21 条，占 **全部失败条目**（如 56 条）的 37.5%
- **DockQ 柱** = 全部失败条目；**Ag / CDR 柱** = 其中 RMSD > 2 Å 的子集

---

## `failed_interfaces.csv` 主要列

| 列 | 意思 |
|----|------|
| `interface_name` | 如 `8ivx-assembly1\|A\|B` |
| `dockq_score` | DockQ 分数 |
| `antigen_rmsd` | 抗原叠合 RMSD (Å) |
| `cdr1/2/3_rmsd` | 各 CDR 叠合 RMSD (Å) |
| `gt_path` / `pred_path` | GT 和预测结构路径 |
| `antibody_chain` / `antigen_chain` | GT 链号 |
| `antibody_chain_pred` / `antigen_chain_pred` | Pred 链号 |

---

## 其他用法

```bash
# 已有 CSV，只重导出图和柱状图
sbatch run.sh --predict-dir /path/to/checkpoint --skip-rmsd

# 改阈值
sbatch run.sh --predict-dir /path/to/checkpoint --dockq-threshold 0.23 --rmsd-threshold 2.0

# 指定线程数
sbatch run.sh --predict-dir /path/to/checkpoint --workers 12

# 高质量 PDF（慢）：ray trace 2400x1800
sbatch run.sh --predict-dir /path/to/checkpoint --no-fast-png
```

---

## 建议怎么看

1. 先看 **`category_summary.pdf`** — 失败里抗原/CDR 问题各占多少  
2. 再查 **`failed_interfaces.csv`** — 具体条目和数值  
3. 打开对应 **`pse_*` 目录**：交互看 `.pse`；快速翻页看 **`all.pdf`**

