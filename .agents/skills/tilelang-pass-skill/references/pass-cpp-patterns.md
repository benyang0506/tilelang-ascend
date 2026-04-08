# C++ Pass 模式与模板

本文档提供 TileLang-Ascend 中 C++ Pass 的常见模式和可直接使用的代码模板。

## 1. 只读分析 Pass（Visitor 模式）

用于收集信息、不修改 IR。

```cpp
#include <tvm/tir/stmt_functor.h>
#include <tvm/tir/transform.h>

namespace tvm {
namespace tl {

using namespace tir;

// ========== Visitor 类 ==========
class BufferInfoCollector : public StmtExprVisitor {
 public:
  // 收集结果
  Map<Var, Array<PrimExpr>> buffer_shapes;

  void VisitStmt_(const AllocateNode *op) final {
    // 记录分配信息
    buffer_shapes.Set(op->buffer_var,
                      Array<PrimExpr>(op->extents.begin(), op->extents.end()));
    // 继续递归遍历子节点
    StmtExprVisitor::VisitStmt_(op);
  }

  void VisitStmt_(const BufferStoreNode *op) final {
    // 分析写入模式
    StmtExprVisitor::VisitStmt_(op);
  }

  void VisitExpr_(const BufferLoadNode *op) final {
    // 分析读取模式
    StmtExprVisitor::VisitExpr_(op);
  }
};

// ========== Pass 函数 ==========
using namespace tir::transform;

Pass CollectBufferInfo() {
  auto pass_func = [=](PrimFunc f, IRModule m, PassContext ctx) {
    BufferInfoCollector collector;
    collector(f->body);  // 遍历函数体

    // 将结果写入函数属性
    PrimFuncNode *fptr = f.CopyOnWrite();
    auto fn_attr = fptr->attrs.CopyOnWrite();
    fn_attr->dict.Set("buffer_info", collector.buffer_shapes);
    return f;
  };
  return CreatePrimFuncPass(pass_func, 0, "tl.CollectBufferInfo", {});
}

TVM_REGISTER_GLOBAL("tl.transform.CollectBufferInfo")
    .set_body_typed(CollectBufferInfo);

}  // namespace tl
}  // namespace tvm
```

## 2. IR 变换 Pass（Mutator 模式）

用于修改 IR 节点，最常用的模式。

```cpp
#include <tvm/arith/analyzer.h>
#include <tvm/tir/analysis.h>
#include <tvm/tir/stmt_functor.h>
#include <tvm/tir/transform.h>

#include "common/attr.h"

namespace tvm {
namespace tl {

using namespace tir;

// ========== Mutator 类 ==========
class YourPassRewriter : public arith::IRMutatorWithAnalyzer {
 public:
  // 静态工厂方法（推荐模式）
  static PrimFunc Substitute(PrimFunc f) {
    arith::Analyzer analyzer;
    YourPassRewriter rewriter(&analyzer);
    PrimFuncNode *fptr = f.CopyOnWrite();
    fptr->body = rewriter.VisitStmt(f->body);
    return f;
  }

 private:
  using arith::IRMutatorWithAnalyzer::IRMutatorWithAnalyzer;

  // ---- 重写语句节点 ----
  Stmt VisitStmt_(const ForNode *op) final {
    // 先递归处理子节点
    Stmt body = VisitStmt(op->body);

    // 你的变换逻辑
    if (ShouldTransform(op)) {
      return TransformFor(op, body);
    }

    // 如果 body 没变，直接返回原节点（避免无用拷贝）
    if (body.same_as(op->body)) {
      return GetRef<Stmt>(op);
    }
    // body 变了，创建新 For
    return For(op->loop_var, op->min, op->extent, op->kind, body,
               op->thread_binding, op->annotations);
  }

  Stmt VisitStmt_(const BlockNode *op) final {
    // Block 节点通常需要处理 alloc_buffers
    auto n = arith::IRMutatorWithAnalyzer::VisitStmt_(op);
    Block block = Downcast<Block>(n);
    // 修改 block 的 alloc_buffers 等
    return block;
  }

  // ---- 重写表达式节点 ----
  PrimExpr VisitExpr_(const BufferLoadNode *op) final {
    // 修改 buffer 读取
    return arith::IRMutatorWithAnalyzer::VisitExpr_(op);
  }

  // ---- 辅助方法 ----
  bool ShouldTransform(const ForNode *op) {
    return op->kind == ForKind::kParallel;
  }

  Stmt TransformFor(const ForNode *op, Stmt body) {
    // 实际变换逻辑
    return body;
  }
};

// ========== Pass 创建 + 注册 ==========
using namespace tir::transform;

Pass YourPassName() {
  auto pass_func = [=](PrimFunc f, IRModule m, PassContext ctx) {
    return YourPassRewriter::Substitute(std::move(f));
  };
  return CreatePrimFuncPass(pass_func, 0, "tl.YourPassName", {});
}

TVM_REGISTER_GLOBAL("tl.transform.YourPassName")
    .set_body_typed(YourPassName);

}  // namespace tl
}  // namespace tvm
```

## 3. 带参数的 Pass

当 Pass 需要接收外部参数（如 target、platform 等）时。

