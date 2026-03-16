## 前言
对程序中内存操作构建 SSA

## 正文
从 `MemorySSAAnalysis::run` 开始
```cpp
MemorySSAAnalysis::Result MemorySSAAnalysis::run(Function &F,
                                                 FunctionAnalysisManager &AM) {
  auto &DT = AM.getResult<DominatorTreeAnalysis>(F);
  auto &AA = AM.getResult<AAManager>(F);
  return MemorySSAAnalysis::Result(std::make_unique<MemorySSA>(F, &AA, &DT));
}
```
找到 `std::make_unique<MemorySSA>(F, &AA, &DT)` 对应的构造函数:
```cpp
// 全函数构建, 为整个函数的所有基本块构建 MemorySSA
MemorySSA::MemorySSA(Function &Func, AliasAnalysis *AA, DominatorTree *DT)
    : DT(DT), F(&Func), LiveOnEntryDef(nullptr), Walker(nullptr),
      SkipWalker(nullptr) {
  // Build MemorySSA using a batch alias analysis. This reuses the internal
  // state that AA collects during an alias()/getModRefInfo() call. This is
  // safe because there are no CFG changes while building MemorySSA and can
  // significantly reduce the time spent by the compiler in AA, because we will
  // make queries about all the instructions in the Function.
  assert(AA && "No alias analysis?");
  BatchAAResults BatchAA(*AA);
  buildMemorySSA(BatchAA, iterator_range(F->begin(), F->end()));
  // Intentionally leave AA to nullptr while building so we don't accidentally
  // use non-batch AliasAnalysis.
  this->AA = AA;
  // Also create the walker here.
  getWalker();
}

// 单循环构建, 仅为循环内的基本块构建 MemorySSA
MemorySSA::MemorySSA(Loop &L, AliasAnalysis *AA, DominatorTree *DT)
    : DT(DT), L(&L), LiveOnEntryDef(nullptr), Walker(nullptr),
      SkipWalker(nullptr) {
  // Build MemorySSA using a batch alias analysis. This reuses the internal
  // state that AA collects during an alias()/getModRefInfo() call. This is
  // safe because there are no CFG changes while building MemorySSA and can
  // significantly reduce the time spent by the compiler in AA, because we will
  // make queries about all the instructions in the Function.
  assert(AA && "No alias analysis?");
  BatchAAResults BatchAA(*AA);
  buildMemorySSA(
      BatchAA, map_range(L.blocks(), [](const BasicBlock *BB) -> BasicBlock & {
        return *const_cast<BasicBlock *>(BB);
      }));
  // Intentionally leave AA to nullptr while building so we don't accidentally
  // use non-batch AliasAnalysis.
  this->AA = AA;
  // Also create the walker here.
  getWalker();
}
```
在看 `buildMemorySSA` 之前明确一个概念:
- 循环预头: 它是循环的唯一前驱, 所有进入循环的路径都必须经过预头. 它的唯一后继是循环的头块(Loop Header)

