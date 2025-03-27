# DVSIM

DVSIM 是一个EDA工具流管理器 / 构建与运行系统

## 架构

1. **流程步骤** ：仿真工具流含构建、运行、覆盖率合并（若启用）、覆盖率报告（若启用）步骤；其他 EDA 工具流仅到构建步骤。
2. **关键组件**
   * **DUT 配置（Hjson）** ：以树状文件结构存储，定义工具流和工具等信息，可递归导入辅助文件。
   * **解析器** ：解析 Hjson 文件构建流程管理器对象，处理变量替换。
   * **模式对象创建** ：生成构建、运行、测试、回归等模式对象，合并命令行和配置设置。
   * **可部署对象创建** ：依据测试和回归目标构建可部署对象，处理种子重设和构建依赖。
   * **调度器** ：依据依赖关系调度作业，监控状态和时间，记录结果。
   * **报告生成** ：汇总作业信息生成 Markdown 和 HTML 报告，可选择发布。

使用 FuseSoC 工具生成 SystemVerilog 源的[文件列表](https://opentitan.org/book/util/dvsim/doc/glossary.html#filelist)。

## DVSIM Testplanner

testplanner 解析 Hjson 格式的测试计划：

* 在文档中内联扩展测试计划为表格
* 结合仿真结果生成报告

### testplan 结构

#### 测试点（Testpoints）

* **属性** ：
* `name`：`lower_snake_case`命名法（如 `page_erase`）
* `stage`：验证阶段（V1/V2/V2S/V3）
* `desc`：详细描述（支持 Markdown）
* `tests`：关联测试列表（初始为空，`["N/A"]`表示不映射结果）
* `tags`：标签过滤（如 `["gls", "pa"]`）

#### 覆盖组（Covergroups）

* **属性** ：
* `name`：`lower_snake_case`命名（后缀 `_cg`）
* `desc`：覆盖功能描述

#### 共享测试计划

* **导入方式** ：
* `import_testplans`指定路径（相对 `$REPO_TOP`）
* 支持通配符替换（`{name}`），支持单值 / 多值扩展

### 使用示例

```bash
# 生成测试计划表格
./testplanner.py testplan.hjson -o output.html

# 结合仿真结果生成报告
./testplanner.py testplan.hjson -s results.hjson

# 标签过滤（包含foo，排除bar）
./testplanner.py testplan.hjson:foo:-bar
```
