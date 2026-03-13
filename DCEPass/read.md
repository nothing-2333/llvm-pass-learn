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

看一下 `DCEInstruction`:
```cpp
static bool DCEInstruction(Instruction *I,
                           SmallSetVector<Instruction *, 16> &WorkList,
                           const TargetLibraryInfo *TLI) {
  if (isInstructionTriviallyDead(I, TLI)) {
    if (!DebugCounter::shouldExecute(DCECounter))
      return false;

    salvageDebugInfo(*I);
    salvageKnowledge(I);

    // Null out all of the instruction's operands to see if any operand becomes
    // dead as we go.
    for (unsigned i = 0, e = I->getNumOperands(); i != e; ++i) {
      Value *OpV = I->getOperand(i);
      I->setOperand(i, nullptr);

      if (!OpV->use_empty() || I == OpV)
        continue;

      // If the operand is an instruction that became dead as we nulled out the
      // operand, and if it is 'trivially' dead, delete it in a future loop
      // iteration.
      if (Instruction *OpI = dyn_cast<Instruction>(OpV))
        if (isInstructionTriviallyDead(OpI, TLI))
          WorkList.insert(OpI);
    }

    I->eraseFromParent();
    ++DCEEliminated;
    return true;
  }
  return false;
}
```
这是执行死代码删除的核心函数, 首先调用 `isInstructionTriviallyDead` 检查指令是否是无副作用且无使用的, 如果是的话就是死指令, 调用 `salvageDebugInfo` 与 `salvageKnowledge` 保留指令关联的调试信息和语义知识, 然后调用 for 循环清空指令的所有操作数, 并检查改操作数:
- 操作数 OpV 是一个 Instruction 类型, 而非常量、全局变量等其他 Value
- 执行 I->setOperand(i, nullptr) 后, OpV 已经没有任何使用
- OpV 不是指令 I 本身, 排除自引用的情况
- 满足 isInstructionTriviallyDead(OpV, TLI)

如这样的 ir:
```ir
%a = add i32 %b, %c
%d = mul i32 %a, 10
```

再来看一下 `wouldInstructionBeTriviallyDead`:
```cpp
bool llvm::isInstructionTriviallyDead(Instruction *I,
                                      const TargetLibraryInfo *TLI) {
  if (!I->use_empty())
    return false;
  return wouldInstructionBeTriviallyDead(I, TLI);
}

bool llvm::wouldInstructionBeTriviallyDead(const Instruction *I,
                                           const TargetLibraryInfo *TLI) {
  if (I->isTerminator())
    return false;

  // We don't want the landingpad-like instructions removed by anything this
  // general.
  if (I->isEHPad())
    return false;

  if (const DbgLabelInst *DLI = dyn_cast<DbgLabelInst>(I)) {
    if (DLI->getLabel())
      return false;
    return true;
  }

  if (auto *CB = dyn_cast<CallBase>(I))
    if (isRemovableAlloc(CB, TLI))
      return true;

  if (!I->willReturn()) {
    auto *II = dyn_cast<IntrinsicInst>(I);
    if (!II)
      return false;

    switch (II->getIntrinsicID()) {
    case Intrinsic::experimental_guard: {
      // Guards on true are operationally no-ops.  In the future we can
      // consider more sophisticated tradeoffs for guards considering potential
      // for check widening, but for now we keep things simple.
      auto *Cond = dyn_cast<ConstantInt>(II->getArgOperand(0));
      return Cond && Cond->isOne();
    }
    // TODO: These intrinsics are not safe to remove, because this may remove
    // a well-defined trap.
    case Intrinsic::wasm_trunc_signed:
    case Intrinsic::wasm_trunc_unsigned:
    case Intrinsic::ptrauth_auth:
    case Intrinsic::ptrauth_resign:
    case Intrinsic::ptrauth_resign_load_relative:
      return true;
    default:
      return false;
    }
  }

  if (!I->mayHaveSideEffects())
    return true;

  // Special case intrinsics that "may have side effects" but can be deleted
  // when dead.
  if (const IntrinsicInst *II = dyn_cast<IntrinsicInst>(I)) {
    // Safe to delete llvm.stacksave and launder.invariant.group if dead.
    if (II->getIntrinsicID() == Intrinsic::stacksave ||
        II->getIntrinsicID() == Intrinsic::launder_invariant_group)
      return true;

    // Intrinsics declare sideeffects to prevent them from moving, but they are
    // nops without users.
    if (II->getIntrinsicID() == Intrinsic::allow_runtime_check ||
        II->getIntrinsicID() == Intrinsic::allow_ubsan_check)
      return true;

    if (II->isLifetimeStartOrEnd()) {
      auto *Arg = II->getArgOperand(0);
      if (isa<PoisonValue>(Arg))
        return true;

      // If the only uses of the alloca are lifetime intrinsics, then the
      // intrinsics are dead.
      return llvm::all_of(Arg->uses(), [](Use &Use) {
        return isa<LifetimeIntrinsic>(Use.getUser());
      });
    }

    // Assumptions are dead if their condition is trivially true.
    if (II->getIntrinsicID() == Intrinsic::assume &&
        isAssumeWithEmptyBundle(cast<AssumeInst>(*II))) {
      if (ConstantInt *Cond = dyn_cast<ConstantInt>(II->getArgOperand(0)))
        return !Cond->isZero();

      return false;
    }

    if (auto *FPI = dyn_cast<ConstrainedFPIntrinsic>(I)) {
      std::optional<fp::ExceptionBehavior> ExBehavior =
          FPI->getExceptionBehavior();
      return *ExBehavior != fp::ebStrict;
    }
  }

  if (auto *Call = dyn_cast<CallBase>(I)) {
    if (Value *FreedOp = getFreedOperand(Call, TLI))
      if (Constant *C = dyn_cast<Constant>(FreedOp))
        return C->isNullValue() || isa<UndefValue>(C);
    if (isMathLibCallNoop(Call, TLI))
      return true;
  }

  // Non-volatile atomic loads from constants can be removed.
  if (auto *LI = dyn_cast<LoadInst>(I))
    if (auto *GV = dyn_cast<GlobalVariable>(
            LI->getPointerOperand()->stripPointerCasts()))
      if (!LI->isVolatile() && GV->isConstant())
        return true;

  return false;
}
```
可以看到判断了很多:
```cpp
// 使用为非空不能删除
!I->use_empty()
// 跳转指令不能删除
I->isTerminator()
// 异常处理相关的 Pad 指令不能删除
I->isEHPad()
// 调试标签指令
if (const DbgLabelInst *DLI = dyn_cast<DbgLabelInst>(I))
// 处理不会返回的指令
!I->willReturn()
// 无任何副作用的指令可直接删除
!I->mayHaveSideEffects()
/*
bool Instruction::mayHaveSideEffects() const {
  return mayWriteToMemory() || mayThrow() || !willReturn();
}
*/
// 声明有副作用但实际无用户时可删除的 Intrinsic
if (const IntrinsicInst *II = dyn_cast<IntrinsicInst>(I))
```

