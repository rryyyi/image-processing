# 实验与复现

本文整理仓库已提交的历史实验及复现条件。表格来自现有 CSV，文档维护时的单元测试和 smoke 验证不等于重跑完整基准。

## 数据与实验范围

真实配对基准为 [Water-Filling](https://github.com/seungjun45/Water-Filling) 提供的 20 对 Jung 文档，使用 `Original/` 与 `GroundTruth/` 下按 `rect_<编号>` 匹配的样本。原始数据不随本仓库分发。

该集合主要用于阴影和光照修复比较。基准构造 `PipelineConfig.from_mapping({"pages": {"enabled": False}})`，关闭页面检测以保持输出与参考图对齐。它不衡量页面检测、透视矫正或多页排序精度。

这 20 对样本参与了估计器选择和参数调整，报告的是同一小样本集合的结果，缺少独立留出测试集；不能据此推断大规模泛化效果或行业最优。完整 SD7K 训练/评测与 OCR CER/WER 评测尚未完成。

## 已提交的全图结果

数据来源：[benchmark_summary.csv](../results/tables/benchmark_summary.csv)，筛选 `region=full`，按样本计算后取算术均值。

| 方法 | PSNR ↑（dB） | SSIM ↑ | RMSE ↓ |
| --- | ---: | ---: | ---: |
| input | 12.932 | 0.792 | 0.240 |
| doc_image_tool | 15.918 | 0.810 | 0.176 |
| docshadow | 17.300 | 0.834 | 0.148 |
| ours | 18.989 | 0.843 | 0.127 |

`ours` 为 DocRestore。该集合上相对输入的全图 PSNR 均值增加 6.057 dB，相对 DocShadow 增加 1.689 dB；这不是每张图都提升相同幅度。

DocRestore 在 `jung_06`、`jung_09`、`jung_10` 等难例上落后 DocShadow，涉及强阴影、复杂图表和密集版面。可结合[逐图记录](../results/tables/benchmark_per_image.csv)、[失败排名](../results/tables/failure_ranking.csv)和[失败示例图](../results/figures/failure_cases.png)检查结果。

## 指标与实现限制

指标实现位于 [quality/assess.py](../src/docrestore/quality/assess.py)。

| 指标 | 含义 | 解读限制 |
| --- | --- | --- |
| PSNR | 归一化 RGB/BGR 三通道像素误差，使用 `data_range=1` | 对曝光、配准和颜色变化敏感 |
| RMSE | 归一化像素的均方根误差 | 越低越接近参考图，不直接等价于可读性 |
| 全图 SSIM | scikit-image 多通道结构相似度 | 仅为结构相似性指标，不替代 OCR 评测 |
| background_std / illumination_variation | 启发式纸面背景与光照变化 | 背景掩膜可能受布局和纸面颜色影响 |
| edge_energy / laplacian_variance | 边缘与高频清晰度 | 锐化和噪声也可能抬高指标 |
| readability_score | 0–100 的加权图像质量分数 | 无人工或 OCR 校准，不表示识别正确率 |

阴影区域由输入与参考图的低频亮度差得到，默认阈值 0.07，并进行形态学处理。该掩膜是评测启发式掩膜，不是真实人工标注。

**区域 SSIM 的现有限制：** `reference_metrics()` 仅在 SSIM score map 与二维掩膜形状完全一致时计算掩膜平均。对当前三通道图像，score map 通常仍含通道维度，分支会回退到全图 SSIM。因此现有 CSV 的 `shadow/non_shadow` SSIM 不应视为独立区域 SSIM；区域 PSNR 与 RMSE 则直接按二维掩膜索引计算。当前文档维护保留历史算法和表格，尚未修正这一实现或重新计算数据。

输入和参考图必须形状一致，否则函数抛出 `ValueError`。完全一致的图像可产生无穷 PSNR；空区域返回 NaN 指标，分析时应显式处理这些值。

### 耗时

CSV 保留了 `runtime_seconds`，但历史对照包含 CPU 主流程和可选深度模型，且没有独立的统一预热、GPU 同步、重复采样及硬件控制协议。平均时间不适合作为公平吞吐排名。

历史硬件和依赖见[实验日志](experiment_log.md)。主包 manifest 的耗时范围与基准外层计时也不同，详见[架构说明](architecture.md#输出与耗时的解释)。

## 复现历史基准

在仓库根目录执行，先按 [README](../README.md#快速开始)创建 Python 3.12 环境并激活：

```bash
python -m pip install -e ".[dev]"
python -m pytest
```

### 1. 准备数据与传统基线

以下目录布局是当前脚本的约定：

```text
third_party/
├── Water-Filling/
│   ├── Original/
│   └── GroundTruth/
├── Doc-Image-Tool/
│   └── function_method/DocShadowRemoval.py
└── DocShadow-SD7K/                  # 仅 docshadow 方法需要
data/models/docshadow/SD7K.pth      # 仅 docshadow 方法需要
```

按[参考记录](references.md)中的历史版本获取代码，以降低上游变更带来的差异。以下命令先克隆，再固定版本；如缩写无法唯一定位，应对照上游提交核实：

```bash
git clone https://github.com/seungjun45/Water-Filling.git third_party/Water-Filling
git -C third_party/Water-Filling checkout 9e1fee1

git clone https://github.com/jiangnanboy/Doc-Image-Tool.git third_party/Doc-Image-Tool
git -C third_party/Doc-Image-Tool checkout c7f3731
```

数据下载、第三方依赖和许可需另行确认。脚本会直接加载 Doc-Image-Tool 的 Python 文件；即使 `--methods input,ours` 没有选中它，当前初始化逻辑仍会加载该模块，因此这个仓库仍需准备。

### 2. 运行不含深度模型的基准

```bash
python scripts/run_benchmark.py --methods input,doc_image_tool,ours --save-all --output results/benchmark_local
```

这会计算相应方法的结果，不包含 DocShadow，也不会自动生成上表中的完整四方法对照。`--limit 2` 可用于先检查两对数据是否能正常运行，但小样本运行不是完整复现。

**输出副作用：** `--output` 控制图像、`per_image.csv` 和 `run_metadata.json` 的运行目录；脚本仍会覆盖 `results/tables/benchmark_summary.csv` 与 `benchmark_per_image.csv`。重跑前保存现有表格，提交结果时检查 `git diff`，避免将子集结果误当成完整基准。相同运行目录也可能保留旧文件，建议每次使用新目录。

### 3. 可选 DocShadow 基线

```bash
git clone https://github.com/CXH-Research/DocShadow-SD7K.git third_party/DocShadow-SD7K
git -C third_party/DocShadow-SD7K checkout b9395ed
python -m pip install -e ".[deep]"
```

按照上游说明取得官方 `SD7K.pth`，放到 `data/models/docshadow/SD7K.pth`。PyTorch 的 CPU/CUDA 构建和上游其他依赖需要按机器实际情况配置，安装 `.[deep]` 不代表所有上游环境已满足。

确认适配器和模型可运行后：

```bash
python scripts/run_benchmark.py --methods input,doc_image_tool,ours,docshadow --save-all --output results/benchmark_with_docshadow
```

默认 `--methods` 就包含 DocShadow；没有准备模型时应显式使用前面的三方法命令。第三方模型加载逻辑见 [adapters/docshadow.py](../src/docrestore/adapters/docshadow.py)。

## 其他实验与产物

| 实验 | 入口 | 已提交结果 |
| --- | --- | --- |
| 光照估计器比较 | [run_estimator_comparison.py](../scripts/run_estimator_comparison.py) | [estimator_summary.csv](../results/tables/estimator_summary.csv) |
| 模块消融 | [run_ablation.py](../scripts/run_ablation.py) | [ablation_summary.csv](../results/tables/ablation_summary.csv) |
| 参数敏感性 | [run_param_sensitivity.py](../scripts/run_param_sensitivity.py) | [sensitivity_summary.csv](../results/tables/sensitivity_summary.csv) |
| paper-guided 探索 | [run_paper_guided.py](../scripts/run_paper_guided.py) | [paper_guided_summary.csv](../results/tables/paper_guided_summary.csv) |
| 失败分析 | [analyze_failures.py](../scripts/analyze_failures.py) | [failure_ranking.csv](../results/tables/failure_ranking.csv) |
| 合成退化生成 | [generate_synthetic_degradations.py](../scripts/generate_synthetic_degradations.py) | 运行时生成本地数据，原始合成数据不入库 |

这些脚本使用仓库根目录、真实配对数据或已有实验输出中的约定路径。运行前阅读各脚本入口，确认输入已准备；部分脚本同样会覆盖 `results/tables/` 文件。

默认形态学估计器、关闭外观增强和分离亮度/颜色增益的原因见[技术决策](decisions.md)。当前 16 项单元测试覆盖五种估计器及若干几何、质量与修复行为，不代表所有场景、参数组合和后续加入的 `paper_guided` 都已充分验证。

## 后续评测重点

- 划分独立验证集和留出测试集，增加拍照设备、纸张颜色、版面和退化强度的覆盖。
- 修正区域 SSIM 的通道维度处理，再重新生成区域指标。
- 引入 OCR CER/WER，对比原图与修复图的下游任务增益。
- 分离页面检测/透视修复和光照修复的评测协议。
- 在统一硬件与输入尺寸下报告预热、重复次数、P50/P95、资源占用与失败率。

这些是待开展的工作，不是当前仓库已完成的结果。
