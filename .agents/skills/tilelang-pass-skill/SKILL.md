---
name: tilelang-pass-skill
description: TileLang-Ascend TIR Pass 开发与修改指南。用于新增或修改编译器 IR 变换 Pass，包括 C++ Pass（src/transform/）和 Python Pass 封装（tilelang/transform/）。当用户要求编写新 Pass、修改现有 Pass、调试 Pass 逻辑、或理解编译流水线时触发此 skill。
---

# TileLang-Ascend Pass 开发指南

## 概述

TileLang-Ascend 的编译流水线通过一系列 TIR Pass 将前端 Python DSL 逐步 lower 为可在昇腾 NPU 上执行的 Ascend C 代码。Pass 分为两层实现：

- **C++ Pass**（`src/transform/*.cc`）：核心 IR 变换逻辑，性能敏感
- **Python 封装**（`tilelang/transform/__init__.py`）：通过 FFI 暴露给 Python 编译流水线

## 核心原则

### 原则 1：理解编译流水线再动手
- ✅ 先阅读 [pass-pipeline.md](references/pass-pipeline.md) 了解 Pass 执行顺序
- ✅ 确认你的 Pass 应该插入在哪个阶段
- ❌ 禁止：不了解上下游 Pass 就盲目修改 IR

### 原则 2：参考同类 Pass 实现
- ✅ 在 `src/transform/` 中找到功能最相似的 Pass 作为模板
- ✅ 遵循现有的代码风格和注册模式
- ❌ 禁止：从零开始写 Pass 而不参考已有实现

### 原则 3：双层修改不可遗漏
新增 Pass 时必须同步完成：
1. C++ 实现 + FFI 注册（`src/transform/`）
2. Python 封装（`tilelang/transform/__init__.py`）
3. 插入编译流水线（`tilelang/engine/phase.py`）
4. 如需配置开关，添加 PassConfigKey（`tilelang/transform/pass_config.py`）

### 原则 4：保证对已有算子无回归
- ✅ 修改 Pass 后运行 `bench_test.sh` 验证全量算子
- ✅ 新增 Pass 添加对应测试到 `testing/python/`
- ❌ 禁止：提交未经验证的 Pass 修改

### 原则 5：尽量不修改 TVM 原生 Pass
- ✅ 通过新增 TileLang 自己的 Pass 解决问题（放在 `src/transform/` 或 `tilelang/transform/`）
- ✅ 如果必须修改 TVM 原生 Pass，需在 PR 中说明原因并评估对 TVM 上游合并的影响
- ❌ 禁止：为了便利性直接修改 `tir.transform.*` 的行为

### 原则 6：新增 Pass 保证功能正交
- ✅ 每个 Pass 应只负责一种明确的 IR 变换，与已有 Pass 职责不重叠
- ✅ 新增前检查现有 Pass 是否已覆盖目标功能（参考 [pass-pipeline.md](references/pass-pipeline.md)）
- ✅ 如需扩展现有功能，优先在已有 Pass 中增量修改，而非新增冗余 Pass
- ❌ 禁止：创建与已有 Pass 功能重叠或存在顺序依赖耦合的 Pass

### 原则 7：新增 Pass 必须配套测试
- ✅ 每个新 Pass 至少包含一个 IR 级别测试（验证 IR 变换的正确性）
- ✅ 每个新 Pass 至少包含一个端到端功能测试（验证算子计算结果正确）
- ✅ 测试应覆盖 Pass 的核心逻辑路径和边界条件
- ❌ 禁止：提交无测试的 Pass

## 开发流程

### 阶段一：需求分析

1. 明确 Pass 的目标：对 IR 做什么变换？输入 IR 长什么样？输出 IR 应该是什么样？
2. 确定 Pass 在流水线中的位置（参考 [pass-pipeline.md](references/pass-pipeline.md)）
3. 确定是只读分析（Visitor）还是 IR 变换（Mutator）
4. 确定是否需要 PassConfig 开关

### 阶段二：C++ Pass 实现

参考 [pass-cpp-patterns.md](references/pass-cpp-patterns.md) 中的模板和模式。

#### 基本步骤：