其中 Intrinsic 是 LLVM IR 层面的 “系统调用”, 是编译器理解的、有特殊语义的操作, 而非普通的函数调用:
- llvm.assume: 告诉优化器 “某个条件恒为真”, 辅助优化
  ```ir
  call void @llvm.assume(i1 %cond)
  ```
- llvm.lifetime.start/llvm.lifetime.end: 标记内存块的生命周期, 辅助优化器做内存复用:
  ```ir
  %ptr = alloca i32, align 4
  ; 标记 %ptr 从这里开始有效
  call void @llvm.lifetime.start.p0i8(i64 4, ptr %ptr)
  ; 标记 %ptr 到这里结束有效
  call void @llvm.lifetime.end.p0i8(i64 4, ptr %ptr)
  ```
- llvm.stacksave/llvm.stackrestore: 保存 / 恢复栈指针, 用于动态栈分配:
  ```ir 
  ; 保存当前栈指针
  %sp = call ptr @llvm.stacksave()
  ; 恢复栈指针
  call void @llvm.stackrestore(ptr %sp)
  ```
- 等等

鲸书中讲代码删除是“冗余代码删除 / 公共子表达式消除 / 循环不变代码外提”等, 借助数据流分析驱动的优化, 而这里的 DCEPass 是更简单的删除所有 “无人使用 + 无副作用” 的指令, 不需要复杂的数据流方程计算. 无副作用不用说就是上边的一堆判断, 那无人使用是怎么确定的呢, 这是 LLVM IR 的 use-def 链机制实时维护的. 这个并不在这里讲.

使用一下:

你可以在 llvm/test/Transforms/DCE 目录下找到测试的 ir, 而不用自己写, `llvm/test/Transforms/DCE/basic.ll`:
```lr
declare void @llvm.lifetime.start.p0(ptr nocapture) nounwind
declare void @llvm.lifetime.end.p0(ptr nocapture) nounwind

define void @test() {
  %add = add i32 1, 2
  %sub = sub i32 %add, 1
  ret void
}

define i32 @test_lifetime_alloca() {
  %i = alloca i8, align 4
  call void @llvm.lifetime.start.p0(ptr %i)
  call void @llvm.lifetime.end.p0(ptr %i)
  ret i32 0
}
```
扣出 ir 写入 `/tmp/a.ll`, 来跑一跑:
```bash
./build/bin/opt -passes=dce -S /tmp/a.ll
```
```ir 
; ModuleID = '/tmp/a.ll'
source_filename = "/tmp/a.ll"

; Function Attrs: nocallback nofree nosync nounwind willreturn memory(argmem: readwrite)
declare void @llvm.lifetime.start.p0(ptr captures(none)) #0

; Function Attrs: nocallback nofree nosync nounwind willreturn memory(argmem: readwrite)
declare void @llvm.lifetime.end.p0(ptr captures(none)) #0

define void @test() {
  ret void
}

define i32 @test_lifetime_alloca() {
  ret i32 0
}

attributes #0 = { nocallback nofree nosync nounwind willreturn memory(argmem: readwrite) }
```