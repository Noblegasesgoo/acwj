# 第二部分：解析器介绍

在手写编译器的这个部分，我将要介绍解析器的基本概念，正如我在第一部分提到的，解析器的工作就是识别输入元素的语法和结构元素并且确保他们是符合输入语言的语法规范的。

我们已经几个可以被扫描的语言元素了，即我们的标记（tokens）：

 + 四个数字运算符: `*`, `/`, `+` and `-`
 + 具有1或多个数字的十进制整数 `0` .. `9`

现在让我们来定义我们的解析器是如何来识别语言的语法规范。

## BNF：Backus-Naur 形式

如果你开始涉足计算机语言处理领域的话，你将会在某个时候遇到[BNF ](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)的使用。我将仅仅介绍足够 BNF 的语法来表达我们想要识别的语法。

我们想要有一个语法能够表示所有整数数字的数学表达式。以下是该语法的BNF描述：

```BNF
expression: number // number 的定义在末尾
          | expression '*' expression
          | expression '/' expression
          | expression '+' expression
          | expression '-' expression
          ;

number:  T_INTLIT
         ;
```

垂直线将语法中的选项分隔开，因此上述内容表示的是：

  + 一个表达式可以只是一个数字，或者
  + 一个表达式是由一个“*”标记分隔的两个表达式组成，或者
  + 一个表达式是由一个“/”标记分隔的两个表达式组成，或者
  + 一个表达式是由一个“-”标记分隔的两个表达式组成，或者
  + 一个表达式是由一个“+”标记分隔的两个表达式组成，或者
  + 一个数字总是一个 T_INTLIT token。

BNF语法定义的语法很明显是递归的：一个表达式通过引用别的表达式来定义。但是有方法能够阻止递归：当表达式变成一个数字时，它总是一个 T_INTLIT token，因此不再递归。

在 BNF 中，我们称"expression"（表达式）和"number"（数字）为*非终结符*符号，因为他们是由语法规则生成的。然而，T_INTLIT 是一个终结符号因为它不由任何规则定义。（这段话言外之意就是自定义变量的意思吧，因为 T_INTLIT 是我们自己定义的东西，所以它不属于任何规则）。相反，它是语言中已经识别的token。同样地，四个数学运算符标记也是终结符符号。

## 递归下降解析

考虑到我们语言的语法是递归的，尝试使用递归方式进行解析是有意义的。我们需要做的是读入一个标记（token），然后*向前看*（look ahead）到下一个标记。根据下一个标记是什么来决定我们需要从哪条路径来解析输入内容。这样也许需要我们递归调用已经被我们调用过的函数。

在我们的情况下，任何表达式的第一个标记将会是一个数字，也许后面还跟着一个数学运算符。之后可能只有一个单独的数字或者可能是以一个全新的表达式开始。我们该如何递归的解析这样的情况呢？

我们可以编写下面这样的伪代码：

```
function expression() {
  扫描并检查第一个token是不是数字。如果不是的话就报错。
  Scan and check the first token is a number. Error if it's not
  Get the next token 获取下一个token
  如果我们已经到输入内容的最尾端，返回，即基本情况。
  If we have reached the end of the input, return, i.e. base case
  否则就调用 expression() 函数
  Otherwise, call expression()
}
```

让我们以输入为： `2 + 3 - 5 T_EOF` 并且以 `T_EOF`这个token来表示输入结束来运行这个函数。我将要每次调用 `expression()` 函数都进行编号记录。

Let's run this function on the input `2 + 3 - 5 T_EOF` where `T_EOF`
is a token that reflects the end of the input. I will number each
call to `expression()`.

```
expression0:
  Scan in the 2, it's a number
  Get next token, +, which isn't T_EOF
  Call expression()

    expression1:
      Scan in the 3, it's a number
      Get next token, -, which isn't T_EOF
      Call expression()

        expression2:
          Scan in the 5, it's a number
          Get next token, T_EOF, so return from expression2

      return from expression1
  return from expression0
```

是的，这个函数递归成功的解析了输入内容`2 + 3 - 5 T_EOF`。