1. 在 `src/transform/` 下创建 `your_pass_name.cc`
2. 选择合适的基类（Visitor 或 Mutator）
3. 实现 Pass 逻辑
4. 注册 FFI 接口
5. 在 `CMakeLists.txt` 中添加源文件（如未使用 glob）

#### 最小 C++ Pass 模板：

```cpp
#include <tvm/tir/analysis.h>
#include <tvm/tir/stmt_functor.h>
#include <tvm/tir/transform.h>

#include "common/attr.h"

namespace tvm {
namespace tl {

using namespace tir;

// --- Pass 实现类 ---
class YourPassRewriter : public arith::IRMutatorWithAnalyzer {
 public:
  static PrimFunc Substitute(PrimFunc f) {
    arith::Analyzer analyzer;
    YourPassRewriter rewriter(&analyzer);
    PrimFuncNode *fptr = f.CopyOnWrite();
    fptr->body = rewriter.VisitStmt(f->body);
    return f;
  }

 private:
  using arith::IRMutatorWithAnalyzer::IRMutatorWithAnalyzer;

  // 重写需要变换的 IR 节点
  Stmt VisitStmt_(const ForNode *op) final {
    // 你的变换逻辑
    return arith::IRMutatorWithAnalyzer::VisitStmt_(op);
  }
};

// --- Pass 创建函数 ---
using namespace tir::transform;

Pass YourPassName() {
  auto pass_func = [=](PrimFunc f, IRModule m, PassContext ctx) {
    return YourPassRewriter::Substitute(std::move(f));
  };
  return CreatePrimFuncPass(pass_func, 0, "tl.YourPassName", {});
}

// --- FFI 注册 ---
TVM_REGISTER_GLOBAL("tl.transform.YourPassName")
    .set_body_typed(YourPassName);

}  // namespace tl
}  // namespace tvm
```

### 阶段三：Python 封装

在 `tilelang/transform/__init__.py` 中添加：

```python
def YourPassName():
    """YourPassName - 简要描述

    Returns
    -------
    fpass : tvm.transform.Pass
        The result pass
    """
    return _ffi_api.YourPassName()  # type: ignore
```

### 阶段四：接入编译流水线

在 `tilelang/engine/phase.py` 的合适位置插入：

```python
# 根据 Pass 功能选择插入到 LowerAndLegalize 或 OptimizeForTarget
mod = tilelang.transform.YourPassName()(mod)
```

### 阶段五：添加 PassConfig 开关（可选）

如果 Pass 需要配置开关：

1. 在 `tilelang/transform/pass_config.py` 中添加：
```python
class PassConfigKey(str, Enum):
    # ...existing keys...
    TL_YOUR_PASS_CONFIG = "tl.your_pass_config"
```

2. 在 C++ Pass 中读取配置：
```cpp
Pass YourPassName() {
  auto pass_func = [=](PrimFunc f, IRModule m, PassContext ctx) {
    bool enabled = ctx->GetConfig<Bool>("tl.your_pass_config", Bool(false)).value();
    if (!enabled) return f;
    return YourPassRewriter::Substitute(std::move(f));
  };
  return CreatePrimFuncPass(pass_func, 0, "tl.YourPassName", {});
}
```

3. 用户使用时：
```python
pass_configs = {
    tilelang.PassConfigKey.TL_YOUR_PASS_CONFIG: True,
}
@tilelang.jit(out_idx=[...], pass_configs=pass_configs)
def my_kernel(...):
    ...
```

### 阶段六：测试与验证

每个新增或修改的 Pass 必须包含两类测试：**IR 级别测试** 和 **端到端功能测试**。

#### 6.1 IR 级别测试

IR 测试的核心思路：构造输入 IR → 运行目标 Pass → 验证输出 IR 符合预期。

框架改动会改变 IR 形态，通过 IR 快照对比可以快速发现回归问题。

##### 测试模板

