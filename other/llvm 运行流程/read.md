## 简介
简单的将 LLVM 整个流程串一遍, 走马观花的领略一下整个架构, 源代码 -> AST -> LLVM IR -> 机器码. 从


## 测试代码
test.cpp
```cpp
int main()
{
  int a = 1, b;
  b = 2;
  int c = a + b;
  a = c * b;
  return 0;
}
```

# 1. 生成 AST
clang -Xclang -ast-dump -fsyntax-only your_source.c

# 2. 生成未优化 IR (文本)
clang -S -emit-llvm your_source.c -o unoptimized.ll

# 3. (可选) 生成未优化 IR (bitcode)
clang -emit-llvm -c your_source.c -o unoptimized.bc

# 4. 应用优化并生成优化后 IR (文本)
clang -S -emit-llvm -O2 your_source.c -o optimized.ll
# 或使用 opt
opt -O2 unoptimized.bc -o optimized.bc
llvm-dis optimized.bc -o optimized.ll

# 5. 从优化后 IR 生成汇编
llc optimized.ll -o output.s

# 6. 从汇编生成目标文件
clang -c output.s -o output.o

# 7. 链接成可执行文件 (如果需要)
clang output.o -o your_program


## 源代码 -> AST
先用 LLVM 的组件试一试:
```bash
clang++ -Xclang -ast-dump -fsyntax-only test.cpp
```
输出:
```txt
TranslationUnitDecl 0x634a8f6ea108 <<invalid sloc>> <invalid sloc>
|-TypedefDecl 0x634a8f6ea970 <<invalid sloc>> <invalid sloc> implicit __int128_t '__int128'
| `-BuiltinType 0x634a8f6ea6d0 '__int128'
|-TypedefDecl 0x634a8f6ea9e0 <<invalid sloc>> <invalid sloc> implicit __uint128_t 'unsigned __int128'
| `-BuiltinType 0x634a8f6ea6f0 'unsigned __int128'
|-TypedefDecl 0x634a8f6ead58 <<invalid sloc>> <invalid sloc> implicit __NSConstantString '__NSConstantString_tag'
| `-RecordType 0x634a8f6eaad0 '__NSConstantString_tag'
|   `-CXXRecord 0x634a8f6eaa38 '__NSConstantString_tag'
|-TypedefDecl 0x634a8f6eadf0 <<invalid sloc>> <invalid sloc> implicit __builtin_ms_va_list 'char *'
| `-PointerType 0x634a8f6eadb0 'char *'
|   `-BuiltinType 0x634a8f6ea1b0 'char'
|-TypedefDecl 0x634a8f732ba8 <<invalid sloc>> <invalid sloc> implicit __builtin_va_list '__va_list_tag[1]'
| `-ConstantArrayType 0x634a8f732b50 '__va_list_tag[1]' 1 
|   `-RecordType 0x634a8f6eaee0 '__va_list_tag'
|     `-CXXRecord 0x634a8f6eae48 '__va_list_tag'
`-FunctionDecl 0x634a8f732c50 <test.cpp:2:1, line:9:1> line:2:5 main 'int ()'
  `-CompoundStmt 0x634a8f733148 <line:3:1, line:9:1>
    |-DeclStmt 0x634a8f732ea8 <line:4:5, col:17>
    | |-VarDecl 0x634a8f732d88 <col:5, col:13> col:9 used a 'int' cinit
    | | `-IntegerLiteral 0x634a8f732df0 <col:13> 'int' 1
    | `-VarDecl 0x634a8f732e28 <col:5, col:16> col:16 used b 'int'
    |-BinaryOperator 0x634a8f732f00 <line:5:5, col:9> 'int' lvalue '='
    | |-DeclRefExpr 0x634a8f732ec0 <col:5> 'int' lvalue Var 0x634a8f732e28 'b' 'int'
    | `-IntegerLiteral 0x634a8f732ee0 <col:9> 'int' 2
    |-DeclStmt 0x634a8f733030 <line:6:5, col:18>
    | `-VarDecl 0x634a8f732f38 <col:5, col:17> col:9 used c 'int' cinit
    |   `-BinaryOperator 0x634a8f733010 <col:13, col:17> 'int' '+'
    |     |-ImplicitCastExpr 0x634a8f732fe0 <col:13> 'int' <LValueToRValue>
    |     | `-DeclRefExpr 0x634a8f732fa0 <col:13> 'int' lvalue Var 0x634a8f732d88 'a' 'int'
    |     `-ImplicitCastExpr 0x634a8f732ff8 <col:17> 'int' <LValueToRValue>
    |       `-DeclRefExpr 0x634a8f732fc0 <col:17> 'int' lvalue Var 0x634a8f732e28 'b' 'int'
    |-BinaryOperator 0x634a8f7330f8 <line:7:5, col:13> 'int' lvalue '='
    | |-DeclRefExpr 0x634a8f733048 <col:5> 'int' lvalue Var 0x634a8f732d88 'a' 'int'
    | `-BinaryOperator 0x634a8f7330d8 <col:9, col:13> 'int' '*'
    |   |-ImplicitCastExpr 0x634a8f7330a8 <col:9> 'int' <LValueToRValue>
    |   | `-DeclRefExpr 0x634a8f733068 <col:9> 'int' lvalue Var 0x634a8f732f38 'c' 'int'
    |   `-ImplicitCastExpr 0x634a8f7330c0 <col:13> 'int' <LValueToRValue>
    |     `-DeclRefExpr 0x634a8f733088 <col:13> 'int' lvalue Var 0x634a8f732e28 'b' 'int'
    `-ReturnStmt 0x634a8f733138 <line:8:5, col:12>
      `-IntegerLiteral 0x634a8f733118 <col:12> 'int' 0