再来看 `buildMemorySSA`:
```cpp
template <typename IterT>
void MemorySSA::buildMemorySSA(BatchAAResults &BAA, IterT Blocks) {
  // 创建 "live on entry" 内存访问节点 LiveOnEntry, 代表函数/循环入口前已存在的内存状态
  BasicBlock &StartingPoint = *Blocks.begin();
  LiveOnEntryDef.reset(new MemoryDef(StartingPoint.getContext(), nullptr,
                                     nullptr, &StartingPoint, NextID++));

  // 记录包含 MemoryDef 的基本块
  SmallPtrSet<BasicBlock *, 32> DefiningBlocks;

  // 遍历所有待处理的基本块
  for (BasicBlock &B : Blocks) {
    // 当前块是否包含 MemoryDef
    bool InsertIntoDef = false;
    // 当前块的内存访问列表
    AccessList *Accesses = nullptr;
    // 当前块的内存定义列表
    DefsList *Defs = nullptr;
    // 遍历当前基本块内的所有指令
    for (Instruction &I : B) {
      // 为指令创建对应的 MemoryUse/Def 节点, 过滤非内存操作
      MemoryUseOrDef *MUD = createNewAccess(&I, &BAA);
      if (!MUD)
        continue;

      // 初始化当前块的内存访问列表
      if (!Accesses)
        Accesses = getOrCreateAccessList(&B);
      // Accesses 会把当前基本块中所有内存操作都加进来,  无论是读取 MemoryUse 还是修改 MemoryDef
      Accesses->push_back(MUD);

      // 修改内存的操作, store/call/fence 等
      if (isa<MemoryDef>(MUD)) {
        InsertIntoDef = true;   // 标记当前块包含内存定义
        if (!Defs)
          Defs = getOrCreateDefsList(&B);
        // 将 MemoryDef 加入当前块的定义列表
        Defs->push_back(*MUD);
      }
    }

    // 加入 DefiningBlocks 集合
    if (InsertIntoDef)
      DefiningBlocks.insert(&B);
  }

  // 根据 DefiningBlocks 计算迭代支配边界, 在控制流交汇点插入 MemoryPhi
  placePHINodes(DefiningBlocks);

  // SSA 重命名
  SmallPtrSet<BasicBlock *, 16> Visited;
  if (L) {
    // 单循环构建, 仅为循环内的基本块构建 MemorySSA

    // 移除循环预头的 MemoryPhi 节点, 将依赖改为 LiveOnEntry
    if (auto *P = getMemoryAccess(L->getLoopPreheader())) {
      for (Use &U : make_early_inc_range(P->uses()))
        U.set(LiveOnEntryDef.get());
      removeFromLists(P);
    }

    // 限制 SSA 重命名的范围, 仅处理循环内的基本块
    SmallVector<BasicBlock *> ExitBlocks;
    L->getExitBlocks(ExitBlocks);
    Visited.insert_range(ExitBlocks);
    renamePass(DT->getNode(L->getLoopPreheader()), LiveOnEntryDef.get(),
               Visited);
  } else {
    // 全函数构建, 为整个函数的所有基本块构建 MemorySSA

    // 从支配树根节点开始重命名
    renamePass(DT->getRootNode(), LiveOnEntryDef.get(), Visited);
  }

  // 标记不可达块的内存访问依赖为 LiveOnEntry, 避免悬空依赖
  for (auto &BB : Blocks)
    if (!Visited.count(&BB))
      markUnreachableAsLiveOnEntry(&BB);
}
```
看到这我们可以简单总结一下:
- 记录每条指令的内存使用与定义
- 然后调用 `placePHINodes` 计算支配边界, 插入 PHI
- 然后对内存进行 SSA 重命名, 依据构造函数做不同的处理

接下来我们再看看细节:

