## 简介
在编译优化中, 识别循环结构是非常重要的基础工作, 而强连通分量(SCC)是识别循环的核心工具

对于 SCC 只有鲸书的第七章 141 页有所提及(如果我没看漏的话), 使用 Tarjan 算法, 实话说不容易看懂, 下面这个信息奥赛的老师讲解的很清楚:
- https://www.bilibili.com/video/BV1SY411M7Tv/?spm_id_from=333.1387.favlist.content.click&vd_source=fd7f17cce2cc926b1a321c3996d115e0

## SCC 有什么用?
- 划分结构: 每个 SCC 内部任意节点互相可达, 即 “强连通”, 构建 SCC 后, 原图被拆分为多个 SCC, 这些 SCC 之间形成的新图(SCC-DAG)必然无环. 
- 识别循环: CFG 中的 SCC直接对应程序的循环体(多节点 SCC 必为循环, 单节点 SCC 带自环也为循环). 只有识别出循环, 才能进行循环展开、不变式外提等关键优化. 
- .....

## llvm 中 用于构建 SCC 的代码
经过查找在 `llvm/include/llvm/ADT/SCCIterator.h` 文件中实现了非递归实现的 Tarjan 算法构建 SCC. 让我们来看一看, 核心类是:
```cpp
/// 枚举有向图的强连通分量(SCC), 按照 SCC 有向无环图(DAG)的逆拓扑顺序. 
///
/// 该实现使用 Tarjan 的深度优先搜索(DFS)算法, 通过内部栈构建特定 SCC 中的节点向量. 
/// 注意, 这是一个单向迭代器, 因此不能回溯或重新访问节点. 
template <class GraphT, class GT = GraphTraits<GraphT>>
class scc_iterator : public iterator_facade_base<
                         scc_iterator<GraphT, GT>, std::forward_iterator_tag,
                         const std::vector<typename GT::NodeRef>, ptrdiff_t> {
  using NodeRef = typename GT::NodeRef; ///< 图节点的引用类型
  using ChildItTy = typename GT::ChildIteratorType; ///< 子节点迭代器类型
  using SccTy = std::vector<NodeRef>; ///< 强连通分量类型, 存储节点引用的向量
  using reference = typename scc_iterator::reference; ///< 引用类型

  /// DFS 过程中 VisitStack 的元素. 
  struct StackElement {
    NodeRef Node;         ///< 当前节点指针
    ChildItTy NextChild;  ///< 下一个子节点, 在 DFS 过程中会修改
    unsigned MinVisited;  ///< 当前节点所有子节点的最小访问序号

    StackElement(NodeRef Node, const ChildItTy &Child, unsigned Min)
        : Node(Node), NextChild(Child), MinVisited(Min) {}

    bool operator==(const StackElement &Other) const {
      return Node == Other.Node &&
             NextChild == Other.NextChild &&
             MinVisited == Other.MinVisited;
    }
  };

  /// 用于检测栈上是否有一个完整的 SCC 的访问计数器. 
  /// visitNum 是全局计数器. 
  ///
  /// nodeVisitNumbers 是每个节点的访问序号, 也用作 DFS 标志. 
  unsigned visitNum;
  DenseMap<NodeRef, unsigned> nodeVisitNumbers;

  /// 存储 SCC 中的节点的栈. 
  std::vector<NodeRef> SCCNodeStack;

  /// 当前 SCC, 通过 operator*() 获取. 
  SccTy CurrentSCC;

  /// DFS 栈, 用于维护顺序. 栈顶包含当前节点、下一个要访问的子节点以及所有子节点的最小访问序号. 
  std::vector<StackElement> VisitStack;

  /// 非递归 DFS 遍历中的单次“访问”. 
  void DFSVisitOne(NodeRef N);

  /// 基于栈的 DFS 遍历；定义在下方. 
  void DFSVisitChildren();

  /// 使用 DFS 遍历计算下一个 SCC. 
  void GetNextSCC();

  scc_iterator(NodeRef entryN) : visitNum(0) {
    DFSVisitOne(entryN);
    GetNextSCC();
  }

  /// 当 DFS 栈为空时, 表示迭代结束. 
  scc_iterator() = default;

public:
  static scc_iterator begin(const GraphT &G) {
    return scc_iterator(GT::getEntryNode(G));
  }
  static scc_iterator end(const GraphT &) { return scc_iterator(); }

  /// 直接的循环终止测试, 比与 \c end() 比较更高效. 
  bool isAtEnd() const {
    assert(!CurrentSCC.empty() || VisitStack.empty());
    return CurrentSCC.empty();
  }

  bool operator==(const scc_iterator &x) const {
    return VisitStack == x.VisitStack && CurrentSCC == x.CurrentSCC;
  }

  scc_iterator &operator++() {
    GetNextSCC();
    return *this;
  }

  reference operator*() const {
    assert(!CurrentSCC.empty() && "Dereferencing END SCC iterator!");
    return CurrentSCC;
  }

  /// 检测当前 SCC 是否包含环. 
  ///
  /// 如果 SCC 包含多个节点, 则显然存在环. 如果只有一个节点, 在可能仍存环, 前提是该节点有指向自身的边. 
  bool hasCycle() const;

  /// 告知 \c scc_iterator 指定的 \c Old 节点已被删除, 应使用 \c New 节点替代. 
  void ReplaceNode(NodeRef Old, NodeRef New) {
    assert(nodeVisitNumbers.count(Old) && "Old not in scc_iterator?");
    // 分两步进行赋值, 以防 'New' 尚未在映射中, 插入它会导致映射增长. 
    auto tempVal = nodeVisitNumbers[Old];
    nodeVisitNumbers[New] = tempVal;
    nodeVisitNumbers.erase(Old);
  }
};
```
从 begin 方法开始:
```cpp
static scc_iterator begin(const GraphT &G) {
  return scc_iterator(GT::getEntryNode(G));
}
```
调用构造函数, 将 entry node 穿进去, 在看看构造函数:
```cpp
scc_iterator(NodeRef entryN) : visitNum(0) {
  DFSVisitOne(entryN);
  GetNextSCC();
}
```
首先来看看 `DFSVisitOne`:
```cpp
// 模板函数：对单个节点执行深度优先搜索(DFS)访问, 用于强连通分量(SCC)的查找
// GraphT：图数据结构的类型
// GT：图遍历辅助工具类的类型, 提供如获取子节点迭代器等方法
template <class GraphT, class GT>
void scc_iterator<GraphT, GT>::DFSVisitOne(NodeRef N) {
  // 增加访问计数器, 用于记录节点被访问的顺序
  ++visitNum;
  
  // 记录当前节点的访问序号, 用于后续判断节点间的可达关系
  nodeVisitNumbers[N] = visitNum;
  
  // 将当前节点压入SCC节点栈, 用于暂存可能属于同一强连通分量的节点
  SCCNodeStack.push_back(N);
  
  // 将当前节点信息压入DFS访问栈：包含节点引用、子节点迭代器和访问序号
  // 用于模拟递归的DFS过程, 实现非递归版本的深度优先搜索
  VisitStack.push_back(StackElement(N, GT::child_begin(N), visitNum));
  
#if 0 // 调试时可启用此代码块
  // 输出调试信息：当前处理的节点及其访问序号
  dbgs() << "TarjanSCC: Node " << N <<
        " : visitNum = " << visitNum << "\n";
#endif
}
```
DFSVisitOne 函数负责初始化:
- 单个节点访问序号 visitNum
- 将 node 与对应序号做映射数组 nodeVisitNumbers
- SCC 的候选栈 SCCNodeStack, 记录当前 DFS 路径上的所有节点, 同一 SCC 的节点必然在 DFS 树的同一条路径上, 且根节点是路径上首个满足 low == visitNum 的节点
- 用来非递归深度优先遍历的栈 VisitStack. 

