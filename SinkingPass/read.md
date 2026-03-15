## 前言
代码下沉, 将指令移动到其结果被实际使用的后继基本块中

## 正文
从 run 开始:
```cpp
PreservedAnalyses SinkingPass::run(Function &F, FunctionAnalysisManager &AM) {
  auto &DT = AM.getResult<DominatorTreeAnalysis>(F);  // 支配树
  auto &LI = AM.getResult<LoopAnalysis>(F);           // 循环信息
  auto &AA = AM.getResult<AAManager>(F);              // 别名分析

  if (!iterativelySinkInstructions(F, DT, LI, AA))
    return PreservedAnalyses::all();

  PreservedAnalyses PA;
  PA.preserveSet<CFGAnalyses>();
  return PA;
}
```
跟入 `iterativelySinkInstructions`:
```cpp
static bool iterativelySinkInstructions(Function &F, DominatorTree &DT,
                                        LoopInfo &LI, AAResults &AA) {
  bool MadeChange, EverMadeChange = false;

  do {
    MadeChange = false;
    LLVM_DEBUG(dbgs() << "Sinking iteration " << NumSinkIter << "\n");
    // 处理所有基本快
    for (BasicBlock &I : F)
      MadeChange |= ProcessBlock(I, DT, LI, AA);
    EverMadeChange |= MadeChange;
    NumSinkIter++;
  } while (MadeChange);

  return EverMadeChange;
}
```
调用 `ProcessBlock` 处理所有基本块, 如果都做处理则退出. 跟入 `ProcessBlock`:
```cpp
static bool ProcessBlock(BasicBlock &BB, DominatorTree &DT, LoopInfo &LI,
                         AAResults &AA) {
  // 跳过不可达的基本块
  if (!DT.isReachableFromEntry(&BB)) return false;

  bool MadeChange = false;

  // 从基本块的末尾向前遍历: 如果先处理前面的指令, 其下沉操作会改变块内指令顺序, 导致迭代器失效, 后向前遍历, 即使当前指令被下沉, 前面的迭代器仍有效
  BasicBlock::iterator I = BB.end();
  --I;        // 最后一条实际指令
  bool ProcessedBegin = false;
  SmallPtrSet<Instruction *, 8> Stores;   // 记录块内遇到的存储指令
  do {
    Instruction *Inst = &*I; // 获取当前待检查/下沉的指令

    // 如果不是第一条指令, 将迭代器前移, 确保后续迭代正确
    ProcessedBegin = I == BB.begin();
    if (!ProcessedBegin)
      --I;

    // 跳过调试指令/伪指令
    if (Inst->isDebugOrPseudoInst())
      continue;

    // 尝试下沉当前指令
    if (SinkInstruction(Inst, Stores, DT, LI, AA)) {
      ++NumSunk;
      MadeChange = true;
    }

    // 当遍历到块的第一条指令时, 结束循环
  } while (!ProcessedBegin);

  return MadeChange;
}
```
从后往前遍历尝试下沉指令.

看看如何下沉指令的, 跟入 `SinkInstruction`:
```cpp
static bool SinkInstruction(Instruction *Inst,
                            SmallPtrSetImpl<Instruction *> &Stores,
                            DominatorTree &DT, LoopInfo &LI, AAResults &AA) {

  // 判断指令是否可移动
  if (!isSafeToMove(Inst, AA, Stores))
    return false;

  // 待确定的下沉目标块
  BasicBlock *SuccToSinkTo = nullptr;

  BasicBlock *BB = Inst->getParent();
  // 遍历指令的所有使用者, 找到所有指令使用者的"最近公共支配块"作为初始候选块
  for (Use &U : Inst->uses()) {
    Instruction *UseInst = cast<Instruction>(U.getUser());
    BasicBlock *UseBlock = UseInst->getParent();

    // 特殊处理 PHI 节点: PHI 指令使用了该指令, 那么追溯到对应的"前驱块"而非 PHI 所在块, 因为 PHI 指令必须是第一条指令
    if (PHINode *PN = dyn_cast<PHINode>(UseInst)) {
      unsigned Num = PHINode::getIncomingValueNumForOperand(U.getOperandNo());
      UseBlock = PN->getIncomingBlock(Num);
    }
    // 跳过不可达的使用者
    if (!DT.isReachableFromEntry(UseBlock))
      continue;

    // 更新 SuccToSinkTo
    // 首次迭代: 候选块 = 第一个有效使用块
    // 后续迭代: 当前候选块与新使用块的最近公共支配块
    if (SuccToSinkTo)
      SuccToSinkTo = DT.findNearestCommonDominator(SuccToSinkTo, UseBlock);
    else
      SuccToSinkTo = UseBlock;
  }

  if (SuccToSinkTo) {
    // 调用 IsAcceptableTarget 判断是否合法, 不合法向上找合法祖先块
    while (SuccToSinkTo != BB &&
           !IsAcceptableTarget(Inst, SuccToSinkTo, DT, LI))
      // 向上遍历支配树, SuccToSinkTo = SuccToSinkTo 直接支配块(IDom)
      SuccToSinkTo = DT.getNode(SuccToSinkTo)->getIDom()->getBlock();
    if (SuccToSinkTo == BB)
      SuccToSinkTo = nullptr;
  }

  if (!SuccToSinkTo)
    return false;

  LLVM_DEBUG(dbgs() << "Sink" << *Inst << " (";
             Inst->getParent()->printAsOperand(dbgs(), false); dbgs() << " -> ";
             SuccToSinkTo->printAsOperand(dbgs(), false); dbgs() << ")\n");

  assert(DT.dominates(BB, SuccToSinkTo) &&
         "SuccToSinkTo must be dominated by current Inst location!");

  // 将指令移动到目标块的"第一个插入点"(跳过所有 PHI 节点, 跳过异常处理垫指令)
  Inst->moveBefore(SuccToSinkTo->getFirstInsertionPt());
  return true;
}
```
- 需找当前指令的使用块的公共支配块
- 对 PHI 节点进行额外处理, 选用对应前驱块
- 处理最近公共支配块在当前块的父循环中的情况, 向上找合法祖先块
- 将指令移动到目标块的"第一个插入点"

