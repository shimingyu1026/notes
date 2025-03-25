# OpenTitian 设计验证方法

## 1 语言与工具

opentitian验证分为两部分：

### 1.1 基于功能测试的设计动态仿真

在每一个 DUT 代码文件夹内都包含有一个 UVM 验证平台、一份测试计划、一份整体的 DV 文档、一套测试用例，以及构建、运行测试和报告当前状态的方法。

使用 Systemverilog

### 1.2 形式属性验证（FPV）

某些 DUT 项目会包含有 SV 测试平台，以及用 SystemVerilog 断言（SVA）语言描述的设计属性。目前还在开发当中。

对于工具的使用，opentitian 使用 Synopsys 的 VCS 进行验证，选择 JasperGold 进行 FPV。（opentitian目标：推广支持Verilator、Yosys、cocoTB 等工具，达到工业验收级别）

## 2 验证完成的定义标准： Stages 和 Checklist

### 2.1 Stages

Opentitian 文档中有对项目开发 stage 的详细定义，下面列出验证阶段的 stage 定义：

**仿真验证阶段**

| 阶段 | 名称         | 定义                                   |
| ---- | ------------ | -------------------------------------- |
| V0   | 初始工作     | 测试平台开发中，测试计划编写           |
| V1   | 测试中       | 测试平台就绪，完成基本测试和覆盖率 90% |
| V2   | 测试完成     | 所有测试通过，覆盖率 90%               |
| V2S  | 安全对策验证 | 安全相关测试完成                       |
| V3   | 验证完成     | 100% 覆盖率，所有问题解决              |

**形式验证阶段**

| 阶段 | 名称         | 定义                         |
| ---- | ------------ | ---------------------------- |
| V0   | 初始工作     | 形式验证环境搭建             |
| V1   | 测试中       | 主要功能断言实现并通过       |
| V2   | 测试完成     | 90% 属性证明，75% 逻辑覆盖率 |
| V2S  | 安全对策验证 | 安全属性验证完成             |
| V3   | 验证完成     | 100% 属性证明，100% 覆盖率   |

在仓库的hjson文件中（一般为 `hw/ip/name/data/name.hjson`或者 `hw/ip/name/data/name.prj.hjson`）：

```json
{
    name:               "gpio"
    version:            "1.0"
    life_stage:         "L1"
    design_stage:       "D2"
    verification_stage: "V1"
    dif_stage:          "S0"
    notes:              "information shown on the dashboard"
}
```

### 2.2 Checklist

在 checklist 中定义了从一个阶段过渡到下一个阶段所需完成的任务列表。参照文档：[Checklist](https://opentitan.org/book/doc/project_governance/checklist/index.html "checklist")

## 3 文档

由 **测试计划文档** 和 **DV 文档** 组成，同时项目状态文档则跟踪工作在各个阶段的进展情况。下面是一些必要组件：

### 3.1 测试计划 Testplan

测试计划由两部分组成：

* **测试计划** ：从高层次概述了为验证设计规范中列出的所有设计功能而计划进行的测试列表。
* **功能覆盖率计划** ：从高层次概述了验证设计规范中列出的功能所需的功能覆盖点和覆盖交叉点列表，以确保这些功能都能通过测试列表进行测试。

Hjson编写，在每一个 DUT 的 data 文件夹内可以看到。testplanner 工具（dvsim）对Hjson进行支持

### 3.2 DV 文档

DV 文档除了详细阐述测试计划外，还涵盖了整体策略、目标、测试平台框图、接口 / 代理列表、VIP（虚拟接口组件）、参考模型、功能覆盖率模型、断言和检查器。如果适用，还会涉及 FPV 目标。该文档采用 Markdown 格式编写，可在每个 DUT 的相应 doc 目录中找到。

## 4 测试自动化

整个测试平台在以下三个关键领域实现了自动生成：

### 4.1 初始 UVM 测试平台生成

opentitian 提供了一个名为 uvmdvgen 的验证启动工具包。它可以为新的 DUT 完全自动生成完整的初始 DV 环境，包括文档（测试计划和 DV 文档）、完整的 UVM 环境（包括测试平台）、构建和运行测试的相关文件，以及一些通用测试用例。

***~~TODO 工具使用方法~~***

### 4.2 UVM 寄存器抽象层（RAL）模型

opentitian 提供了 reggen 工具自动生成 UVM RAL 模型。

### 4.3 测试平台自动化

针对参数化 DUT 的不同配置开发的通用 UVM。

## 5 代码复用

Opentitian 提供了一些常用的验证基础设施组件来辅助测试平台的开发。

### 5.1 DV Base Library

参照文档

### 5.2 Comportable IP DV Library

### 5.3 Common Verification Components

在 hw/dv/sv 目录下提供了多个即插即用的通用验证组件
