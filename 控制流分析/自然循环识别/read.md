## 简介
- 自然循环是 "最小" 的循环结构, 要求循环头必须支配循环内所有节点(保证只有一个入口)
- SCC 可能包含更复杂的结构: 一个 SCC 可能对应多个嵌套循环, 或包含多个入口的非结构化循环(如某些跳转语句形成的复杂循环), 这些不一定是自然循环.

## 自然循环定义/条件
自然循环是满足以下两个条件的 CFG 子图:
- 存在回边: 存在一条从块 B 到块 H 的边, 且 H 支配 B. 这里 H 称为循环的头节点, B 称为回边的源节点. 
- 头节点唯一: B 可以有多个, H 只能有一个.
- 闭合性: 能不经过 H 到达 B, 且被 H 支配. 循环体包含 H 和所有满足此条件的节点.

## LLVM 中的处理
通过 Tarjan 算法构建 SCC, 找出所有循环, 在从中筛选自然循环做循环有关的优化处理, 而非自然循环, 不将其视为 “循环” 进行优化, 按普通控制流处理.

## llvm 用来识别自然循环的代码
识别自然循环的代码在 `llvm/include/llvm/Support/GenericLoopInfoImpl.h` 中, 让我们来看看, 从 `analyze` 方法看起:
```cpp
/// 分析循环信息，通过后序遍历支配树(DominatorTree)来发现循环，
/// 并在每个子循环中交错进行反向控制流图(CFG)遍历(discoverAndMapSubloop)。
/// 反向遍历会跳过内部子循环，因此这部分算法的时间复杂度与控制流图的边数呈线性关系。
/// 然后在一次正向控制流图遍历(PopulateLoopDFS)中填充子循环和块(Block)向量。
///
/// 在两次控制流图遍历中，每个块会被看到三次：
/// 1) 通过反向控制流图遍历被发现并映射。
/// 2) 在正向深度优先搜索(DFS)控制流图遍历中被访问。
/// 3) 按照正向DFS的后序顺序反向插入到循环中。
///
/// 块向量是包含式的，因此步骤3需要根据循环深度对每个块进行插入操作。
template <class BlockT, class LoopT>
void LoopInfoBase<BlockT, LoopT>::analyze(const DomTreeBase<BlockT> &DomTree) {
  // 对支配树进行后序遍历。
  const DomTreeNodeBase<BlockT> *DomRoot = DomTree.getRootNode();
  for (auto DomNode : post_order(DomRoot)) {

    BlockT *Header = DomNode->getBlock(); // 获取当前支配树节点对应的块，作为潜在的循环头
    SmallVector<BlockT *, 4> Backedges; // 用于存储反向边

    // 检查潜在循环头的每个前驱节点。
    for (const auto Backedge : children<Inverse<BlockT *>>(Header)) {
      // 如果Header支配前驱块，并且前驱块从入口可达, 则这是一条回边，表明可能形成一个新循环
      if (DomTree.dominates(Header, Backedge) &&
          DomTree.isReachableFromEntry(Backedge)) {
        Backedges.push_back(Backedge);
      }
    }
    // 执行反向控制流图遍历来发现并映射这个循环中的块。
    if (!Backedges.empty()) {
      LoopT *L = AllocateLoop(Header); // 为新循环分配循环对象
      discoverAndMapSubloop(L, ArrayRef<BlockT *>(Backedges), this, DomTree); // 发现并映射子循环中的块
    }
  }
  // 执行一次正向控制流图遍历来为所有循环填充块和子循环向量。
  PopulateLoopsDFS<BlockT, LoopT> DFS(this); // 创建正向DFS对象
  DFS.traverse(DomRoot->getBlock()); // 从支配树根节点对应的块开始遍历
}
```
树的前序遍历-根左右, 中序遍历-左根右, 后序遍历-左右根. 这里的前中后说的是根的位置. 对支配树进行后序遍历的原因是要先处理内层的循环, 在处理外层循环, 在支配书中外层循环的头节点是内层循环头节点的祖先, 内层循环的节点会被标记为 “已属于某个循环”，外层循环的分析会自动跳过这些节点, 这种 “自底向上” 的处理方式，确保了每个循环只被分析一次，且嵌套关系被正确记录。

LLVM 中遍历后继节点:
```cpp
// 遍历当前块(Header)的所有后继节点
for (auto SuccBlock : children<BlockT*>(Header)) {
    // SuccBlock 是当前块的一个后继节点
    // 处理后继节点...
}
```
LLVM 中遍历前驱节点:
```cpp
// 遍历当前块(Header)的所有前驱节点
for (auto PredBlock : children<Inverse<BlockT*>>(Header)) {
    // PredBlock 是当前块的一个前驱节点
    // 处理前驱节点...
}
```

所以 `for (const auto Backedge : children<Inverse<BlockT *>>(Header))` 就是在遍历每一个潜在头节点的前驱块, 看看是否满足 `存在回边` 这一条件. 同时也隐含了 `头节点唯一` 这一条件, 而如果存在多个候选头节点，它们必须都支配回边起点，这意味着这些候选头节点之间形成支配关系, 在后序遍历下, 内层循环会先被识别, 然后标记.

