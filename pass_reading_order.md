一些相关路径:
- llvm/lib/Transforms
- llvm/lib/Analysis


## 豆包
| **进阶级（核心优化）** | Scalar | DeadStoreElimination.cpp | Dead Store Elimination | ★★★ | Load/Store分析、内存操作优化 | 理解内存相关IR的处理逻辑 |
| | Scalar | LICM.cpp | Loop Invariant Code Motion | ★★★ | 循环分析、LoopPass使用、循环不变量外提 | 入门循环优化的核心Pass |
| | Scalar | SimplifyCFGPass.cpp | Simplify CFG | ★★★ | 基本块合并、分支简化、控制流优化 | 理解CFG（控制流图）的修改 |
| | Scalar | GVN.cpp | Global Value Numbering | ★★★☆ | 冗余指令消除、值编号分析 | 学习"值等价性"的分析思路 |
| | InstCombine | InstructionCombining.cpp | Instruction Combine | ★★★☆ | 复杂指令模式匹配、表达式化简 | LLVM最核心的优化Pass，建议先看InstCombineAddSub.cpp等单文件 |
| | Scalar | IndVarSimplify.cpp | Induction Variable Simplify | ★★★☆ | 循环变量分析、循环变量简化 | 深入理解循环内部的变量优化 |
| **高阶（模块/高级优化）** | IPO | ConstantMerge.cpp | Constant Merge | ★★★ | ModulePass使用、全局常量合并 | 入门模块级优化（跨函数） |
| | IPO | GlobalDCE.cpp | Global DCE | ★★★ | 全局函数/变量无用分析、模块级删除 | 理解ModulePass与FunctionPass的区别 |
| | IPO | AlwaysInliner.cpp | Always Inliner | ★★★☆ | 函数内联、CallSite处理、IR克隆 | 学习函数内联的基本逻辑 |
| | IPO | ArgumentPromotion.cpp | Argument Promotion | ★★★☆ | 函数参数优化、参数类型调整 | 理解跨函数的参数优化思路 |
| | Scalar | LoopUnrollPass.cpp | Loop Unroll | ★★★★ | 循环展开、循环次数分析 | 深入学习循环优化的经典Pass |
| | Vectorize | SLPVectorizer.cpp | SLP Vectorizer | ★★★★ | 向量化、指令打包、SIMD优化 | 学习现代CPU的向量化优化逻辑 |
| | Vectorize | LoopVectorize.cpp | Loop Vectorizer | ★★★★☆ | 循环向量化、向量化合法性分析 | 更复杂的循环向量化，适合进阶学习 |
| **工具类/辅助Pass（拓展）** | Instrumentation | BoundsChecking.cpp | Bounds Checking | ★★★ | IR插桩、边界检查注入 | 学习"插桩类Pass"的编写思路 |
| | Utils | Mem2Reg.cpp | Memory to Register | ★★★☆ | 栈变量提升为寄存器、SSA构造 | 理解SSA（静态单赋值）的核心概念 |
| | Scalar | SROA.cpp | Scalar Replacement of Aggregates | ★★★☆ | 聚合类型拆分、内存优化 | 深入理解标量化优化 |
| | Utils | PromoteMemoryToRegister.cpp | Promote Memory to Register | ★★★☆ | 手动实现Mem2Reg的核心逻辑 | 辅助理解Mem2Reg的底层原理 |
| | Instrumentation | GCOVProfiling.cpp | GCOV Profiling | ★★★★ | 插桩、覆盖率分析、数据收集 | 学习"分析类Pass"的编写思路 |


## 我
| 学习顺序 | 路径 |
| 1 | llvm/include/llvm/Transforms/Utils/HelloWorld.h |
| 2 | llvm/include/llvm/Transforms/Scalar/DCE.h |
| 3 | llvm/lib/Transforms/Scalar/ADCE.cpp |
| 4 | llvm/lib/Transforms/Scalar/InstSimplifyPass.cpp |
| 5 | llvm/lib/Transforms/Scalar/Sink.cpp |