当然，我们并没有对输入做任何处理，不过这并不是解释器的工作内容。解释器的内容就是识别输入并且提醒一些语法错误。别的功能将会对输入进行*语义分析*也就是理解并执行这个输入的含义。

> 之后，你将会发现实际情况并非是正确的。在很多场景下，
> 将语法分析和语义分析交织在一起使用是有意义的。

## 抽象语法树

做语义分析，我们需要要么能解释已识别的输入内容，要么能将其转换成另一种格式的代码，比如汇编代码。在旅程的这一部分，我们将为输入构建一个解释器。但为了达到这个目标，我们首先将把输入转换成一个[抽象语法树](https://en.wikipedia.org/wiki/Abstract_syntax_tree)，也称为AST。

我强烈推荐你去读一读这篇有关 ASTs的简短解释：

- [Leveling Up One’s Parsing Game With ASTs](https://medium.com/basecs/leveling-up-ones-parsing-game-with-asts-d7a6fc2400ff) by Vaidehi Joshi

它真的写得很好并且真的能够帮助去理解ASTs的目的和结构。“不用担心，等你回来时我一直都在这”，你随时可以点击去看。

我们将在`defs.h`文件中构建并描述每个节点在AST中的结构：

```c
// AST node types AST节点类型
enum {
  A_ADD, A_SUBTRACT, A_MULTIPLY, A_DIVIDE, A_INTLIT
};

// Abstract Syntax Tree structure 抽象语法树（AST）结构
struct ASTnode {
  int op;                               // "Operation" to be performed on this tree 在树上执行的操作
  struct ASTnode *left;                 // Left and right child trees 左右子树
  struct ASTnode *right;
  int intvalue;                         // For A_INTLIT, the integer value 对于A_INTLIT，整数值
};
```

一些AST节点，比如 `op` 值为 `A_ADD` 和 `A_SUBTRACT` 将拥有两个由 `left` 和 `right` 指向的子ASTs节点。稍后，我们会对子树的值进行加减运算。

另一种选择是，具有`op`值为A_INTLIT的AST节点表示一个整数值。它没有子树子节点，只有一个在`intvalue`字段中的值。

## 构建AST节点和树

`tree.c`文件里有构造ASTs的代码。最通用的函数`mkastnode()`接受四个AST节点字段的值。它用来分配节点，填充字段值并且返回节点指针：

```c
// Build and return a generic AST node 构建并返回一个通用的AST节点
struct ASTnode *mkastnode(int op, struct ASTnode *left,
                          struct ASTnode *right, int intvalue) {
  struct ASTnode *n;

  // Malloc a new ASTnode 分配新的AST节点
  n = (struct ASTnode *) malloc(sizeof(struct ASTnode));
  if (n == NULL) {
    fprintf(stderr, "Unable to malloc in mkastnode()\n");
    exit(1);
  }
  // Copy in the field values and return it 复制字段值并返回它
  n->op = op;
  n->left = left;
  n->right = right;
  n->intvalue = intvalue;
  return (n);
}
```

鉴于此，我们可以编写更具体的用于创建一个叶子AST节点的函数（即没有子节点的节点），并且创建一个带有单个子节点的AST节点：

```c
// Make an AST leaf node 创建一个AST叶子节点
struct ASTnode *mkastleaf(int op, int intvalue) {
  return (mkastnode(op, NULL, NULL, intvalue));
}

// Make a unary AST node: only one child 创建一个一元AST节点：只有一个子节点
struct ASTnode *mkastunary(int op, struct ASTnode *left, int intvalue) {
  return (mkastnode(op, left, NULL, intvalue));
```

## AST的目的

我们将使用AST来存储我们识别出的每个表达式。这样之后我们就可以递归的遍历它来计算表达式的最终值。我们希望能处理数学运算符的优先级。下面是一个🌰：

思考一下这个表达式`2 * 3 + 4 * 5`。现在，乘法运算符的优先级高于加法。因此，我们希望在进行加法操作之前将乘法操作数绑定在一起并执行这些操作。

如果我们生成了这样一颗AST树：

```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

然后当我们遍历树史，我们首先执行`2*3`，然后再执行`4*5`。一旦我们有了这些结果，我们就可以将其回传到树的根节点来执行加法。

## 一个简单的表达式解析器

现在，我们可以复用从我们扫描器中的标记值作为AST节点中的运算符值。不过我比较喜欢保持token与AST节点的概念分开来看。所以，首先我们要有一个函数来映射token的值与AST节点操作的值。以上内容以及解析器的其余部分都在 `expr.c`文件中：

```c
// Convert a token into an AST operation. 将token转换为AST操作符
int arithop(int tok) {
  switch (tok) {
    case T_PLUS:
      return (A_ADD);
    case T_MINUS:
      return (A_SUBTRACT);
    case T_STAR:
      return (A_MULTIPLY);
    case T_SLASH:
      return (A_DIVIDE);
    default:
      fprintf(stderr, "unknown token in arithop() on line %d\n", Line);
      exit(1);
  }
}
```

switch语句中的default语句块触发时机是当我们无法将给到的token转换为对应的AST节点类型的情况。这将成为我们解析器中语法检查的一部分。

我们需要一个能检查下一个标记是否是一个整数字面值，并且构建一个持有该字面值的AST节点的函数。以下就是这个函数：

```c
// Parse a primary factor and return an
// AST node representing it. 解析一个主要因子并返回表示它的AST节点。
static struct ASTnode *primary(void) {
  struct ASTnode *n;
	
  // For an INTLIT token, make a leaf AST node for it  对一个INTLIT token，创建一个AST叶子节点
  // and scan in the next token. Otherwise, a syntax error 并且扫描下一个scan，否则，
  // for any other token type. 对于其他任何标记类型，都会产生语法错误。
  switch (Token.token) {
    case T_INTLIT:
      n = mkastleaf(A_INTLIT, Token.intvalue);
      scan(&Token);
      return (n);
    default:
      fprintf(stderr, "syntax error on line %d\n", Line);
      exit(1);
  }
}
```

我们假设 `Token` 是一个全局变量，并且他已经是最近一次从输入内容中扫描到的最新值。 `data.h`:

```c
extern_ struct token    Token;
```

and in `main()`:

```c
  scan(&Token);                 // Get the first token from the input 获取第一个token
  n = binexpr();                // Parse the expression in the file 解析表达式
```

现在我们可以为解析器写代码了：

```c
// Return an AST tree whose root is a binary operator 返回一个以二进制运算符为根节点的AST树
struct ASTnode *binexpr(void) {
  struct ASTnode *n, *left, *right;
  int nodetype;

  // Get the integer literal on the left. 获取左树中的整数字面量的值
  // Fetch the next token at the same time. 的同时获取下一个token
  left = primary();

  // If no tokens left, return just the left node 如果左侧没有tokens，那么就只返回左侧节点
  if (Token.token == T_EOF)
    return (left);

  // Convert the token into a node type 将token转换为对应的节点类型
  nodetype = arithop(Token.token);

  // Get the next token in 获取下一个token
  scan(&Token);

  // Recursively get the right-hand tree 递归获取右树
  right = binexpr();

  // Now build a tree with both sub-trees 现在构建一个包含两个子树的树
  n = mkastnode(nodetype, left, right, 0);
  return (n);
}
```

注意这个简单的解析代码中没有任何处理不同操作符优先级的内容。目前，以上代码将所有操作符都视为同一优先级。如果你跟着代码解析了`2 * 3 + 4 * 5` 表达式，你将会看到这样的一棵树被创建：

```
     *
    / \
   2   +
      / \
     3   *
        / \
       4   5
```

这显然不是正确的。这棵树中`4*5` 将得到 20，之后就开始运行 `3+20` 得到23，替代掉了原本应该的 `2*3` 得到6的过程

所以我为什么这么做？我就是想要给你展示写一个简单的解析器是十分容易的事情，但是同时让其执行语义分析更难。



## 解释树

现在我们有了（不正确的）AST树，让我们编写一些代码来解释它。同样，我们将编写递归代码来遍历树。以下是伪代码：

```
interpretTree:
  First, interpret the left-hand sub-tree and get its value 首先，解释左子树并且获得值
  Then, interpret the right-hand sub-tree and get its value	其次解释右子树并且活得值
  Perform the operation in the node at the root of our tree	执行当前树根节点中的操作符
  on the two sub-tree values, and return this value 来运算左右子树当前的值，并且返回。
```

回到正确的AST树:

```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

他的调用结构应该看起来是这样：

```
interpretTree0(tree with +):
  Call interpretTree1(left tree with *):
     Call interpretTree2(tree with 2):
       No maths operation, just return 2
     Call interpretTree3(tree with 3):
       No maths operation, just return 3
     Perform 2 * 3, return 6

  Call interpretTree1(right tree with *):
     Call interpretTree2(tree with 4):
       No maths operation, just return 4
     Call interpretTree3(tree with 5):
       No maths operation, just return 5
     Perform 4 * 5, return 20

  Perform 6 + 20, return 26
```

## 编写解释树

这在`interp.c`中，遵循上面的伪代码：

```c
// Given an AST, interpret the
// operators in it and return
// a final value.
int interpretAST(struct ASTnode *n) {
  int leftval, rightval;

  // Get the left and right sub-tree values
  if (n->left)
    leftval = interpretAST(n->left);
  if (n->right)
    rightval = interpretAST(n->right);

  switch (n->op) {
    case A_ADD:
      return (leftval + rightval);
    case A_SUBTRACT:
      return (leftval - rightval);
    case A_MULTIPLY:
      return (leftval * rightval);
    case A_DIVIDE:
      return (leftval / rightval);
    case A_INTLIT:
      return (n->intvalue);
    default:
      fprintf(stderr, "Unknown AST operator %d\n", n->op);
      exit(1);
  }
}
```

再次提醒，switch语句块中的默认模块是来处理我们没法解释的AST节点类型。这也是我们语法解析其中的一部分。

## 构建解析器



这里还有一些其他代码，比如在`main()`中调用解释器的部分。

```c
  scan(&Token);                 // Get the first token from the input
  n = binexpr();                // Parse the expression in the file
  printf("%d\n", interpretAST(n));      // Calculate the final result
  exit(0);
```

你可以运行以下命令来构建解析器：

```
$ make
cc -o parser -g expr.c interp.c main.c scan.c tree.c
```

我提供了一些输入文件来供解析器测试，不过你也可以自己去做自己的输入文件来测试。请记住，计算的结果是不正确的。不过解析器应该能够检测到类似连续数字，连续运算符，末尾缺少数字等输入错误。我还在解释器中添加了一些可供调试的方法，所以你可以看到AST树节点遍历的顺序来进行评估：

```
$ cat input01
2 + 3 * 5 - 8 / 3

$ ./parser input01
int 2
int 3
int 5
int 8
int 3
8 / 3
5 - 2
3 * 3
2 + 9
11

$ cat input02
13 -6+  4*
5
       +
08 / 3

$ ./parser input02
int 13
int 6
int 4
int 5
int 8
int 3
8 / 3
5 + 2
4 * 7
6 + 28
13 - 34
-21

$ cat input03
12 34 + -56 * / - - 8 + * 2

$ ./parser input03
unknown token in arithop() on line 1

$ cat input04
23 +
18 -
45.6 * 2
/ 18

$ ./parser input04
Unrecognised character . on line 3

$ cat input05
23 * 456abcdefg

$ ./parser input05
Unrecognised character a on line 1
```

## Conclusion and What's Next

A parser recognises the grammar of the language and checks that the
input to the compiler conforms to this grammar. If it doesn't, the parser
should print out an error message. As our expression grammar is
recursive, we have chosen to write a recursive descent parser to
recognise our expressions.

Right now the parser works, as shown by the above output, but it fails
to get the semantics of the input right. In other words, it doesn't
calculate the correct value of the expressions.

In the next part of our compiler writing journey, we will modify
the parser so that it also does the semantic analysis of the
expressions to get the right maths results. [Next step](../03_Precedence/Readme.md)