```
来到 `clang/lib/Lex/Lexer.cpp` 中看 Lex 方法: 
```cpp
bool Lexer::Lex(Token &Result) {
  // 确保当前词法分析器不是用于处理依赖指令的词法分析器. 
  assert(!isDependencyDirectivesLexer());

  // 开始一个新的词法单元. 
  Result.startToken();

  // 为 LexTokenInternal 函数设置一些空白字符相关的标志. 
  if (IsAtStartOfLine) {
    // 如果当前在行的开头, 设置 StartOfLine 标志. 
    Result.setFlag(Token::StartOfLine);
    // 重置 IsAtStartOfLine 标志. 
    IsAtStartOfLine = false;
  }

  if (HasLeadingSpace) {
    // 如果有前导空白字符, 设置 LeadingSpace 标志. 
    Result.setFlag(Token::LeadingSpace);
    // 重置 HasLeadingSpace 标志. 
    HasLeadingSpace = false;
  }

  if (HasLeadingEmptyMacro) {
    // 如果有前导空宏, 设置 LeadingEmptyMacro 标志. 
    Result.setFlag(Token::LeadingEmptyMacro);
    // 重置 HasLeadingEmptyMacro 标志. 
    HasLeadingEmptyMacro = false;
  }

  // 保存当前是否在物理行的开头. 
  bool atPhysicalStartOfLine = IsAtPhysicalStartOfLine;
  // 重置 IsAtPhysicalStartOfLine 标志. 
  IsAtPhysicalStartOfLine = false;

  // 检查是否处于原始模式(raw mode). 
  bool isRawLex = isLexingRawMode();
  // 使用 (void) 来避免未使用的变量警告. 
  (void) isRawLex;

  // 调用 LexTokenInternal 函数进行实际的词法分析. 
  bool returnedToken = LexTokenInternal(Result, atPhysicalStartOfLine);

  // 在调用 LexTokenInternal 后, 词法分析器可能会被销毁. 
  // 确保在原始模式下, 词法分析必须成功. 
  assert((returnedToken || !isRawLex) && "Raw lex must succeed");

  // 返回是否成功返回了一个词法单元. 
  return returnedToken;
}