```python
"""test_pass_your_pass_name.py
放置在 testing/python/ 下
"""
import pytest
import tilelang
import tilelang.language as T
import tilelang.transform
from tilelang import tvm as tvm
from tvm import tir


def construct_input_ir():
    """构造用于测试的输入 PrimFunc"""
    M, N = 128, 256

    @T.prim_func
    def func(
        A: T.Tensor((M, N), "float16"),
        B: T.Tensor((M, N), "float16"),
    ):
        with T.Kernel(1, is_npu=True) as cid:
            a_ub = T.alloc_ub((M, N), "float16")
            T.copy(A[0, 0], a_ub)
            T.tile.abs(a_ub, a_ub)
            T.copy(a_ub, B[0, 0])

    return func


def test_your_pass_ir_transform():
    """验证 Pass 对 IR 的变换是否正确"""
    func = construct_input_ir()
    mod = tvm.IRModule({"main": func})

    # 如需运行前置 Pass（保证输入 IR 满足 Pass 前提条件）
    mod = tilelang.transform.AscendInferBufferScope()(mod)

    # 运行目标 Pass
    mod_after = tilelang.transform.YourPassName()(mod)

    # --- 验证方式 1：检查 IR 文本中的关键特征 ---
    ir_text = str(mod_after)
    # 检查 Pass 插入/删除/修改了预期的 IR 结构
    assert "expected_keyword" in ir_text, f"Pass 未插入预期的 IR 结构"
    assert "removed_pattern" not in ir_text, f"Pass 未移除应删除的 IR 结构"

    # --- 验证方式 2：检查生成代码中的特征 ---
    # 适用于需要验证最终代码生成效果的场景
    # code = tilelang.engine.lower(func).get_source()
    # assert "expected_codegen_pattern" in code

    # --- 验证方式 3：结构化检查 IR 节点属性 ---
    result_func = mod_after["main"]
    # 检查函数属性
    # assert "your_attr" in result_func.attrs
    # 遍历 IR 节点检查结构
    # ...


def test_your_pass_preserves_unrelated_ir():
    """验证 Pass 不会修改不应变换的 IR"""
    func = construct_unrelated_ir()
    mod = tvm.IRModule({"main": func})
    mod_before_text = str(mod)

    mod_after = tilelang.transform.YourPassName()(mod)
    mod_after_text = str(mod_after)

    # 对于不匹配条件的 IR，Pass 应保持不变
    assert mod_before_text == mod_after_text, "Pass 修改了不应变换的 IR"
```

##### IR 文本特征检查的常用模式

```python
ir_text = str(mod_after)

# 1. 验证同步指令插入
assert "set_flag" in ir_text or "wait_flag" in ir_text

# 2. 验证 buffer scope 映射
assert 'scope = "local.UB"' in ir_text

# 3. 验证循环结构变换
assert "for (v, 0, 8)" in ir_text  # 循环被 unroll

# 4. 验证属性标注
assert '"memory_scope": "L1"' in ir_text
```

##### 生成代码检查模式

```python
# 对于需要验证最终 Ascend C 代码的场景
artifact = tilelang.engine.lower(func)
code = artifact.get_source()

# 验证生成代码包含预期指令
assert "DataCopy" in code
assert "SetFlag" in code
assert "WaitFlag" in code
```

#### 6.2 端到端功能测试

与 `testing/python/language/` 中现有测试模式一致，验证算子端到端计算正确：

```python
def test_your_pass_e2e():
    """端到端验证 Pass 后算子计算正确"""
    @T.prim_func
    def kernel(A: T.Tensor((M, N), "float16"), B: T.Tensor((M, N), "float16")):
        # ... kernel 实现 ...
        pass

    ref_output = compute_reference(input_data)  # torch 参考实现

    jit_kernel = tilelang.JITKernel(kernel, out_idx=[1])
    output = jit_kernel(input_data)

    torch.testing.assert_close(output, ref_output, rtol=1e-2, atol=1e-2)
```

#### 6.3 测试文件命名与组织

- IR 测试：`testing/python/test_pass_<pass_name>.py`
- 功能测试：可复用 `testing/python/language/` 中的已有测试，或新建 `testing/python/test_pass_<pass_name>_e2e.py`
- 运行：`cd examples && bash bench_test.sh`（会自动发现 `testing/python/` 下的测试）

#### 6.4 测试设计要点

