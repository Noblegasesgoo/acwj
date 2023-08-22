# 第一部分： 词法扫描介绍

我们以写一个简单的词法扫描器为手写编译器旅程的开始，如我上面所说的，扫描器的工作是识别输入语言内容的语法元素以及 *tokens*。

我们将从一个只有五个词法元素的语言开始：

- 四个基本运算操作：`*`, `/`, `+`,  `-`。
- 一个或多个数字的十进制整数 `0` .. `9`。

我们扫描的每个 `token` 都将存储在这个结构中（`defs.h`）：

```c
// Token structure
struct token {
    int token;
    int intvalue;
}
```

`token`字段的值来可以是如下这些值之一（`defs.h`）。

```c
// Tokens
enum {
  T_PLUS, T_MINUS, T_STAR, T_SLASH, T_INTLIT
};
```

当 `token` 的值是 `T_INTLIT`（即整数字面量）的时候， `intvalue` 字段将持有我们所扫描到的整数值。

## `scan.c` 中的函数方法

`scan.c`文件中拥有我们词法扫描器的函数方法。我们将从读取输入文件一次读取一个字符。不过有的时候我们在输入流中读取得太靠前，此时就需要“放回”一个字符。 我们还想要追踪当前在哪一行以便我们在调试信息中打印行号。以上所有这些功能都由 `next()` 函数完成：

```c
// Get the next character from the input file.
static int next(void) {
    int c;
    
    if (Putback) {			// 使用输入字符 Use the character put
		c = Putback; 	    // 如果只剩一个就返回 back if there is one
		Putback = 0;		
		return c;
    }
    
    c = fgetc(Intfile);		// 读取输入文件 Read from input file
    if ('\n' == c) { 		
        Line ++;		    // 行数统计变量自增 Increment line count
        return c;
    }
}
```

变量 `Putback` 、`Line` 与我们的输入文件指针一起定义在了 `data.h` 文件中：

```c
extern_ int     Line;
extern_ int     Putback;
extern_ FILE    *Infile;
```

所有C文件都会包含 `extern` 被 `extern_` 替换的代码。但是 `main.c` 文件会移除 `extern_`；因此这些变量将会“属于” `main.c`。

最后，我们怎么将一个字符放入输入流中呢？就像下面这样：

```c
// Put back an unwanted character 将不需要的字符放回输入流中
static void putback(int c) {
  Putback = c;
}
```

## 忽略空白字符

我们需要一个能读取并且能跳过空白字符直至获取到一个非空白字符的函数方法，并且返回这个字符。就像下面这样：

```c
// Skip past input that we don't need to deal with, 跳过我们不需要处理的输入内容，
// i.e. whitespace, newlines. Return the first 即空白字符、换行字符。返回第一个我们需要处理的字符。
// character we do need to deal with.
static int skip(void) {
  int c;

  c = next();
  while (' ' == c || '\t' == c || '\n' == c || '\r' == c || '\f' == c) {
    c = next();
  }
  return (c);
}
```

## 扫描Tokens：`scan()`

所以现在我们可以读取到我们需要的字符了；如果我们读取了一个多余的字符，我们也可以将其放回。我们现在就可以写我们的第一个词法扫描器了：

```c
// Scan and return the next token found in the input. 扫描并且返回输入内容中的下一个token
// Return 1 if token valid, 0 if no tokens left. 如果token有效则返回1，如果没有剩余token则返回0
int scan(struct token *t) {
  int c;

  // Skip whitespace 跳过空白字符
  c = skip();

  // Determine the token based on 根据输入字符
  // the input character 确定token
  switch (c) {
  case EOF:
    return (0);
  case '+':
    t->token = T_PLUS;
    break;
  case '-':
    t->token = T_MINUS;
    break;
  case '*':
    t->token = T_STAR;
    break;
  case '/':
    t->token = T_SLASH;
    break;
  default:
    // More here soon 之后会有更多类型
  }

  // We found a token 我们找到了token
  return (1);
}
```

以上就是对于简单字符`tokens`标记的内容了：每一个被识别的字符，都会变成一个`token`。你也许会说：为什么不直接将被识别的字符放入`struct token`中呢？我们之后将要对例如 `==` 以及`if` `where`关键字这样的多字符tokens进行标记时会给你答案。因此，有一个枚举的标记值列表会让事情变得更加简便。

## 整数字面值的值

事实上，我们不得不去面对需要去标记类似`3827`和`87731`这样整数字面值的情况。以下是`switch`段中缺少的`default`代码：

```c
  default:

    // If it's a digit, scan the 如果是数字，就扫描他的整数字面量
    // literal integer value in
    if (isdigit(c)) {
      t->intvalue = scanint(c);
      t->token = T_INTLIT;
      break;
    }

    printf("Unrecognised character %c on line %d\n", c, Line);
    exit(1);
```