首先是 `createNewAccess`:
```cpp
template <typename AliasAnalysisType>
MemoryUseOrDef *MemorySSA::createNewAccess(Instruction *I,
                                           AliasAnalysisType *AAP,
                                           const MemoryUseOrDef *Template) {
  // 过滤特殊内置函数, 这些函数虽标记为内存操作, 但实际无真实内存读写, 需提前过滤
  if (IntrinsicInst *II = dyn_cast<IntrinsicInst>(I)) {
    switch (II->getIntrinsicID()) {
    default:
      break;
    case Intrinsic::allow_runtime_check:              // 运行时检查允许
    case Intrinsic::allow_ubsan_check:                // UBSan检查允许
    case Intrinsic::assume:                           // 假设断言
    case Intrinsic::experimental_noalias_scope_decl:  // 无别名作用域声明
    case Intrinsic::pseudoprobe:                      // 伪探针
      return nullptr;
    }
  }

  // 过滤既不读也不写内存
  if (!I->mayReadFromMemory() && !I->mayWriteToMemory())
    return nullptr;

  bool Def, Use;

  // 有模板节点, 复用模板的内存属性, 避免重复分析
  if (Template) {
    Def = isa<MemoryDef>(Template);
    Use = isa<MemoryUse>(Template);
    // 调试模式, 验证模板属性与分析结果的一致
#if !defined(NDEBUG)
    ModRefInfo ModRef = AAP->getModRefInfo(I, std::nullopt);
    bool DefCheck, UseCheck;
    DefCheck = isModSet(ModRef) || isOrdered(I);
    UseCheck = isRefSet(ModRef);
    assert((Def == DefCheck || !DefCheck) &&
           "Memory accesses should only be reduced");
    if (!Def && Use != UseCheck) {
      assert(!UseCheck && "Invalid template");
    }
#endif
  } else {
    // 获取指令的 ModRefInfo, 判断指令是读/写/读写内存
    ModRefInfo ModRef = AAP->getModRefInfo(I, std::nullopt);

    Def = isModSet(ModRef) || isOrdered(I);
    Use = isRefSet(ModRef);
  }

  // 极少数情况, 指令标记为内存操作, 但分析后无读/写属性, 如空调用
  if (!Def && !Use)
    return nullptr;

  // 返回结果
  MemoryUseOrDef *MUD;
  if (Def) {
    MUD = new MemoryDef(I->getContext(), nullptr, I, I->getParent(), NextID++);
  } else {
    MUD = new MemoryUse(I->getContext(), nullptr, I, I->getParent());
    if (isUseTriviallyOptimizableToLiveOnEntry(*AAP, I)) {
      MemoryAccess *LiveOnEntry = getLiveOnEntryDef();
      MUD->setOptimized(LiveOnEntry);
    }
  }
  ValueToMemoryAccess[I] = MUD;
  return MUD;
}

// AAP->getModRefInfo 就是更具 opcode 做匹配
ModRefInfo AAResults::getModRefInfo(const Instruction *I,
                                    const std::optional<MemoryLocation> &OptLoc,
                                    AAQueryInfo &AAQIP) {
  if (OptLoc == std::nullopt) {
    if (const auto *Call = dyn_cast<CallBase>(I))
      return getMemoryEffects(Call, AAQIP).getModRef();
  }

  const MemoryLocation &Loc = OptLoc.value_or(MemoryLocation());

  switch (I->getOpcode()) {
  case Instruction::VAArg:
    return getModRefInfo((const VAArgInst *)I, Loc, AAQIP);
  case Instruction::Load:
    return getModRefInfo((const LoadInst *)I, Loc, AAQIP);
  case Instruction::Store:
    return getModRefInfo((const StoreInst *)I, Loc, AAQIP);
  case Instruction::Fence:
    return getModRefInfo((const FenceInst *)I, Loc, AAQIP);
  case Instruction::AtomicCmpXchg:
    return getModRefInfo((const AtomicCmpXchgInst *)I, Loc, AAQIP);
  case Instruction::AtomicRMW:
    return getModRefInfo((const AtomicRMWInst *)I, Loc, AAQIP);
  case Instruction::Call:
  case Instruction::CallBr:
  case Instruction::Invoke:
    return getModRefInfo((const CallBase *)I, Loc, AAQIP);
  case Instruction::CatchPad:
    return getModRefInfo((const CatchPadInst *)I, Loc, AAQIP);
  case Instruction::CatchRet:
    return getModRefInfo((const CatchReturnInst *)I, Loc, AAQIP);
  default:
    assert(!I->mayReadOrWriteMemory() &&
           "Unhandled memory access instruction!");
    return ModRefInfo::NoModRef;
  }
}
```
核心是根据 opcode 判断读/写内存, 还是普通指令.

然后看 `placePHINodes` 的实现细节:
```cpp
void MemorySSA::placePHINodes(
    const SmallPtrSetImpl<BasicBlock *> &DefiningBlocks) {
  // 根据支配树计算支配边界, 结果存入 IDFBlocks
  ForwardIDFCalculator IDFs(*DT);
  IDFs.setDefiningBlocks(DefiningBlocks);
  SmallVector<BasicBlock *, 32> IDFBlocks;
  IDFs.calculate(IDFBlocks);

  // 遍历所有需要插Phi的块, 逐个创建 MemoryPhi
  for (auto &BB : IDFBlocks)
    createMemoryPhi(BB);
}

MemoryPhi *MemorySSA::createMemoryPhi(BasicBlock *BB) {
  assert(!getMemoryAccess(BB) && "MemoryPhi already exists for this BB");
  // NextID 是 PHI 唯一 ID
  MemoryPhi *Phi = new MemoryPhi(BB->getContext(), BB, NextID++);
  // 将 Phi 插入到块的内存访问列表头部
  insertIntoListsForBlock(Phi, BB, Beginning);
  ValueToMemoryAccess[BB] = Phi;
  return Phi;
}

void MemorySSA::insertIntoListsForBlock(MemoryAccess *NewAccess,
                                        const BasicBlock *BB,
                                        InsertionPlace Point) {
  // 获取/创建当前块的 Accesses 列表, 存储所有内存操作, Phi/Use/Def
  auto *Accesses = getOrCreateAccessList(BB);

  // 插入到块首
  if (Point == Beginning) {
    // 如果是 MemoryPhi 节点: 必须插入到 Accesses/Defs 列表的最前端
    if (isa<MemoryPhi>(NewAccess)) {
      Accesses->push_front(NewAccess);
      auto *Defs = getOrCreateDefsList(BB);
      Defs->push_front(*NewAccess);
    } else {
      // 非 Phi 节点, 插入到所有 Phi 节点之后
      auto AI = find_if_not(
          *Accesses, [](const MemoryAccess &MA) { return isa<MemoryPhi>(MA); });
      Accesses->insert(AI, NewAccess);
      if (!isa<MemoryUse>(NewAccess)) {
        auto *Defs = getOrCreateDefsList(BB);
        auto DI = find_if_not(
            *Defs, [](const MemoryAccess &MA) { return isa<MemoryPhi>(MA); });
        Defs->insert(DI, *NewAccess);
      }
    }
  } else {
    // 插入到块尾
    Accesses->push_back(NewAccess);
    if (!isa<MemoryUse>(NewAccess)) {
      auto *Defs = getOrCreateDefsList(BB);
      Defs->push_back(*NewAccess);
    }
  }
  // 标记该块的编号缓存无效, 后续需重新计算块内节点的执行顺序
  BlockNumberingValid.erase(BB);
}
```
这里就是根据支配树计算支配边界, 然后创建空的 PHI 节点, 然后插入.