在来看看 `IsAcceptableTarget` 判断是否合法逻辑:
```cpp
static bool IsAcceptableTarget(Instruction *Inst, BasicBlock *SuccToSinkTo,
                               DominatorTree &DT, LoopInfo &LI) {
  assert(Inst && "Instruction to be sunk is null");
  assert(SuccToSinkTo && "Candidate sink target is null");

  // 禁止将指令下沉到异常处理垫(EH-pad)块
  if (SuccToSinkTo->isEHPad())
    return false;

  if (SuccToSinkTo->getUniquePredecessor() != Inst->getParent()) {
    // 禁止将读内存指令下沉, 其他前驱路径中可能有 store 指令修改同一内存地址, 导致 load 读取到错误值
    if (Inst->mayReadFromMemory() &&
        !Inst->hasMetadata(LLVMContext::MD_invariant_load))
      return false;

    // 禁止将指令下沉到"不同的循环"中
    Loop *succ = LI.getLoopFor(SuccToSinkTo);
    Loop *cur = LI.getLoopFor(Inst->getParent());
    if (succ != nullptr && succ != cur)
      return false;
  }

  return true;
}
```
- EH-pad 块直接不合法
- `getUniquePredecessor` 用于判断一个基本块是否只有唯一的前驱基本块, 非 nullptr → 当前块只有一个前驱, nullptr → 当前块前驱数量 ≠ 1, 如果 `SuccToSinkTo` 不是唯一前驱, 或者唯一前驱不是当前块就进入检查
- 总结就是当 SuccToSinkTo 的唯一前驱是当前块时, 意味着控制流是"单一路径". 不存在多路径执行的风险, 因此无需再做 Load 指令, 循环归属的额外检查

实验一下:
```ir
@A = external global i32
@B = external global i32

define i32 @foo(i1 %z) {
  %l = load i32, ptr @A
  store i32 0, ptr @B
  br i1 %z, label %true, label %false
true:
  ret i32 %l
false:
  ret i32 0
}

define i32 @diamond(i32 %a, i32 %b, i32 %c) {
  %1 = mul nsw i32 %c, %b
  %2 = icmp sgt i32 %a, 0
  br i1 %2, label %B0, label %B1

B0:                                       ; preds = %0
  br label %X

B1:                                      ; preds = %0
  br label %X

X:                                     ; preds = %5, %3
  %.01 = phi i32 [ %c, %B0 ], [ %a, %B1 ]
  %R = sub i32 %1, %.01
  ret i32 %R
}
```
```bash
./build/bin/opt -passes=sink -S /tmp/a.ll
```
```ir
; ModuleID = '/tmp/a.ll'
source_filename = "/tmp/a.ll"

@A = external global i32
@B = external global i32

define i32 @foo(i1 %z) {
  store i32 0, ptr @B, align 4
  br i1 %z, label %true, label %false

true:                                             ; preds = %0
  %l = load i32, ptr @A, align 4
  ret i32 %l

false:                                            ; preds = %0
  ret i32 0
}

define i32 @diamond(i32 %a, i32 %b, i32 %c) {
  %1 = icmp sgt i32 %a, 0
  br i1 %1, label %B0, label %B1

B0:                                               ; preds = %0
  br label %X

B1:                                               ; preds = %0
  br label %X

X:                                                ; preds = %B1, %B0
  %.01 = phi i32 [ %c, %B0 ], [ %a, %B1 ]
  %2 = mul nsw i32 %c, %b
  %R = sub i32 %2, %.01
  ret i32 %R
}
```