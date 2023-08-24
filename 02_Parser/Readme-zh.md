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

The code in `tree.c` has the functions to build ASTs. The most
general function, `mkastnode()`, takes values for all four
fields in an AST node. It allocates the node, populates the field
values and returns a pointer to the node:

```c
// Build and return a generic AST node
struct ASTnode *mkastnode(int op, struct ASTnode *left,
                          struct ASTnode *right, int intvalue) {
  struct ASTnode *n;

  // Malloc a new ASTnode
  n = (struct ASTnode *) malloc(sizeof(struct ASTnode));
  if (n == NULL) {
    fprintf(stderr, "Unable to malloc in mkastnode()\n");
    exit(1);
  }
  // Copy in the field values and return it
  n->op = op;
  n->left = left;
  n->right = right;
  n->intvalue = intvalue;
  return (n);
}
```

Given this, we can write more specific functions that make a leaf AST
node (i.e. one with no children), and make an AST node with a single child:

```c
// Make an AST leaf node
struct ASTnode *mkastleaf(int op, int intvalue) {
  return (mkastnode(op, NULL, NULL, intvalue));
}

// Make a unary AST node: only one child
struct ASTnode *mkastunary(int op, struct ASTnode *left, int intvalue) {
  return (mkastnode(op, left, NULL, intvalue));
```

## Purpose of the AST

We are going to use an AST to store each expression that we recognise
so that, later on, we can traverse it recursively to calculate the
final value of the expression. We do want to deal with the precedence of the 
maths operators. Here is an example.

Consider the expression `2 * 3 + 4 * 5`. Now, multiplication has higher
precedence that addition. Therefore, we want to *bind* the multiplication
operands together and perform these operations before we do the addition.

If we generated the AST tree to look like this:

```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

then, when traversing the tree, we would perform `2*3` first, then `4*5`.
Once we have these results, we can then pass them up to the root of the
tree to perform the addition.

## A Naive Expression Parser

Now, we could re-use the token values from our scanner as the AST node
operation values, but I like to keep the concept of tokens and AST nodes
separate. So, to start with, I'm going to have a function to map
the token values into AST node operation values. This, along with the
rest of the parser, is in `expr.c`:

```c
// Convert a token into an AST operation.
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

The default statement in the switch statement fires when we can't convert
the given token into an AST node type. It's going to form part of the
syntax checking in our parser.

We need a function to check that the next token is an integer literal,
and to build an AST node to hold the literal value. Here it is:

```c
// Parse a primary factor and return an
// AST node representing it.
static struct ASTnode *primary(void) {
  struct ASTnode *n;

  // For an INTLIT token, make a leaf AST node for it
  // and scan in the next token. Otherwise, a syntax error
  // for any other token type.
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

This assumes that there is a global variable `Token`, and that it
already has the most recent token scanned in from the input. In
`data.h`:

```c
extern_ struct token    Token;
```

and in `main()`:

```c
  scan(&Token);                 // Get the first token from the input
  n = binexpr();                // Parse the expression in the file
```

Now we can write the code for the parser:

```c
// Return an AST tree whose root is a binary operator
struct ASTnode *binexpr(void) {
  struct ASTnode *n, *left, *right;
  int nodetype;

  // Get the integer literal on the left.
  // Fetch the next token at the same time.
  left = primary();

  // If no tokens left, return just the left node
  if (Token.token == T_EOF)
    return (left);

  // Convert the token into a node type
  nodetype = arithop(Token.token);

  // Get the next token in
  scan(&Token);

  // Recursively get the right-hand tree
  right = binexpr();

  // Now build a tree with both sub-trees
  n = mkastnode(nodetype, left, right, 0);
  return (n);
}
```

Notice that nowhere in this naive parser code is there anything to
deal with different operator precedence. As it stands, the code treats
all operators as having equal precedence. If you follow the code as
it parses the expression `2 * 3 + 4 * 5`, you will see that it
builds this AST:

```
     *
    / \
   2   +
      / \
     3   *
        / \
       4   5
```

This is definitely not correct. It will multiply `4*5` to get 20,
then do `3+20` to get 23 instead of doing `2*3` to get 6.

So why did I do this? I wanted to show you that writing a simple parser
is easy, but getting it to also do the semantic analysis is harder.

## Interpreting the Tree

Now that we have our (incorrect) AST tree, let's write some code to
interpret it. Again, we are going to write recursive code to traverse
the tree. Here's the pseudo-code:

```
interpretTree:
  First, interpret the left-hand sub-tree and get its value
  Then, interpret the right-hand sub-tree and get its value
  Perform the operation in the node at the root of our tree
  on the two sub-tree values, and return this value
```

Going back to the correct AST tree:

```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

the call structure would look like:

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

## Code to Interpret the Tree

This is in `interp.c` and follows the above pseudo-code:

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

Again, the default statement in the switch statement fires when we can't 
interpret the AST node type. It's going to form part of the
sematic checking in our parser.

## Building the Parser

There is some other code here and the, like the call to the interpreter
in `main()`:

```c
  scan(&Token);                 // Get the first token from the input
  n = binexpr();                // Parse the expression in the file
  printf("%d\n", interpretAST(n));      // Calculate the final result
  exit(0);
```

You can now build the parser by doing:

```
$ make
cc -o parser -g expr.c interp.c main.c scan.c tree.c
```

I've provided several input files for you to test the parser on, but 
of course you can create your own. Remember, the calculated results
are incorrect, but the parser should detect input errors like
consecutive numbers, consecutive operators, and a number missing at the end
of the input. I've also added some debugging code to the interpreter so
you can see which AST tree nodes get evaluated in which order:

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