在来看看 GetNextSCC:
```cpp
// 模板函数：获取下一个强连通分量(SCC)
// GraphT：图数据结构的类型
// GT：图遍历辅助工具类的类型
template <class GraphT, class GT> 
void scc_iterator<GraphT, GT>::GetNextSCC() {
  CurrentSCC.clear();  // 清空当前SCC容器, 为存储下一个SCC做准备
  
  // 当DFS访问栈不为空时, 继续处理
  while (!VisitStack.empty()) {
    // 处理当前节点的所有子节点(继续深度优先搜索)
    DFSVisitChildren();

    // 获取栈顶节点(当前正在处理的叶子节点)的信息
    NodeRef visitingN = VisitStack.back().Node;  // 当前节点引用
    unsigned minVisitNum = VisitStack.back().MinVisited;  // 该节点可达的最小访问序号
    // 断言：确保当前节点的子节点迭代器已到达末尾(所有子节点都已处理)
    assert(VisitStack.back().NextChild == GT::child_end(visitingN));
    VisitStack.pop_back();  // 将处理完毕的节点从栈中移除

    // 将当前节点的最小访问序号向上传播给父节点
    // 这是Tarjan算法的关键, 用于判断父节点是否能到达更小访问序号的节点
    if (!VisitStack.empty() && VisitStack.back().MinVisited > minVisitNum)
      VisitStack.back().MinVisited = minVisitNum;

#if 0  // 调试时可启用此代码块
    // 输出调试信息：弹出的节点、其最小访问序号和自身访问序号
    dbgs() << "TarjanSCC: Popped node " << visitingN <<
          " : minVisitNum = " << minVisitNum << "; Node visit num = " <<
          nodeVisitNumbers[visitingN] << "\n";
#endif

    // 如果当前节点的最小访问序号不等于自身访问序号, 说明它不是SCC的根节点
    // 继续处理栈中的其他节点
    if (minVisitNum != nodeVisitNumbers[visitingN])
      continue;

    // 找到一个完整的SCC！该SCC包含SCCNodeStack中从栈顶到当前节点的所有节点
    // 将这些节点复制到CurrentSCC中, 并重置它们的访问标记
    // 这样做会暂停DFS遍历, 直到下一次调用++运算符时继续
    do {
      // 将栈顶节点加入当前SCC
      CurrentSCC.push_back(SCCNodeStack.back());
      SCCNodeStack.pop_back();  // 从栈中移除该节点
      // 标记该节点已被分配到某个SCC(使用~0U表示无效值)
      nodeVisitNumbers[CurrentSCC.back()] = ~0U;
    } while (CurrentSCC.back() != visitingN);  // 直到弹出的是当前SCC的根节点
    
    return;  // 返回, 当前SCC已找到
  }
}

// 模板函数：深度优先搜索(DFS)中处理当前栈顶节点的所有子节点
// 核心作用：遍历子节点, 更新访问状态和最小可达访问序号, 推动DFS进程
// GraphT：图数据结构的类型
// GT：图遍历辅助工具类的类型, 提供获取子节点迭代器等方法
template <class GraphT, class GT>
void scc_iterator<GraphT, GT>::DFSVisitChildren() {
  // 断言：DFS访问栈不能为空(确保有可处理的当前节点)
  assert(!VisitStack.empty());

  // 循环处理栈顶节点的所有未遍历子节点：当子节点迭代器未到末尾时持续执行
  while (VisitStack.back().NextChild != GT::child_end(VisitStack.back().Node)) {
    // 栈顶(TOS, Top Of Stack)节点仍有未处理的子节点, 继续执行DFS
    // 获取当前待处理的子节点, 并将迭代器后移(标记为"已访问过该子节点")
    NodeRef childN = *VisitStack.back().NextChild++;

    // 在nodeVisitNumbers(节点访问序号映射表)中查找该子节点
    typename DenseMap<NodeRef, unsigned>::iterator Visited =
        nodeVisitNumbers.find(childN);

    if (Visited == nodeVisitNumbers.end()) {
      // 子节点从未被访问过：初始化其DFS状态
      // 调用DFSVisitOne为该子节点分配访问序号、压入SCC候选栈和DFS访问栈
      DFSVisitOne(childN);
      // 继续处理栈顶新压入的子节点的子节点(深度优先特性)
      continue;
    }

    // 子节点已被访问过：获取其访问序号
    unsigned childNum = Visited->second;

    // 更新当前栈顶节点的最小可达访问序号(MinVisited)
    // 若子节点的访问序号更小, 说明当前节点可通过该子节点到达更早访问的节点
    if (VisitStack.back().MinVisited > childNum)
      VisitStack.back().MinVisited = childNum;
  }
}
```
![alt text](image.png)
可以看到这里的算法和视频中是一样的, 只不过 llvm 是用 VisitStack 实现非递归的 dfs, 视频中是用递归的 dfs, 这个算法的核心思想是 DFS 遍历控制流程图同时更新节点的访问序号和最小访问序号(从该节点出发, 通过 DFS 树的边或回边可到达的所有节点中, 最小的访问序号), 然后最小访问序号等于自身访问序号就构成了一个 SCC. 