最后看 `renamePass`:
```cpp
MemoryAccess *MemorySSA::renameBlock(BasicBlock *BB, MemoryAccess *IncomingVal,
                                     bool RenameAllUses) {
  auto It = PerBlockAccesses.find(BB);
  if (It != PerBlockAccesses.end()) {
    AccessList *Accesses = It->second.get();
    // 按执行顺序遍历所有内存操作
    for (MemoryAccess &L : *Accesses) {
      if (MemoryUseOrDef *MUD = dyn_cast<MemoryUseOrDef>(&L)) {
        // 为 Use/Def 设置依赖 = 当前活跃定义 = IncomingVal
        if (MUD->getDefiningAccess() == nullptr || RenameAllUses)
          MUD->setDefiningAccess(IncomingVal);
        // 如果是 Def, 更新活跃定义 IncomingVal 为当前 Def
        if (isa<MemoryDef>(&L))
          IncomingVal = &L;
      } else {
        // Phi 节点也更新活跃定义 IncomingVal 为当前 Phi
        IncomingVal = &L;
      }
    }
  }
  return IncomingVal;
}

// 将当前块的活跃定义传入后继块的 MemoryPhi 节点, 填充入边依赖
void MemorySSA::renameSuccessorPhis(BasicBlock *BB, MemoryAccess *IncomingVal,
                                    bool RenameAllUses) {
  // 遍历当前块的所有 CFG 后继块
  for (const BasicBlock *S : successors(BB)) {
    auto It = PerBlockAccesses.find(S);
    if (It == PerBlockAccesses.end() || !isa<MemoryPhi>(It->second->front()))
      continue;

    // 只处理有 MemoryPhi 的后继块
    AccessList *Accesses = It->second.get();
    auto *Phi = cast<MemoryPhi>(&Accesses->front());
    if (RenameAllUses) {
      // 覆盖已有入边
      bool ReplacementDone = false;

      // 遍历Phi的所有入边, 找到对应当前块的入边并更新入边的 IncomingVal 为当前值
      for (unsigned I = 0, E = Phi->getNumIncomingValues(); I != E; ++I)
        if (Phi->getIncomingBlock(I) == BB) {
          Phi->setIncomingValue(I, IncomingVal);
          ReplacementDone = true;
        }
      (void) ReplacementDone;
      assert(ReplacementDone && "Incomplete phi during partial rename");
    } else
      // 新增入边
      Phi->addIncoming(IncomingVal, BB);
  }
}

void MemorySSA::renamePass(DomTreeNode *Root, MemoryAccess *IncomingVal,
                           SmallPtrSetImpl<BasicBlock *> &Visited,
                           bool SkipVisited, bool RenameAllUses) {
  assert(Root && "Trying to rename accesses in an unreachable block");

  SmallVector<RenamePassData, 32> WorkStack;
  bool AlreadyVisited = !Visited.insert(Root->getBlock()).second;
  if (SkipVisited && AlreadyVisited)
    return;

  // 处理根节点
  IncomingVal = renameBlock(Root->getBlock(), IncomingVal, RenameAllUses);
  renameSuccessorPhis(Root->getBlock(), IncomingVal, RenameAllUses);
  // 将根节点压入 WorkStack
  WorkStack.push_back({Root, Root->begin(), IncomingVal});

  // 遍历支配树子节点
  while (!WorkStack.empty()) {
    DomTreeNode *Node = WorkStack.back().DTN;
    DomTreeNode::const_iterator ChildIt = WorkStack.back().ChildIt;
    IncomingVal = WorkStack.back().IncomingVal;

    // 子节点迭代器到末尾 → 当前节点处理完毕, 弹出栈
    if (ChildIt == Node->end()) {
      WorkStack.pop_back();
    } else {
      DomTreeNode *Child = *ChildIt;
      ++WorkStack.back().ChildIt;
      BasicBlock *BB = Child->getBlock();
      // 标记该块是否已访问
      AlreadyVisited = !Visited.insert(BB).second;
      if (SkipVisited && AlreadyVisited) {
        // 跳过已访问块, 获取该块的最后一个 Def 作为活跃定义
        if (auto *BlockDefs = getBlockDefs(BB))
          IncomingVal = &*BlockDefs->rbegin();
      } else
        // 重命名块内所有内存操作, 更新活跃定义
        IncomingVal = renameBlock(BB, IncomingVal, RenameAllUses);
      
      // 填充当前块所有后继块中 Phi 节点的入边依赖
      renameSuccessorPhis(BB, IncomingVal, RenameAllUses);
      // 将子节点压入栈, 准备处理其子节点
      WorkStack.push_back({Child, Child->begin(), IncomingVal});
    }
  }
}
```
这个方法就是遍历支配树, 计算, 更新依赖, 填充 PHI 指令.

