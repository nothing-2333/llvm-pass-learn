## 前言
死代码删除

## 正文
`llvm/lib/Transforms/Scalar/DCE.cpp`:
```cpp
PreservedAnalyses DCEPass::run(Function &F, FunctionAnalysisManager &AM) {
  if (!eliminateDeadCode(F, &AM.getResult<TargetLibraryAnalysis>(F)))
    return PreservedAnalyses::all();

  PreservedAnalyses PA;
  PA.preserveSet<CFGAnalyses>();
  return PA;
}
```
首先调用 `AM.getResult<TargetLibraryAnalysis>(F)` 获取 TargetLibraryAnalysis 的分析结果, 提供目标平台的库函数信息, 如 printf 是标准库函数, 有副作用不能删等. 然后调用 `eliminateDeadCode` 执行死代码消除, 函数无任何修改则返回 false, 对应 `return PreservedAnalyses::all();`; 有修改则返回 true, 对应 `PA.preserveSet<CFGAnalyses>();`, 保留所有 CFG 相关的分析结果, DCE 仅删除指令, 未修改基本块/分支结构, 因此 CFG 相关分析(如 DominatorTree, LoopInfo)仍有效.

再来看看 `eliminateDeadCode`:
```cpp
static bool eliminateDeadCode(Function &F, TargetLibraryInfo *TLI) {
  bool MadeChange = false;
  SmallSetVector<Instruction *, 16> WorkList;
  // Iterate over the original function, only adding insts to the worklist
  // if they actually need to be revisited. This avoids having to pre-init
  // the worklist with the entire function's worth of instructions.
  for (Instruction &I : llvm::make_early_inc_range(instructions(F))) {
    // We're visiting this instruction now, so make sure it's not in the
    // worklist from an earlier visit.
    if (!WorkList.count(&I))
      MadeChange |= DCEInstruction(&I, WorkList, TLI);
  }

  while (!WorkList.empty()) {
    Instruction *I = WorkList.pop_back_val();
    MadeChange |= DCEInstruction(I, WorkList, TLI);
  }
  return MadeChange;
}
```
`MadeChange` 作为返回值, 如果有死代码被删除就设为 true, `llvm::make_early_inc_range` 确保了即使当前指令被删除, 迭代器仍能正常遍历下一个, `instructions(F)` 将函数中的指令扁平化, 不用再去嵌套遍历基本块, 然后调用 `DCEInstruction(&I, WorkList, TLI)` 删除死代码. 而这个算法的核心点是有两个循环, 第一个循环执行时 `WorkList` 刚刚被创建, 在执行 `DCEInstruction(&I, WorkList, TLI);` 的过程中可能会填充 `WorkList`, 因为删除了死代码指令 A, 可能使得指令 B 变为死代码, 此时指令 B 会被加入 `WorkList`, 然后下边通过 `while (!WorkList.empty())` 这个循环去处理干净.