DFSVisitChildren 函数只负责一条路走到黑, 一直往 VisitStack 中 push, 并更新节点的 NextChild, 防止重复访问, 然后 GetNextSCC 中才会回溯, 将 VisitStack pop_back, 并检查是否构成了 SCC, 最后将找到的 SCC 存入 CurrentSCC 属性中. 

提供了 ++ 操作符重载:
```cpp
scc_iterator &operator++() {
  GetNextSCC();
  return *this;
}
```

在来看看 hasCycle:
```cpp
// 模板函数：判断当前强连通分量(SCC)是否包含环
// GraphT：图数据结构的类型
// GT：图遍历辅助工具类的类型, 提供获取子节点迭代器等方法
template <class GraphT, class GT>
bool scc_iterator<GraphT, GT>::hasCycle() const {
    // 断言：当前SCC容器不能为空(避免对结束状态的迭代器解引用)
    assert(!CurrentSCC.empty() && "Dereferencing END SCC iterator!");

    // 情况1：若SCC包含多个节点, 根据强连通分量定义, 必然存在环
    // (强连通分量要求任意两节点互相可达, 多节点时至少形成一个闭合回路)
    if (CurrentSCC.size() > 1)
      return true;

    // 情况2：若SCC仅含单个节点, 需检查该节点是否有指向自身的自环
    NodeRef N = CurrentSCC.front();  // 获取SCC中唯一的节点
    // 遍历该节点的所有子节点
    for (ChildItTy CI = GT::child_begin(N), CE = GT::child_end(N); CI != CE;
         ++CI) {
      // 若存在子节点等于自身(即自环), 则该SCC包含环
      if (*CI == N)
        return true;
    }

    // 情况3：单个节点且无自环, SCC不包含环
    return false;
  }
```
一个查看 CurrentSCC 是否包含环的小函数.