| 测试维度 | 要求 |
|----------|------|
| 正向变换 | 验证 Pass 对目标 IR 模式执行了预期的变换 |
| 无关 IR | 验证 Pass 不修改不匹配条件的 IR（正交性） |
| 边界条件 | 空循环体、单元素 buffer、嵌套结构等 |
| 前置依赖 | 如果 Pass 依赖上游 Pass 的输出，测试中需先运行前置 Pass |
| 回归保护 | 修改 Pass 后确认已有 IR 测试仍通过 |

## C++ Pass 常用基类

| 基类 | 用途 | 典型场景 |
|------|------|----------|
| `StmtExprVisitor` | 只读遍历 IR，收集信息 | Buffer shape 收集、分析 |
| `StmtExprMutator` | 修改 IR 节点 | 简单的节点替换 |
| `arith::IRMutatorWithAnalyzer` | 修改 IR + 算术约束分析 | 需要 bound 推导的变换 |
| `IndexDataTypeRewriter` | 修改索引数据类型 | int32→int64 提升 |

## IR 节点速查

在编写 Pass 时，你通常需要匹配和变换以下 IR 节点：

| 节点类型 | 用途 | 常见 Visitor 方法 |
|----------|------|-------------------|
| `ForNode` | 循环 | `VisitStmt_(const ForNode*)` |
| `BufferStoreNode` | buffer 写入 | `VisitStmt_(const BufferStoreNode*)` |
| `BufferLoadNode` | buffer 读取 | `VisitExpr_(const BufferLoadNode*)` |
| `AllocateNode` | 内存分配 | `VisitStmt_(const AllocateNode*)` |
| `BlockNode` | Block 作用域 | `VisitStmt_(const BlockNode*)` |
| `AttrStmtNode` | 属性标注 | `VisitStmt_(const AttrStmtNode*)` |
| `LetStmtNode` | 变量绑定 | `VisitStmt_(const LetStmtNode*)` |
| `IfThenElseNode` | 条件分支 | `VisitStmt_(const IfThenElseNode*)` |
| `CallNode` | 函数/intrinsic 调用 | `VisitExpr_(const CallNode*)` |
| `SeqStmtNode` | 语句序列 | `VisitStmt_(const SeqStmtNode*)` |

## 常用 IR 操作

```cpp
// 创建新的 For 循环
For(var, min, extent, ForKind::kSerial, body)

// 创建 BufferStore
BufferStore(buffer, value, indices)

// 创建 Call intrinsic
Call(return_type, Op::Get("tl.your_op"), args)

// Buffer 的 COW 修改
PrimFuncNode *fptr = f.CopyOnWrite();
fptr->body = new_body;

// 添加函数属性
auto fn_attr = fptr->attrs.CopyOnWrite();
fn_attr->dict.Set("your_attr", value);
```

## Pass 间数据传递

Pass 之间通过 IR 属性（Attribute）传递信息：

```cpp
// Pass A：写入属性
auto fn_attr = fptr->attrs.CopyOnWrite();
fn_attr->dict.Set("my_data", my_array);

// Pass B：读取属性
if (f->attrs.defined() && f->attrs->dict.count("my_data")) {
    auto data = Downcast<Array<...>>(f->attrs->dict["my_data"]);
}
```

## 调试 Pass

### 方法 1：打印 IR
在 `tilelang/engine/lower.py` 中插入：
```python
print("=== Before YourPass ===")
print(mod)
mod = tilelang.transform.YourPassName()(mod)
print("=== After YourPass ===")
print(mod)
```

### 方法 2：C++ 日志
```cpp
#include <tvm/runtime/logging.h>
LOG(INFO) << "YourPass: processing buffer " << buffer->name;
```

### 方法 3：GDB 调试
```bash
gdb --args python your_test.py
(gdb) break YourPassRewriter::VisitStmt_
(gdb) run
```

## 参考文档

- [编译流水线详解](references/pass-pipeline.md)
- [C++ Pass 模式与模板](references/pass-cpp-patterns.md)
- [架构参考](../tilelang-custom-skill/architecture.md)
- [错误诊断](../tilelang-custom-skill/tilelang-error-fixer/SKILL.md)