实验一下:
```ir
define i32 @foo(ptr %p) {
entry:
  br label %loopbegin

loopbegin:
  %n = phi i32 [ 0, %entry ], [ %1, %sw.epilog ]
  %m = alloca i32, align 4
  switch i32 %n, label %sw.default [
    i32 0, label %sw.bb
    i32 1, label %sw.bb1
    i32 2, label %sw.bb2
    i32 3, label %sw.bb3
  ]

sw.bb:
  store i32 1, ptr %m, align 4
  br label %sw.epilog

sw.bb1:
  store i32 2, ptr %m, align 4
  br label %sw.epilog

sw.bb2:
  store i32 3, ptr %m, align 4
  br label %sw.epilog

sw.bb3:
  store i32 4, ptr %m, align 4
  br label %sw.epilog

sw.default:
  store i32 5, ptr %m, align 4
  br label %sw.epilog

sw.epilog:
  %0 = load i32, ptr %m, align 4

  %1 = load volatile i32, ptr %p, align 4
  %2 = icmp eq i32 %0, %1
  br i1 %2, label %exit, label %loopbegin

exit:
  ret i32 %1
}
```
```bash
./build/bin/opt -passes="module(function(print<memoryssa>))" -S /tmp/a.ll
```
```ir
MemorySSA for function: foo
define i32 @foo(ptr %p) {
entry:
  br label %loopbegin

loopbegin:                                        ; preds = %sw.epilog, %entry
; 8 = MemoryPhi({entry,liveOnEntry},{sw.epilog,6})
  %n = phi i32 [ 0, %entry ], [ %1, %sw.epilog ]
  %m = alloca i32, align 4
  switch i32 %n, label %sw.default [
    i32 0, label %sw.bb
    i32 1, label %sw.bb1
    i32 2, label %sw.bb2
    i32 3, label %sw.bb3
  ]

sw.bb:                                            ; preds = %loopbegin
; 1 = MemoryDef(8)
  store i32 1, ptr %m, align 4
  br label %sw.epilog

sw.bb1:                                           ; preds = %loopbegin
; 2 = MemoryDef(8)
  store i32 2, ptr %m, align 4
  br label %sw.epilog

sw.bb2:                                           ; preds = %loopbegin
; 3 = MemoryDef(8)
  store i32 3, ptr %m, align 4
  br label %sw.epilog

sw.bb3:                                           ; preds = %loopbegin
; 4 = MemoryDef(8)
  store i32 4, ptr %m, align 4
  br label %sw.epilog

sw.default:                                       ; preds = %loopbegin
; 5 = MemoryDef(8)
  store i32 5, ptr %m, align 4
  br label %sw.epilog

sw.epilog:                                        ; preds = %sw.default, %sw.bb3, %sw.bb2, %sw.bb1, %sw.bb
; 7 = MemoryPhi({sw.default,5},{sw.bb,1},{sw.bb1,2},{sw.bb2,3},{sw.bb3,4})
; MemoryUse(7)
  %0 = load i32, ptr %m, align 4
; 6 = MemoryDef(7)
  %1 = load volatile i32, ptr %p, align 4
  %2 = icmp eq i32 %0, %1
  br i1 %2, label %exit, label %loopbegin

exit:                                             ; preds = %sw.epilog
  ret i32 %1
}
```