一旦我们遇到一个十进制数字字符，我们调用名为`scanint()`的辅助函数，并将第一个字符作为参数传递给它。该函数会返回扫描到的整数值。为了实现这一点，函数将会读取每一个输入字符，检查是否是合法的数字并且构建最终数字，以下是代码：

```c
// Scan and return an integer literal 从输入文件中扫描并返回整数字面量
// value from the input file. Store 将值作为字符串存储在Text中
// the value as a string in Text.
static int scanint(int c) {
  int k, val = 0;

  // Convert each character into an int value 转换每一个字符为int值
  while ((k = chrpos("0123456789", c)) >= 0) {
    val = val * 10 + k;
    c = next();
  }

  // We hit a non-integer character, put it back. 当我们遇到一个非整数字符是，将其放回
  putback(c);
  return val;
}
```

我们以为0的`val`值开始，每次我从`0，9`的范围中获取一个字符串我们就用 `chrpos()`函数来将其转换为`int`值。我们将`val`变大十倍，并且将这个新数字添加到其中。

举个🌰，如果我们有字符集  `3`, `2`, `8`, 我们要将其转换成`328`我们这么做：

- `val = 0 * 10 + 3`，即3
- `val = 3 * 10 + 2`，即32
- `val = 32 * 10 + 8`，即328

在最后，你有没有注意到一个名为 `putback(c)` 的函数（您注意到了对`putback(c)`的调用吗？）？在这里我们找到一个不属于十进制数字的字符。我们不能简单的将他丢弃，但幸运的是我们可以将其放回到输出流中以便之后使用。

你也许会询问这个点：为什么不用 ASCII 码的 '0' 到`c`简单的减去该字符从而获得该字符的整形值呢？稍后我们会使用`chrpos("0123456789abcdef")` 函数将十六进制数字进行转换便是答案。

以下是 `chrpos()` 的代码：

```c
// Return the position of character c 返回字符c位于字符串s中的位置，找不到的话就返回-1
// in string s, or -1 if c not found
static int chrpos(char *s, int c) {
  char *p;

  p = strchr(s, c);
  return (p ? p - s : -1);
}
```

这些暂时就是目前词法分析扫描器中的代码内容了。

## 让词法分析扫描器开始工作

下面`main.c` 的代码将上述内容中的词法分析扫描器投入使用。该函数打开一个文件，然后扫描其中的tokens：

```c
void main(int argc, char *argv[]) {
  ...
  init();
  ...
  Infile = fopen(argv[1], "r");
  ...
  scanfile();
  exit(0);
}
```

接着 `scanfile()` 在存在新标记时循环，并打印出标记的详细信息：

And `scanfile()` loops while there is a new token and prints out the
details of the token:

```c
// List of printable tokens 可打印的标记列表
char *tokstr[] = { "+", "-", "*", "/", "intlit" };

// Loop scanning in all the tokens in the input file. 循环扫描输入文件的所有token
// Print out details of each token found. 打印所有找到的token的详细信息
static void scanfile() {
  struct token T;

  while (scan(&T)) {
    printf("Token %s", tokstr[T.token]);
    if (T.token == T_INTLIT)
      printf(", value %d", T.intvalue);
    printf("\n");
  }
}
```

## 一些示例输入文件

我提供了一些示例输入文件，你可以看看通过扫描器能从他们中扫描到些什么tokens以及看看哪些文件是被扫描器拒绝了的。

```
$ make
cc -o scanner -g main.c scan.c

$ cat input01
2 + 3 * 5 - 8 / 3

$ ./scanner input01
Token intlit, value 2
Token +
Token intlit, value 3
Token *
Token intlit, value 5
Token -
Token intlit, value 8
Token /
Token intlit, value 3

$ cat input04
23 +
18 -
45.6 * 2
/ 18

$ ./scanner input04
Token intlit, value 23
Token +
Token intlit, value 18
Token -
Token intlit, value 45
Unrecognised character . on line 3
```

## 总结及下一步

我们开始做了一个可以做标记四个数学运算符以及数字字面值的小而简易的词法分析扫描器。我们注意到如果读取输入时读取得太远，就需要跳过空白字符并将字符放回。

单个字符的tokens扫描起来是很简单的，但是如果是多个字符tokens的扫描就会变得比较麻烦。不过在最后，`scan()` 函数会将输入文件中的下一个标记返回到一个名为 `struct token` 的变量中：

```c
struct token {
  int token;
  int intvalue;
};
```

在下一节中，我们会构建一个递归深入的解析器来解析我们输入文件的语法，计算并打印出每个文件的最终值[Next step](../02_Parser/Readme.md)。
