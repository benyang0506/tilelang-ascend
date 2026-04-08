# 编译流水线详解

TileLang-Ascend 编译流程分为两个主要阶段，所有 Pass 在 `tilelang/engine/phase.py` 中编排。

## 总体流程

```
Python DSL (@tilelang.jit)
    ↓
Phase 1: LowerAndLegalize (降级与合法化)
    ↓
Phase 2: OptimizeForTarget (目标优化)
    ↓
代码生成 (codegen_ascend_pto.cc)
    ↓
CANN 工具链 → NPU 执行
```

## Phase 1: LowerAndLegalize

将前端 DSL IR 降级为中间表示，主要完成 scope 推断、layout 推导、向量化 lower 等。

```
AscendInferBufferScope    ← 推断 buffer 所属内存层级（L1/UB/L0A 等）
    ↓
BufferShapeCollector      ← 收集初始 buffer shape 信息
    ↓
BindTarget                ← 绑定硬件 target 信息
    ↓
HostProcesser             ← host 端代码处理
    ↓
Simplify                  ← 表达式化简
    ↓
AscendLowerParallelToVector ← T.Parallel 循环 → Vector 指令
    ↓
LayoutInference           ← 内存 layout 推导（zN 等）
    ↓
CollectBufferShapes       ← 重新收集 buffer shape（pipeline 可能改变维度）
    ↓
LowerTileOp              ← 降级 tile 级操作到底层指令
    ↓
LegalizeVectorizedLoop    ← 向量化循环合法化
    ↓
LegalizeSafeMemoryAccess  ← 内存访问安全性检查
    ↓
Simplify                  ← 再次化简
```

## Phase 2: OptimizeForTarget

针对 Ascend 硬件进行优化和最终 lower。

```
Flatten2DBuffer           ← ND buffer → 2D（Ascend PTO 要求）
    ↓
FlattenBuffer             ← 多维索引 → 线性偏移
    ↓
AscendStorageRewrite      ← 存储访问优化
    ↓
VectorizeLoop             ← 循环向量化
    ↓
ConfigIndexBitwidth       ← 索引位宽配置
    ↓
InjectSoftwarePipeline    ← 软件流水线注入
    ↓
AscendMemoryPlanning      ← 内存地址规划（buffer reuse）
    ↓
CrossCorePipeline         ← 核间流水线处理
    ↓
CombineCV                 ← Cube/Vector 作用域合并
    ↓
AscendSyncInsert          ← 同步指令自动插入
    ↓
AscendLowerOpaqueBlock    ← OpaqueBlock 消除
    ↓
Simplify / ThreadSync等   ← 最终清理
```

## 新 Pass 的插入位置参考

| 你的 Pass 功能 | 建议插入位置 |
|----------------|-------------|
| 前端 IR 规范化 | Phase 1 开头，`AscendInferBufferScope` 前后 |
| Buffer scope/layout 相关 | Phase 1 `LayoutInference` 前后 |
| 向量化/并行化相关 | Phase 1 `AscendLowerParallelToVector` 前后 |
| Tile 操作降级 | Phase 1 `LowerTileOp` 前后 |
| Buffer 形状变换 | Phase 2 `Flatten2DBuffer` 前后 |
| 存储优化 | Phase 2 `AscendStorageRewrite` 前后 |
| 流水线优化 | Phase 2 `InjectSoftwarePipeline` 前后 |
| 内存地址规划 | Phase 2 `AscendMemoryPlanning` 前后 |
| 同步插入 | Phase 2 `AscendSyncInsert` 前后 |
| 核间 Cube/Vector 融合 | Phase 2 `CombineCV` 前后 |

## 关键 PassConfig 开关

| 配置键 | 功能 | 默认值 |
|--------|------|--------|
| `tl.ascend_auto_sync` | 自动插入核内同步 | `false` |
| `tl.ascend_memory_planning` | 自动内存规划 | `false` |
| `tl.ascend_auto_cv_combine` | 自动 Cube/Vector 合并 | `false` |
| `tl.ascend_auto_cross_core_sync` | 自动核间同步 | `false` |
| `tl.Simplify` | 额外化简 | `false` |
| `tir.disable_vectorize` | 禁用向量化 | `false` |
| `tir.disable_storage_rewrite` | 禁用存储重写 | `false` |

## Pass 间属性传递约定

| 属性名 | 写入 Pass | 读取 Pass | 内容 |
|--------|-----------|-----------|------|
| `initial_buffer_shapes` | `BufferShapeCollector` | `Flatten2DBuffer` | 初始 buffer shape |
| `logic_buffer_shapes` | `Flatten2DBuffer` | 后续 Pass | 2D 对齐后的 shape |
| `buffer_shapess` | `PTOSaveBufferShape` | CodeGen | PTO 用 buffer shape |
| `use_swizzle` | `FrontendLegalize` | 后续优化 | 是否使用 swizzle |
