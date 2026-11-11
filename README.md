# LLVM BOLT ExtTSP Local Refinement Optimization

## 1. 项目简介

本项目基于 LLVM BOLT，设计并实现了一种针对 ExtTSP（Extended Traveling Salesman Problem）基本块布局算法的局部优化方法。

LLVM BOLT 是一种基于执行性能数据（Profile-Guided Optimization）的二进制优化工具，能够通过重新排列函数和基本块来改善程序的执行性能。

本项目在原有 ExtTSP 算法生成的布局基础上，引入 **Score-Guarded Local Refinement（基于评分保护的局部优化）**。

该方法通过有限范围内的基本块交换搜索，寻找能够进一步提高 ExtTSP 评分的布局，从而改善原始算法可能存在的局部次优结果。

## 2. 开发环境

- 操作系统：Ubuntu 24.04
- 开发环境：VMware 虚拟机
- 项目：LLVM BOLT
- 基准源码日期：2026 年 11 月 2 日
- Git Commit：`74c9938e2b57c708900db28348d8fc9fc0604eb9`
- 开发语言：C++
- 构建工具：CMake + Ninja
- 容器环境：Docker

## 3. 优化方案

### 3.1 原始算法

原版 ExtTSP 根据基本块执行频率、控制流跳转关系以及代码布局距离，为基本块生成优化后的排列顺序。

由于布局搜索采用启发式策略，生成的结果不一定是所有局部交换操作下的最优布局。

### 3.2 新增算法

本项目在原始 ExtTSP 排序结果之后增加局部交换优化。

主要步骤：

1. 使用原版 ExtTSP 生成初始基本块布局。
2. 计算当前布局的 ExtTSP 评分。
3. 在有限搜索窗口内尝试交换基本块。
4. 重新计算候选布局的 ExtTSP 评分。
5. 仅接受评分增益超过阈值的交换。
6. 重复有限轮搜索，得到最终布局。

### 3.3 主要设计特点

- **可选启用**：新增命令行选项，默认关闭。
- **评分保护**：只接受评分增加的候选布局。
- **入口保护**：不交换函数入口基本块。
- **限制搜索规模**：仅对符合规模要求的函数执行局部搜索。
- **控制算法开销**：限制搜索窗口和最大迭代轮数。
- **兼容原有流程**：关闭新功能时保留原版 ExtTSP 的排序逻辑。

优化参数：

| 参数 | 设置 |
|---|---|
| 适用基本块数量 | 4～64 |
| 搜索窗口 | 8 |
| 最大迭代轮数 | 2 |
| 最小评分增益 | 1.0 |
| 默认状态 | 关闭 |

## 4. 源码修改

本项目主要修改：

`bolt/lib/Passes/ReorderAlgorithm.cpp`

新增命令行参数：

```bash
-bolt-ext-tsp-local-refine=true
```

关闭优化：

```bash
-bolt-ext-tsp-local-refine=false
```

源码修改以 Git Patch 的形式提供：

`exttsp_local_refine.patch`

## 5. 实验结果

本项目使用不同 GCC 优化级别编译测试程序，验证新增算法的正确性和布局评分变化。

### 5.1 ExtTSP 评分测试

| 编译选项 | 接受交换次数 | 评分增益 | 正确性 |
|---|---:|---:|---|
| GCC -O1 | 0 | 0 | 通过 |
| GCC -O2 | 2 | +975.82 | 通过 |
| GCC -O3 | 2 | +975.82 | 通过 |

在 GCC -O2 和 GCC -O3 测试中，新增算法各接受了 2 次局部交换，使 ExtTSP 评分提高约 975.82。

在 GCC -O1 测试中，算法未找到满足条件的局部交换，因此保持原有布局。

### 5.2 正确性验证

对原始程序、原版 ExtTSP 优化程序以及开启新增优化后的程序进行输出比较。

测试结果表明，三种 GCC 优化配置下的程序输出均保持一致。

### 5.3 运行性能测试

使用 VMware Ubuntu 虚拟机进行多轮性能对照测试。

在 30 轮成组测试中：

- 原版 ExtTSP 运行时间中位数：107.3028 ms
- ExtTSP + Local Refinement 运行时间中位数：109.5151 ms
- 成对运行时间变化中位数：+0.0060%
- 新优化运行更快的轮次：15/30

测试环境存在较明显的时间波动，因此目前尚不能确认新增算法带来了稳定的实际运行加速或回退。

**本项目验证了 ExtTSP 评分改善，但不将评分改善直接等同于真实性能提升。**

## 6. Docker 镜像

本项目提供了包含修改后 LLVM BOLT 可执行程序的 Docker 运行镜像。

镜像已上传至 GitHub 仓库的 **Releases** 页面。

镜像文件：

`bolt-exttsp-refine-20261102.tar.gz`

### 6.1 导入镜像

下载 Release 中的 Docker 镜像压缩包后执行：

```bash
docker load -i bolt-exttsp-refine-20261102.tar.gz
```

### 6.2 查看 BOLT 版本

```bash
docker run --rm bolt-exttsp-refine:20261102
```

### 6.3 运行优化实验

```bash
docker run --rm bolt-exttsp-refine:20261102 \
  bash -lc '
    set -euo pipefail

    llvm-bolt ./branchy \
      -o /tmp/branchy.refine \
      -data=/work/branchy.fdata \
      -reorder-blocks=ext-tsp \
      -tsp-threshold=0 \
      -reorder-functions=none \
      -bolt-ext-tsp-local-refine=true

    diff -u <(./branchy) <(/tmp/branchy.refine)

    echo "PASS: Docker optimization test"
  '
```

该实验已在 Docker 容器中执行成功，实际接受 2 次交换，评分增益约为 975.82，程序输出比较通过。

**注意：该镜像是预编译 BOLT 的运行与实验环境，不是从 LLVM 源码重新编译的完整构建环境。**

## 7. 项目文件

本仓库提供以下主要交付内容：

| 文件 | 说明 |
|---|---|
| `exttsp_local_refine.patch` | BOLT 源码优化补丁 |
| `BOLT_ExtTSP_局部优化_设计与实验报告.docx` | 中文设计与实验分析报告 |
| GitHub Releases | Docker 运行镜像 |

详细的算法设计、源码修改位置、实验分析和局限性请参考 Word 报告。

## 8. 总结

本项目在 LLVM BOLT 原有 ExtTSP 基本块重排算法基础上，实现了一种基于评分保护的局部交换优化方法。

实验表明，新增算法能够在部分测试配置中进一步改善 ExtTSP 布局评分，并保持测试程序的功能正确性。

由于当前性能测试环境存在较大波动，尚未验证稳定的实际运行加速效果。

本项目的主要成果是完成了一种可开关、搜索范围受限、能够实际改善 ExtTSP 评分的局部布局优化，并通过 Docker 提供了可运行的实验环境。

---

**Project:** LLVM BOLT ExtTSP Local Refinement  
**Baseline:** LLVM commit `74c9938e2b57c708900db28348d8fc9fc0604eb9`  
**Platform:** Linux x86-64  
**Language:** C++