```cpp
Pass YourPassWithArgs(Target target, std::string platform) {
  auto pass_func = [=](PrimFunc f, IRModule m, PassContext ctx) {
    // 从 PassContext 读取配置
    bool enabled = ctx->GetConfig<Bool>("tl.your_config", Bool(false)).value();
    if (!enabled) return f;

    // 使用参数
    return YourRewriter::Substitute(std::move(f), target, platform);
  };
  return CreatePrimFuncPass(pass_func, 0, "tl.YourPassWithArgs", {});
}

// FFI 注册（带参数）
TVM_REGISTER_GLOBAL("tl.transform.YourPassWithArgs")
    .set_body_typed(YourPassWithArgs);
```

Python 封装：
```python
def YourPassWithArgs(target, platform):
    return _ffi_api.YourPassWithArgs(target, platform)
```

## 4. 两阶段 Pass（收集 + 变换）

先收集信息再变换，常见于需要全局分析的场景。

```cpp
class InfoCollector : public StmtExprVisitor {
 public:
  std::unordered_map<const VarNode*, BufferInfo> info_map;

  void VisitStmt_(const AllocateNode *op) final {
    // 收集信息
    info_map[op->buffer_var.get()] = AnalyzeBuffer(op);
    StmtExprVisitor::VisitStmt_(op);
  }
};

class Transformer : public arith::IRMutatorWithAnalyzer {
 public:
  static PrimFunc Substitute(PrimFunc f) {
    // 阶段 1：收集
    InfoCollector collector;
    collector(f->body);

    // 阶段 2：变换
    arith::Analyzer analyzer;
    Transformer transformer(&analyzer);
    transformer.info_ = std::move(collector.info_map);

    PrimFuncNode *fptr = f.CopyOnWrite();
    fptr->body = transformer.VisitStmt(f->body);
    return f;
  }

 private:
  using arith::IRMutatorWithAnalyzer::IRMutatorWithAnalyzer;
  std::unordered_map<const VarNode*, BufferInfo> info_;

  Stmt VisitStmt_(const AllocateNode *op) final {
    if (auto it = info_.find(op->buffer_var.get()); it != info_.end()) {
      // 利用收集到的信息进行变换
    }
    return arith::IRMutatorWithAnalyzer::VisitStmt_(op);
  }
};
```

## 5. Buffer 重映射模式

当需要替换 buffer 引用时的标准模式。

```cpp
class BufferRemapper : public arith::IRMutatorWithAnalyzer {
 public:
  static Stmt Substitute(Stmt stmt, Map<Buffer, Buffer> remap) {
    arith::Analyzer analyzer;
    BufferRemapper remapper(&analyzer);
    remapper.buffer_remap_ = std::move(remap);
    return remapper.VisitStmt(stmt);
  }

 private:
  using arith::IRMutatorWithAnalyzer::IRMutatorWithAnalyzer;
  Map<Buffer, Buffer> buffer_remap_;

  Buffer RemapBuffer(Buffer buf) {
    auto it = buffer_remap_.find(buf);
    return it != buffer_remap_.end() ? (*it).second : buf;
  }

  Stmt VisitStmt_(const BufferStoreNode *op) final {
    auto new_buf = RemapBuffer(op->buffer);
    auto stmt = arith::IRMutatorWithAnalyzer::VisitStmt_(op);
    if (!new_buf.same_as(op->buffer)) {
      auto n = stmt.as<BufferStoreNode>();
      auto new_node = runtime::make_object<BufferStoreNode>(*n);
      new_node->buffer = new_buf;
      return Stmt(new_node);
    }
    return stmt;
  }

  PrimExpr VisitExpr_(const BufferLoadNode *op) final {
    auto new_buf = RemapBuffer(op->buffer);
    auto expr = arith::IRMutatorWithAnalyzer::VisitExpr_(op);
    if (!new_buf.same_as(op->buffer)) {
      auto n = expr.as<BufferLoadNode>();
      auto new_node = runtime::make_object<BufferLoadNode>(*n);
      new_node->buffer = new_buf;
      return PrimExpr(new_node);
    }
    return expr;
  }
};
```

## 6. 常用头文件

```cpp
// 必需
#include <tvm/tir/stmt_functor.h>       // Visitor/Mutator 基类
#include <tvm/tir/transform.h>          // Pass 注册
#include <tvm/tir/analysis.h>           // IR 分析工具

// 常用
#include <tvm/arith/analyzer.h>         // 算术约束分析
#include <tvm/tir/op.h>                 // IR 操作构造
#include <tvm/tir/builtin.h>           // 内建函数
#include <tvm/runtime/logging.h>        // LOG(INFO) 日志

// TileLang 特有
#include "common/attr.h"                // 属性常量
#include "common/collector.h"           // 收集器工具
#include "common/operation_config.h"    // Ascend 操作配置
```

## 7. 常见陷阱

### 7.1 忘记递归子节点
```cpp
// 错误：没有递归
Stmt VisitStmt_(const ForNode *op) final {
  return GetRef<Stmt>(op);  // 子节点未被访问！
}

// 正确：先递归再处理
Stmt VisitStmt_(const ForNode *op) final {
  auto n = arith::IRMutatorWithAnalyzer::VisitStmt_(op);  // 递归
  // 在 n 上做你的变换
  return n;
}
```

### 7.2 CopyOnWrite 遗漏
```cpp
// 错误：直接修改只读节点
fptr->body = new_body;  // fptr 是原始指针

// 正确：先 COW
PrimFuncNode *fptr = f.CopyOnWrite();
fptr->body = new_body;
```

### 7.3 same_as 检查
```cpp
// 好习惯：如果没有变化就返回原节点，避免无谓的内存分配
Stmt body = VisitStmt(op->body);
if (body.same_as(op->body)) {
  return GetRef<Stmt>(op);
}
```