再来看看 `discoverAndMapSubloop`:
```cpp
//===----------------------------------------------------------------------===//
/// 稳定的循环信息分析 - 使用稳定的迭代器构建循环树，使得结果不依赖于使用列表(块的前驱)的顺序。

/// 发现具有指定反向边的子循环，满足以下条件：
/// 1. 此循环内的所有块都被映射到此循环或其子循环。
/// 2. 此循环内的所有子循环都将它们的父循环设置为这个循环或其子循环。
template <class BlockT, class LoopT>
static void discoverAndMapSubloop(LoopT *L, ArrayRef<BlockT *> Backedges,
                                  LoopInfoBase<BlockT, LoopT> *LI,
                                  const DomTreeBase<BlockT> &DomTree) {
  typedef GraphTraits<Inverse<BlockT *>> InvBlockTraits;

  unsigned NumBlocks = 0; // 用于记录此循环内的块数量
  unsigned NumSubloops = 0; // 用于记录此循环内的子循环数量

  // 使用工作列表进行反向控制流图遍历。
  std::vector<BlockT *> ReverseCFGWorklist(Backedges.begin(), Backedges.end());
  while (!ReverseCFGWorklist.empty()) {
    BlockT *PredBB = ReverseCFGWorklist.back(); // 获取工作列表中的最后一个块
    ReverseCFGWorklist.pop_back(); // 从工作列表中移除该块

    LoopT *Subloop = LI->getLoopFor(PredBB); // 获取该块所属的循环
    if (!Subloop) {
      if (!DomTree.isReachableFromEntry(PredBB)) // 如果该块从入口不可达，则跳过
        continue;

      // 这是一个未发现的块。将其映射到当前循环。
      LI->changeLoopFor(PredBB, L); // 将块映射到当前循环
      ++NumBlocks; // 块数量加1
      if (PredBB == L->getHeader()) // 跳过循环头的前驱
        continue;
      // 将该块的所有前驱块加入工作列表。
      ReverseCFGWorklist.insert(ReverseCFGWorklist.end(),
                                InvBlockTraits::child_begin(PredBB),
                                InvBlockTraits::child_end(PredBB));
    } else {
      // 这是一个已发现的块。找到它最外层的已发现循环。
      Subloop = Subloop->getOutermostLoop();

      // 如果它已经被发现是当前循环的子循环，则跳过。
      if (Subloop == L)
        continue;

      // 发现当前循环的一个子循环。
      Subloop->setParentLoop(L); // 将子循环的父循环设置为当前循环
      ++NumSubloops; // 子循环数量加1
      NumBlocks += Subloop->getBlocksVector().capacity(); // 更新块数量
      PredBB = Subloop->getHeader(); // 获取子循环的头块
      // 继续沿着不是来自子循环树内部的循环边的前驱进行遍历。
      // 注意，一个前驱可能直接到达另一个尚未被发现为当前循环子循环的子循环，我们必须对其进行遍历。
      for (const auto Pred : children<Inverse<BlockT *>>(PredBB)) {
        if (LI->getLoopFor(Pred) != Subloop)
          ReverseCFGWorklist.push_back(Pred);
      }
    }
  }
  L->getSubLoopsVector().reserve(NumSubloops); // 为子循环向量预留空间
  L->reserveBlocks(NumBlocks); // 为块向量预留空间
}
```
实现的逻辑非常清晰, 从反向边出发，反向遍历 CFG, 若块未归属任何循环，将其映射到当前循环 L,  若块已属其他循环，将其所在最外层循环的父循环设置为 L, 大体是这样. 

沿 CFG 反向遍历, 即遍历前驱节点, 收集所有能到达 B 且不经过 H 的节点, 根据 `X 支配 Y, 那么 X 就支配 Y 的前驱` 这一推论, 也符合了 `闭合性` 这一条件.

再来看 `traverse`:
```cpp
/// 用于在循环内执行正向DFS的顶级驱动函数
template <class BlockT, class LoopT>
void PopulateLoopsDFS<BlockT, LoopT>::traverse(BlockT *EntryBlock) {
  // 按后序(post-order)遍历从EntryBlock可达的所有基本块
  // 对每个基本块调用insertIntoLoop()方法，将其插入到适当的循环结构中
  for (BlockT *BB : post_order(EntryBlock))
    insertIntoLoop(BB);
}

/// 将单个基本块按后序添加到其祖先循环中
/// 如果该基本块是子循环的头块，会将子循环按后序添加到其父循环中
/// 然后反转已完成的子循环的基本块和子循环向量，以获得反向后序(RPO)
template <class BlockT, class LoopT>
void PopulateLoopsDFS<BlockT, LoopT>::insertIntoLoop(BlockT *Block) {
  // 获取当前基本块所属的循环(如果存在)
  LoopT *Subloop = LI->getLoopFor(Block);
  
  // 如果当前块是某个循环的头块(说明我们已处理完该循环的所有基本块)
  if (Subloop && Block == Subloop->getHeader()) {
    // 这部分代码在处理完子循环的所有基本块后执行一次
    
    // 如果不是最外层循环，将当前子循环添加到其父循环的子循环列表中
    if (!Subloop->isOutermost())
      Subloop->getParentLoop()->getSubLoopsVector().push_back(Subloop);
    // 如果是最外层循环，将其添加到顶级循环列表中
    else
      LI->addTopLevelLoop(Subloop);

    // 为了方便处理，基本块和子循环是以后序插入的
    // 反转列表(除了始终位于开头的循环头块)以获得反向后序(RPO)
    Subloop->reverseBlock(1);  // 从索引1开始反转，保留头块在首位
    std::reverse(Subloop->getSubLoopsVector().begin(),
                 Subloop->getSubLoopsVector().end());

    // 移动到父循环，准备将当前块添加到所有祖先循环中
    Subloop = Subloop->getParentLoop();
  }
  
  // 遍历当前循环及其所有祖先循环
  // 将当前基本块添加到每个循环的块列表中
  for (; Subloop; Subloop = Subloop->getParentLoop())
    Subloop->addBlockEntry(Block);
}
```
实现也很清晰, 核心思想就是子循环的基本块也是父循环的基本块, 子循环与父循环是双向映射的. 注意这里处理的都是前面识别出在循环中的基本块. 