## 豆包
| 路径 / 文件名 | 作用 |
| --- | --- |
| `Analysis/LoopInfo.cpp` | 循环识别 |
| `Analysis/MemorySSA.cpp` | 内存 SSA |
| `Analysis/AliasAnalysis.cpp` | 别名分析 |
| `Analysis/CallGraph.cpp` | 调用图 |
| `Scalar/GVN.cpp` / `NewGVN.cpp` | 全局值编号 |
| `Scalar/JumpThreading.cpp` | 跳转优化 |
| `Scalar/LoopUnrollPass.cpp` | 循环展开 |
| `Scalar/FlattenCFGPass.cpp` | CFG 平坦化 |
| `Scalar/SimplifyCFGPass.cpp` | CFG 简化 |
| `Scalar/LICM.cpp` | 循环外提 |
| `InstCombine/InstructionCombining.cpp` | 指令化简合并 |
| `IPO/Inliner.cpp` | 函数内联 |
| `Utils/Mem2Reg.cpp` | 内存转寄存器 |
| `Utils/LowerSwitch.cpp` | Switch 展开 |
| `Instrumentation/SanitizerCoverage.cpp` | 插桩 |
| `llvm/lib/IR/Dominators.cpp` | 构建支配树 |

## 我
| 学习顺序 | 路径 |
| --- | --- |
| 1 | llvm/include/llvm/Transforms/Utils/HelloWorld.h |
| 2 | llvm/include/llvm/Transforms/Scalar/DCE.h |
| 3 | llvm/lib/Transforms/Scalar/ADCE.cpp |
| 4 | llvm/lib/Transforms/Scalar/InstSimplifyPass.cpp |
| 5 | llvm/lib/Transforms/Scalar/Sink.cpp |
| 6 | llvm/lib/Analysis/MemorySSA.cpp |
| 7 | llvm/lib/Transforms/Scalar/LICM.cpp |