/// Reset all flags to cleared.
void startToken() {
  Kind = tok::unknown;
  Flags = 0;
  PtrData = nullptr;
  UintData = 0;
  Loc = SourceLocation().getRawEncoding();
}
```
实际的工作在 LexTokenInternal 方法中完成:
```cpp
/// LexTokenInternal - 实现C家族语言的基础词法分析器, 是性能极度关键的代码段. 
/// 函数假设源代码缓冲区末尾有null字符('\0'), 返回的是“预处理Token”而非普通Token, 
/// 因此属于内部接口. 调用前需确保Result的Flags已被清空. 
bool Lexer::LexTokenInternal(Token &Result, bool TokAtPhysicalStartOfLine) {
LexStart: // 词法分析入口标签, 用于循环重试
  assert(!Result.needsCleaning() && "Result needs cleaning");
  assert(!Result.hasPtrData() && "Result has not been reset");

  // CurPtr - 将缓冲区指针BufferPtr缓存到自动变量, 减少成员变量访问开销(性能优化)
  const char *CurPtr = BufferPtr;

  // 处理Token间常见的水平空白(空格、制表符等)
  if (isHorizontalWhitespace(*CurPtr)) {
    // 循环跳过所有连续的水平空白
    do {
      ++CurPtr;
    } while (isHorizontalWhitespace(*CurPtr));

    // 若开启“保留空白模式”(如需要保留代码格式), 直接返回跳过的空白作为Token
    // 下一次词法分析会返回空白后的有效Token
    if (isKeepWhitespaceMode()) {
      // 构造空白对应的Token(类型为tok::unknown)
      FormTokenWithChars(Result, CurPtr, tok::unknown);
      // FIXME：下一个Token的LeadingSpace标志会丢失, 待修复
      return true;
    }

    // 更新缓冲区指针到空白后的位置, 标记当前Token前有空白
    BufferPtr = CurPtr;
    Result.setFlag(Token::LeadingSpace);
  }

  // 临时变量：用于存储字符长度(处理多字节字符如UTF-8时使用)
  unsigned SizeTmp, SizeTmp2;  

  // 读取当前字符, 并将缓冲区指针向前推进(消费该字符)
  char Char = getAndAdvanceChar(CurPtr, Result);
  // 存储当前解析出的Token类型
  tok::TokenKind Kind;

  // 若当前字符不是垂直空白(换行、回车), 重置换行指针(标记非行首位置)
  if (!isVerticalWhitespace(Char))
    NewLinePtr = nullptr;

  // 核心分支：按当前字符类型解析不同Token
  switch (Char) {
  case 0:  // 处理null字符('\0')
    // 检查是否到达文件末尾(缓冲区末尾的null字符)
    if (CurPtr-1 == BufferEnd)
      // 解析文件结束Token, 返回分析结果
      return LexEndOfFile(Result, CurPtr-1);

    // 检查当前位置是否为代码补全点(如IDE的自动补全触发位置)
    if (isCodeCompletionPoint(CurPtr-1)) {
      // 初始化Token, 构造代码补全类型Token
      Result.startToken();
      FormTokenWithChars(Result, CurPtr, tok::code_completion);
      return true;
    }

    // 非文件末尾的null字符属于非法字符, 输出诊断信息(非原始模式下)
    if (!isLexingRawMode())
      Diag(CurPtr-1, diag::null_in_file); // 提示“文件中存在null字符”
    // 标记后续Token前有空白
    Result.setFlag(Token::LeadingSpace);
    // 跳过后续空白(若开启保留空白模式, 直接返回空白Token)
    if (SkipWhitespace(Result, CurPtr, TokAtPhysicalStartOfLine))
      return true; // 保留空白模式：返回空白Token

    // 词法分析器状态未变, 跳转到入口重试(手动消除尾递归, 避免栈溢出)
    goto LexNextToken;

  case 26:  // 处理DOS/CP/M系统的EOF标记(Ctrl+Z, ASCII码26)
    // 若开启微软扩展模式, 将Ctrl+Z视为文件结束
    if (LangOpts.MicrosoftExt) {
      if (!isLexingRawMode())
        // 提示“微软扩展：Ctrl+Z被视为文件结束”
        Diag(CurPtr-1, diag::ext_ctrl_z_eof_microsoft);
      return LexEndOfFile(Result, CurPtr-1);
    }

    // 未开启微软扩展时, Ctrl+Z视为无效字符, 标记Token类型为unknown
    Kind = tok::unknown;
    break;

  case '\r': // 处理回车符(Windows换行格式：\r\n)
    // 若回车后紧跟换行, 一并消费换行符(统一转换为单换行)
    if (CurPtr[0] == '\n')
      (void)getAndAdvanceChar(CurPtr, Result); // 强制转换为void, 避免未使用警告
    // 穿透到'\n'分支继续处理(C++11特性：[[fallthrough]]显式标记穿透)
    [[fallthrough]];
  case '\n': // 处理换行符
    // 若当前正在解析预处理指令(如#define), 换行表示指令结束, 返回EOD(指令结束)Token
    if (ParsingPreprocessorDirective) {
      // 标记预处理指令解析完成
      ParsingPreprocessorDirective = false;

      // 恢复注释保留模式(若指令解析时临时关闭)
      if (PP)
        resetExtendedTokenMode();

      // 消费换行后回到行首, 更新行首标记
      IsAtStartOfLine = true;
      IsAtPhysicalStartOfLine = true;
      NewLinePtr = CurPtr - 1; // 记录换行位置

      // 指令结束, Token类型为EOD
      Kind = tok::eod;
      break;
    }

    // 非预处理指令场景：清除“前有空白”标记(换行本身视为分隔, 无需额外标记)
    Result.clearFlag(Token::LeadingSpace);

    // 跳过换行后的空白(若开启保留空白模式, 返回空白Token)
    if (SkipWhitespace(Result, CurPtr, TokAtPhysicalStartOfLine))
      return true; // 保留空白模式：返回空白Token

    // 仅解析到空白, 跳转到入口重试
    goto LexNextToken;

  // 处理水平空白字符(空格、制表符、换页符、垂直制表符)
  case ' ': case '\t': case '\f': case 'v':
  SkipHorizontalWhitespace: // 跳过水平空白的标签
    // 标记后续Token前有空白
    Result.setFlag(Token::LeadingSpace);
    // 跳过空白(保留空白模式下返回空白Token)
    if (SkipWhitespace(Result, CurPtr, TokAtPhysicalStartOfLine))
      return true; // 保留空白模式：返回空白Token

  SkipIgnoredUnits: // 跳过“忽略单元”(空白+注释)的标签
    // 重置CurPtr到当前缓冲区位置(处理注释后需重新检查空白)
    CurPtr = BufferPtr;

    // 快速跳过注释(避免进入大switch分支, 提升性能)
    // 场景1：// 行注释(需满足：允许行注释 + 非传统C模式/C++模式)
    if (CurPtr[0] == '/' && CurPtr[1] == '/' && !inKeepCommentMode() &&
        LineComment && (LangOpts.CPlusPlus || !LangOpts.TraditionalCPP)) {
      // 跳过行注释(保留注释模式下返回注释Token)
      if (SkipLineComment(Result, CurPtr+2, TokAtPhysicalStartOfLine))
        return true; // 保留注释模式：返回注释Token
      // 注释后可能还有空白, 跳回继续处理
      goto SkipIgnoredUnits;
    }
    // 场景2：/* */ 块注释(无需区分语言模式, C/C++通用)
    else if (CurPtr[0] == '/' && CurPtr[1] == '*' && !inKeepCommentMode()) {
      // 跳过块注释(保留注释模式下返回注释Token)
      if (SkipBlockComment(Result, CurPtr+2, TokAtPhysicalStartOfLine))
        return true; // 保留注释模式：返回注释Token
      // 注释后重试
      goto LexNextToken;
    }
    // 场景3：仍有水平空白, 跳回继续跳过
    else if (isHorizontalWhitespace(*CurPtr)) {
      goto SkipHorizontalWhitespace;
    }

    // 仅解析到空白/注释, 跳转到入口重试
    goto LexNextToken;

  // 处理数字常量(C99 6.4.4.1：整数常量；C99 6.4.4.2：浮点数常量)
  case '0': case '1': case '2': case '3': case '4':
  case '5': case '6': case '7': case '8': case '9':
    // 通知MIOpt(机器指令优化模块)：已读取非空白/非注释Token
    MIOpt.ReadToken();
    // 调用专门函数解析数字常量(整数/浮点数), 返回分析结果
    return LexNumericConstant(Result, CurPtr);

  // 处理以'u'开头的Token：标识符(如"uber")或UTF-16/UTF-8常量(C11/C++11+)
  case 'u':
    MIOpt.ReadToken(); // 通知MIOpt：读取有效Token

    // 仅C11/C++11及以上支持UTF常量
    if (LangOpts.CPlusPlus11 || LangOpts.C11) {
      // 读取'u'后的下一个字符, 获取字符长度(处理多字节)
      Char = getCharAndSize(CurPtr, SizeTmp);

      // 场景1：UTF-16字符串常量(如u"hello")
      if (Char == '"')
        return LexStringLiteral(Result, 
                                ConsumeChar(CurPtr, SizeTmp, Result), // 消费'"'
                                tok::utf16_string_literal); // Token类型：UTF-16字符串

      // 场景2：UTF-16字符常量(如u'x')
      if (Char == '\'')
        return LexCharConstant(Result, 
                               ConsumeChar(CurPtr, SizeTmp, Result), // 消费"'"
                               tok::utf16_char_constant); // Token类型：UTF-16字符

      // 场景3：UTF-16原始字符串常量(如uR"(hello)", C++11+)
      if (Char == 'R' && LangOpts.CPlusPlus11 &&
          getCharAndSize(CurPtr + SizeTmp, SizeTmp2) == '"')
        return LexRawStringLiteral(Result,
                               // 消费'u'后的'R'和'"'
                               ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result),
                                           SizeTmp2, Result),
                               tok::utf16_string_literal); // Token类型：UTF-16原始字符串

      // 场景4：UTF-8常量(如u8"hello", C++17/C2x+)
      if (Char == '8') {
        char Char2 = getCharAndSize(CurPtr + SizeTmp, SizeTmp2);

        // 子场景4.1：UTF-8字符串常量(u8"hello")
        if (Char2 == '"')
          return LexStringLiteral(Result,
                               // 消费'u'后的'8'和'"'
                               ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result),
                                           SizeTmp2, Result),
                               tok::utf8_string_literal);

        // 子场景4.2：UTF-8字符常量(u8'x', C++17/C2x+)
        if (Char2 == '\'' && (LangOpts.CPlusPlus17 || LangOpts.C2x))
          return LexCharConstant(
              Result, 
              // 消费'u'后的'8'和'"'
              ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result),
                          SizeTmp2, Result),
              tok::utf8_char_constant);

        // 子场景4.3：UTF-8原始字符串常量(u8R"(hello)", C++11+)
        if (Char2 == 'R' && LangOpts.CPlusPlus11) {
          unsigned SizeTmp3;
          char Char3 = getCharAndSize(CurPtr + SizeTmp + SizeTmp2, SizeTmp3);
          if (Char3 == '"') {
            return LexRawStringLiteral(Result,
                   // 消费'u'后的'8'、'R'和'"'
                   ConsumeChar(ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result),
                                           SizeTmp2, Result),
                               SizeTmp3, Result),
                   tok::utf8_string_literal);
          }
        }
      }
    }

    // 若不是UTF常量, 将'u'视为标识符开头(如"uint"), 继续解析标识符
    return LexIdentifierContinue(Result, CurPtr);

  // 处理以'U'开头的Token：标识符(如"Uber")或UTF-32常量(C11/C++11+)
  case 'U':
    MIOpt.ReadToken(); // 通知MIOpt：读取有效Token

    if (LangOpts.CPlusPlus11 || LangOpts.C11) {
      Char = getCharAndSize(CurPtr, SizeTmp);

      // 场景1：UTF-32字符串常量(如U"hello")
      if (Char == '"')
        return LexStringLiteral(Result, 
                                ConsumeChar(CurPtr, SizeTmp, Result),
                                tok::utf32_string_literal);

      // 场景2：UTF-32字符常量(如U'x')
      if (Char == '\'')
        return LexCharConstant(Result, 
                               ConsumeChar(CurPtr, SizeTmp, Result),
                               tok::utf32_char_constant);

      // 场景3：UTF-32原始字符串常量(如UR"(hello)", C++11+)
      if (Char == 'R' && LangOpts.CPlusPlus11 &&
          getCharAndSize(CurPtr + SizeTmp, SizeTmp2) == '"')
        return LexRawStringLiteral(Result,
                               ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result),
                                           SizeTmp2, Result),
                               tok::utf32_string_literal);
    }

    // 不是UTF常量时, 视为标识符开头(如"Uint")
    return LexIdentifierContinue(Result, CurPtr);

  // 处理以'R'开头的Token：标识符(如"Result")或C++11+原始字符串常量(如R"(hello)")
  case 'R':
    MIOpt.ReadToken(); // 通知MIOpt：读取有效Token

    if (LangOpts.CPlusPlus11) { // 仅C++11及以上支持原始字符串
      Char = getCharAndSize(CurPtr, SizeTmp);
      // 原始字符串常量(如R"(hello)")
      if (Char == '"')
        return LexRawStringLiteral(Result,
                                   ConsumeChar(CurPtr, SizeTmp, Result), // 消费'"'
                                   tok::string_literal); // Token类型：普通原始字符串
    }

    // 不是原始字符串时, 视为标识符开头(如"Rename")
    return LexIdentifierContinue(Result, CurPtr);

  // 处理以'L'开头的Token：标识符(如"Loony")或宽字符常量(如L'x'、L"xyz")
  case 'L':
    MIOpt.ReadToken(); // 通知MIOpt：读取有效Token
    Char = getCharAndSize(CurPtr, SizeTmp);

    // 场景1：宽字符串常量(如L"中文")
    if (Char == '"')
      return LexStringLiteral(Result, 
                              ConsumeChar(CurPtr, SizeTmp, Result),
                              tok::wide_string_literal); // Token类型：宽字符串

    // 场景2：宽字符原始字符串(如LR"(中文)", C++11+)
    if (LangOpts.CPlusPlus11 && Char == 'R' &&
        getCharAndSize(CurPtr + SizeTmp, SizeTmp2) == '"')
      return LexRawStringLiteral(Result,
                               ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result),
                                           SizeTmp2, Result),
                               tok::wide_string_literal);

    // 场景3：宽字符常量(如L'中')
    if (Char == '\'')
      return LexCharConstant(Result, 
                             ConsumeChar(CurPtr, SizeTmp, Result),
                             tok::wide_char_constant); // Token类型：宽字符

    // 不是宽常量时, 视为标识符开头(如"Lazy"), 穿透到标识符分支
    [[fallthrough]];

  // 处理标识符(C99 6.4.2)：字母、下划线开头, 后续可跟字母、数字、下划线
  case 'A': case 'B': case 'C': case 'D': case 'E': case 'F': case 'G':
  case 'H': case 'I': case 'J': case 'K':    /*'L'已处理*/case 'M': case 'N':
  case 'O': case 'P': case 'Q':    /*'R'已处理*/case 'S': case 'T':    /*'U'已处理*/
  case 'V': case 'W': case 'X': case 'Y': case 'Z':
  case 'a': case 'b': case 'c': case 'd': case 'e': case 'f': case 'g':
  case 'h': case 'i': case 'j': case 'k': case 'l': case 'm': case 'n':
  case 'o': case 'p': case 'q': case 'r': case 's': case 't':    /*'u'已处理*/
  case 'v': case 'w': case 'x': case 'y': case 'z':
  case '_':
    MIOpt.ReadToken(); // 通知MIOpt：读取有效Token
    // 继续解析标识符剩余部分(如"user123"中的"ser123")
    return LexIdentifierContinue(Result, CurPtr);

  // 处理标识符中的'$'(如PHP风格变量$var, 需开启DollarIdents选项)
  case '$':
    if (LangOpts.DollarIdents) { // 仅开启“允许$在标识符中”时处理
      if (!isLexingRawMode())
        // 提示“扩展特性：标识符中包含$”
        Diag(CurPtr-1, diag::ext_dollar_in_identifier);
      MIOpt.ReadToken();
      // 将$视为标识符开头, 继续解析(如"$user")
      return LexIdentifierContinue(Result, CurPtr);
    }

    // 未开启选项时, $视为无效字符
    Kind = tok::unknown;
    break;

  // 处理字符常量(C99 6.4.4)：如'a'、'\n'
  case '\'':
    MIOpt.ReadToken();
    // 调用专门函数解析字符常量
    return LexCharConstant(Result, CurPtr, tok::char_constant);

  // 处理字符串常量(C99 6.4.5)：如"hello"
  case '"':
    MIOpt.ReadToken();
    // 若正在解析#include的文件名(如#include <stdio.h>), 返回头文件名Token
    if (ParsingFilename)
      return LexAngledStringLiteral(Result, CurPtr);
    // 否则返回普通字符串Token
    return LexStringLiteral(Result, CurPtr, tok::string_literal);

  // 处理标点符号(C99 6.4.6)：单个或组合符号(如+、++、+=)
  case '?':
    Kind = tok::question; // 问号(三元运算符的'?')
    break;
  case '[':
    Kind = tok::l_square; // 左方括号('[')
    break;
  case ']':
    Kind = tok::r_square; // 右方括号(']')
    break;
  case '(':
    Kind = tok::l_paren; // 左圆括号('(')
    break;
  case ')':
    Kind = tok::r_paren; // 右圆括号(')')
    break;
  case '{':
    Kind = tok::l_brace; // 左花括号('{')
    break;
  case '}':
    Kind = tok::r_brace; // 右花括号('}')
    break;
  case '.': // 处理点号(.)、省略号(...)、指针成员访问(.*, C++)
    Char = getCharAndSize(CurPtr, SizeTmp);
    // 场景1：浮点数开头(如.123), 跳转解析数字常量
    if (Char >= '0' && Char <= '9') {
      MIOpt.ReadToken();
      return LexNumericConstant(Result, ConsumeChar(CurPtr, SizeTmp, Result));
    }
    // 场景2：C++指针成员访问(.*)
    else if (LangOpts.CPlusPlus && Char == '*') {
      Kind = tok::periodstar; // Token类型：.*
      CurPtr += SizeTmp; // 消费'*'
    }
    // 场景3：省略号(...)
    else if (Char == '.' && getCharAndSize(CurPtr+SizeTmp, SizeTmp2) == '.') {
      Kind = tok::ellipsis; // Token类型：...
      // 消费后两个'.'
      CurPtr = ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result), SizeTmp2, Result);
    }
    // 场景4：普通点号(如结构体成员访问a.b)
    else {
      Kind = tok::period; // Token类型：.
    }
    break;
  case '&': // 处理&、&&、&=
    Char = getCharAndSize(CurPtr, SizeTmp);
    if (Char == '&') { // 逻辑与(&&)
      Kind = tok::ampamp;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费第二个&
    } else if (Char == '=') { // 按位与赋值(&=)
      Kind = tok::ampequal;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
    } else { // 取地址/按位与(&)
      Kind = tok::amp;
    }
    break;
  case '*': // 处理*、*=
    if (getCharAndSize(CurPtr, SizeTmp) == '=') { // 乘法赋值(*=)
      Kind = tok::starequal;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
    } else { // 指针解引用/乘法(*)
      Kind = tok::star;
    }
    break;
  case '+': // 处理+、++、+=
    Char = getCharAndSize(CurPtr, SizeTmp);
    if (Char == '+') { // 自增(++)
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费第二个+
      Kind = tok::plusplus;
    } else if (Char == '=') { // 加法赋值(+=)
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
      Kind = tok::plusequal;
    } else { // 加法(+)
      Kind = tok::plus;
    }
    break;
  case '-': // 处理-、--、->、->*、-=
    Char = getCharAndSize(CurPtr, SizeTmp);
    if (Char == '-') { // 自减(--)
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费第二个-
      Kind = tok::minusminus;
    }
    // C++成员指针访问(->*)
    else if (Char == '>' && LangOpts.CPlusPlus && getCharAndSize(CurPtr+SizeTmp, SizeTmp2) == '*') {
      // 消费'>'和'*'
      CurPtr = ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result), SizeTmp2, Result);
      Kind = tok::arrowstar; // Token类型：->*
    }
    else if (Char == '>') { // 指针成员访问(->)
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费>
      Kind = tok::arrow;
    }
    else if (Char == '=') { // 减法赋值(-=)
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
      Kind = tok::minusequal;
    } else { // 减法/负号(-)
      Kind = tok::minus;
    }
    break;
  case '~': // 按位取反(~)
    Kind = tok::tilde;
    break;
  case '!': // 处理!、!=
    if (getCharAndSize(CurPtr, SizeTmp) == '=') { // 不等于(!=)
      Kind = tok::exclaimequal;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
    } else { // 逻辑非(!)
      Kind = tok::exclaim;
    }
    break;
  case '/': // 处理/、/=、//注释、/*注释
    Char = getCharAndSize(CurPtr, SizeTmp);
    if (Char == '/') { // // 行注释
      // 注释处理逻辑：即使关闭行注释(如C89), 仍需特殊处理避免歧义
      // 例："foo //**/ bar"在C89中解析为"foo / bar", C++中解析为"foo"
      bool TreatAsComment = LineComment && (LangOpts.CPlusPlus || !LangOpts.TraditionalCPP);
      // 非预处理输出模式下, 若//后是*, 不视为注释(避免C89歧义)
      if (!TreatAsComment && !(PP && PP->isPreprocessedOutput()))
        TreatAsComment = getCharAndSize(CurPtr+SizeTmp, SizeTmp2) != '*';

      if (TreatAsComment) {
        // 跳过行注释(保留注释模式下返回注释Token)
        if (SkipLineComment(Result, ConsumeChar(CurPtr, SizeTmp, Result), TokAtPhysicalStartOfLine))
          return true;
        // 注释后可能有空白, 跳转到快速处理
        goto SkipIgnoredUnits;
      }
    }
    // /* 块注释
    else if (Char == '*') {
      if (SkipBlockComment(Result, ConsumeChar(CurPtr, SizeTmp, Result), TokAtPhysicalStartOfLine))
        return true; // 保留注释模式：返回注释Token
      goto LexNextToken;
    }
    // 除法赋值(/=)
    else if (Char == '=') {
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
      Kind = tok::slashequal;
    }
    // 除法(/)
    else {
      Kind = tok::slash;
    }
    break;
  case '%': // 处理%、%=、二义字符(如%>->}、%:->#, 需开启Digraphs)
    Char = getCharAndSize(CurPtr, SizeTmp);
    if (Char == '=') { // 取模赋值(%=)
      Kind = tok::percentequal;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
    }
    // 二义字符：%> 等价于 }(需开启Digraphs)
    else if (LangOpts.Digraphs && Char == '>') {
      Kind = tok::r_brace;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费>
    }
    // 二义字符：%: 等价于 #, %:%: 等价于 ##(需开启Digraphs)
    else if (LangOpts.Digraphs && Char == ':') {
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费:
      Char = getCharAndSize(CurPtr, SizeTmp);
      // %:%: 等价于 ##(预处理连接符)
      if (Char == '%' && getCharAndSize(CurPtr+SizeTmp, SizeTmp2) == ':') {
        Kind = tok::hashhash;
        CurPtr = ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result), SizeTmp2, Result);
      }
      // 微软扩展：%:@ 等价于 #@(字符化操作符)
      else if (Char == '@' && LangOpts.MicrosoftExt) {
        CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费@
        if (!isLexingRawMode())
          Diag(BufferPtr, diag::ext_charize_microsoft); // 提示微软扩展
        Kind = tok::hashat;
      }
      // %: 等价于 #(预处理指令开头)
      else {
        // 若#在句首, 视为预处理指令(如#define), 跳转处理指令
        if (TokAtPhysicalStartOfLine && !LexingRawMode && !Is_PragmaLexer)
          goto HandleDirective;
        Kind = tok::hash;
      }
    }
    // 取模(%)
    else {
      Kind = tok::percent;
    }
    break;
  case '<': // 处理<、<<、<<=、<=、太空船运算符(<=>, C++20)、二义字符(<:->[、<%->{)
    Char = getCharAndSize(CurPtr, SizeTmp);
    // 正在解析#include的尖括号文件名(如#include <stdio.h>)
    if (ParsingFilename)
      return LexAngledStringLiteral(Result, CurPtr);
    // 左移(<<)
    else if (Char == '<') {
      char After = getCharAndSize(CurPtr+SizeTmp, SizeTmp2);
      // 左移赋值(<<=)
      if (After == '=') {
        Kind = tok::lesslessequal;
        CurPtr = ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result), SizeTmp2, Result);
      }
      // 版本控制冲突标记(如<<<<<<<), 跳过处理
      else if (After == '<' && (IsStartOfConflictMarker(CurPtr-1) || HandleEndOfConflictMarker(CurPtr-1)))
        goto LexNextToken;
      // CUDA扩展：三左移(<<<, 内核调用语法)
      else if (LangOpts.CUDA && After == '<') {
        Kind = tok::lesslessless;
        CurPtr = ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result), SizeTmp2, Result);
      }
      // 普通左移(<<)
      else {
        CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费第二个<
        Kind = tok::lessless;
      }
    }
    // 小于等于(<=)或太空船运算符(<=>, C++20)
    else if (Char == '=') {
      char After = getCharAndSize(CurPtr+SizeTmp, SizeTmp2);
      // C++20太空船运算符(<=>)
      if (After == '>' && LangOpts.CPlusPlus20) {
        if (!isLexingRawMode())
          Diag(BufferPtr, diag::warn_cxx17_compat_spaceship); // 提示C++17兼容性
        CurPtr = ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result), SizeTmp2, Result);
        Kind = tok::spaceship;
        break;
      }
      // C++<=17中, <=后接>需提示加空格(避免歧义)
      if (LangOpts.CPlusPlus && !isLexingRawMode() && After == '>')
        Diag(BufferPtr, diag::warn_cxx20_compat_spaceship)
          << FixItHint::CreateInsertion(getSourceLocation(CurPtr + SizeTmp, SizeTmp2), " ");
      // 普通小于等于(<=)
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
      Kind = tok::lessequal;
    }
    // 二义字符：<: 等价于 [(需开启Digraphs)
    else if (LangOpts.Digraphs && Char == ':') {
      // C++11特殊情况：<::后非:或>时, <视为普通字符(避免模板歧义)
      if (LangOpts.CPlusPlus11 && getCharAndSize(CurPtr + SizeTmp, SizeTmp2) == ':') {
        unsigned SizeTmp3;
        char After = getCharAndSize(CurPtr + SizeTmp + SizeTmp2, SizeTmp3);
        if (After != ':' && After != '>') {
          Kind = tok::less;
          if (!isLexingRawMode())
            Diag(BufferPtr, diag::warn_cxx98_compat_less_colon_colon); // 提示C++98兼容性
          break;
        }
      }
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费:
      Kind = tok::l_square;
    }
    // 二义字符：<% 等价于 {(需开启Digraphs)
    else if (LangOpts.Digraphs && Char == '%') {
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费%
      Kind = tok::l_brace;
    }
    // 编辑器占位符(如<#name#>)
    else if (Char == '#' && SizeTmp == 1 && lexEditorPlaceholder(Result, CurPtr)) {
      return true;
    }
    // 普通小于(<)
    else {
      Kind = tok::less;
    }
    break;
  case '>': // 处理>、>>、>>=、>=
    Char = getCharAndSize(CurPtr, SizeTmp);
    // 大于等于(>=)
    if (Char == '=') {
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
      Kind = tok::greaterequal;
    }
    // 右移(>>)
    else if (Char == '>') {
      char After = getCharAndSize(CurPtr+SizeTmp, SizeTmp2);
      // 右移赋值(>>=)
      if (After == '=') {
        CurPtr = ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result), SizeTmp2, Result);
        Kind = tok::greatergreaterequal;
      }
      // 版本控制冲突标记(如>>>>>>>), 跳过处理
      else if (After == '>' && (IsStartOfConflictMarker(CurPtr-1) || HandleEndOfConflictMarker(CurPtr-1)))
        goto LexNextToken;
      // CUDA扩展：三右移(>>>)
      else if (LangOpts.CUDA && After == '>') {
        Kind = tok::greatergreatergreater;
        CurPtr = ConsumeChar(ConsumeChar(CurPtr, SizeTmp, Result), SizeTmp2, Result);
      }
      // 普通右移(>>)
      else {
        CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费第二个>
        Kind = tok::greatergreater;
      }
    }
    // 普通大于(>)
    else {
      Kind = tok::greater;
    }
    break;
  case '^': // 处理^、^=、OpenCL扩展^^
    Char = getCharAndSize(CurPtr, SizeTmp);
    // 按位异或赋值(^=)
    if (Char == '=') {
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
      Kind = tok::caretequal;
    }
    // OpenCL扩展：逻辑异或(^^)
    else if (LangOpts.OpenCL && Char == '^') {
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费第二个^
      Kind = tok::caretcaret;
    }
    // 按位异或(^)
    else {
      Kind = tok::caret;
    }
    break;
  case '|': // 处理|、||、|=
    Char = getCharAndSize(CurPtr, SizeTmp);
    // 按位或赋值(|=)
    if (Char == '=') {
      Kind = tok::pipeequal;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费=
    }
    // 逻辑或(||)
    else if (Char == '|') {
      // 版本控制冲突标记(如|||||||), 跳过处理
      if (CurPtr[1] == '|' && HandleEndOfConflictMarker(CurPtr-1))
        goto LexNextToken;
      Kind = tok::pipepipe;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费第二个|
    }
    // 按位或(|)
    else {
      Kind = tok::pipe;
    }
    break;
  case ':': // 处理:、::、二义字符(:->])
    Char = getCharAndSize(CurPtr, SizeTmp);
    // 二义字符：:> 等价于 ](需开启Digraphs)
    if (LangOpts.Digraphs && Char == '>') {
      Kind = tok::r_square;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费>
    }
    // 作用域解析符(::, C++)
    else if (Char == ':') {
      Kind = tok::coloncolon;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费第二个:
    }
    // 普通冒号(如三元运算符a?b:c、标签label:)
    else {
      Kind = tok::colon;
    }
    break;
  case ';': // 分号(语句结束符)
    Kind = tok::semi;
    break;
  case '=': // 处理=、==
    Char = getCharAndSize(CurPtr, SizeTmp);
    if (Char == '=') {
      // 版本控制冲突标记(如====), 跳过处理
      if (CurPtr[1] == '=' && HandleEndOfConflictMarker(CurPtr-1))
        goto LexNextToken;
      // 等于(==)
      Kind = tok::equalequal;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费第二个=
    }
    // 赋值(=)
    else {
      Kind = tok::equal;
    }
    break;
  case ',': // 逗号(分隔符, 如函数参数a,b)
    Kind = tok::comma;
    break;
  case '#': // 处理#、##、微软扩展#@、预处理指令开头
    Char = getCharAndSize(CurPtr, SizeTmp);
    // 预处理连接符(##)
    if (Char == '#') {
      Kind = tok::hashhash;
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费第二个#
    }
    // 微软扩展：#@ 字符化操作符(如#define CHAR(x) #@x)
    else if (Char == '@' && LangOpts.MicrosoftExt) {
      Kind = tok::hashat;
      if (!isLexingRawMode())
        Diag(BufferPtr, diag::ext_charize_microsoft); // 提示微软扩展
      CurPtr = ConsumeChar(CurPtr, SizeTmp, Result); // 消费@
    }
    // #在句首, 视为预处理指令(如#define、#include)
    else {
      if (TokAtPhysicalStartOfLine && !LexingRawMode && !Is_PragmaLexer)
        goto HandleDirective; // 跳转处理预处理指令
      Kind = tok::hash; // 普通#(如字符串化操作符#define STR(x) #x)
    }
    break;

  case '@': // Objective-C扩展：@符号(如@interface、@selector)
    if (CurPtr[-1] == '@' && LangOpts.ObjC)
      Kind = tok::at; // 仅Objective-C模式下识别为@Token
    else
      Kind = tok::unknown; // 其他模式下视为无效字符
    break;

  // 处理通用字符名(UCN, 如\u0061表示'a', C99 6.4.3、C++11 [lex.charset]p2)
  case '\\':
    // 汇编预处理模式下不处理UCN
    if (!LangOpts.AsmPreprocessor) {
      // 尝试读取UCN(如\u0061或\U00000061)
      if (uint32_t CodePoint = tryReadUCN(CurPtr, BufferPtr, &Result)) {
        // 若UCN是Unicode空白字符, 跳过处理
        if (CheckUnicodeWhitespace(Result, CodePoint, CurPtr)) {
          if (SkipWhitespace(Result, CurPtr, TokAtPhysicalStartOfLine))
            return true;
          goto LexNextToken;
        }
        // 若UCN是标识符字符, 继续解析标识符(如\u0066oo等价于foo)
        return LexUnicodeIdentifierStart(Result, CodePoint, CurPtr);
      }
    }
    // 无法解析为UCN时, 视为无效字符
    Kind = tok::unknown;
    break;

  // 处理默认情况：非ASCII字符或未匹配的字符
  default: {
    // ASCII字符但未匹配任何分支, 视为无效字符
    if (isASCII(Char)) {
      Kind = tok::unknown;
      break;
    }

    // 处理非ASCII字符(尝试解析为UTF-8)
    llvm::UTF32 CodePoint;
    // 回退CurPtr(因之前已调用getAndAdvanceChar推进指针)
    --CurPtr;
    // 严格解析UTF-8序列, 转换为Unicode码点
    llvm::ConversionResult Status =
        llvm::convertUTF8Sequence((const llvm::UTF8 **)&CurPtr,
                                  (const llvm::UTF8 *)BufferEnd,
                                  &CodePoint,
                                  llvm::strictConversion);
    // 解析成功：判断是否为Unicode空白或标识符字符
    if (Status == llvm::conversionOK) {
      // 是Unicode空白：跳过处理
      if (CheckUnicodeWhitespace(Result, CodePoint, CurPtr)) {
        if (SkipWhitespace(Result, CurPtr, TokAtPhysicalStartOfLine))
          return true;
        goto LexNextToken;
      }
      // 是Unicode标识符字符：继续解析标识符
      return LexUnicodeIdentifierStart(Result, CodePoint, CurPtr);
    }

    // 解析失败：原始模式/预处理指令/预处理输出模式下, 视为无效字符
    if (isLexingRawMode() || ParsingPreprocessorDirective || PP->isPreprocessedOutput()) {
      ++CurPtr;
      Kind = tok::unknown;
      break;
    }

    // 其他模式下：输出无效UTF-8诊断, 丢弃该字符后重试
    Diag(CurPtr, diag::err_invalid_utf8); // 提示“无效的UTF-8字符”
    BufferPtr = CurPtr+1; // 跳过无效字符
    goto LexNextToken;
  }
  }

  // 通知MIOpt：已读取非空白/非注释Token(仅非跳转分支执行)
  MIOpt.ReadToken();

  // 构造最终Token：更新Token的位置、类型, 同步缓冲区指针
  FormTokenWithChars(Result, CurPtr, Kind);
  return true; // 词法分析成功, 返回true

// 处理预处理指令(如#define、#include)的标签
HandleDirective:
  // 构造#Token, 调用预处理器处理指令
  FormTokenWithChars(Result, CurPtr, tok::hash);
  PP->HandleDirective(Result);

  // 若模块加载发生致命错误, 终止解析
  if (PP->hadModuleLoaderFatalFailure())
    return true;

  // 指令处理完成, 返回false表示需重新初始化词法分析状态
  return false;

// 词法分析重试标签：清除残留标记后跳回入口
LexNextToken:
  Result.clearFlag(Token::NeedsCleaning); // 清除“需清理”标记
  goto LexStart;
}
```
可以看到为了兼容 c, c++ 的各个版本, 写的是又臭又长, 不在一一解读, 感兴趣可以自行查看, 为了性能有的地方还是做的很巧妙的.

