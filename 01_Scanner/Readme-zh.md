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

## 扫描Tokens：scan()

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

That's it for the simple one-character tokens: for each recognised
character, turn it into a token. You may ask: why not just put
the recognised character into the `struct token`? The answer is that later
we will need to recognise multi-character tokens such as `==` and keywords
like `if` and `while`. So it will make life easier to have an enumerated
list of token values.

## Integer Literal Values

In fact, we already have to face this situation as we also need to
recognise integer literal values like `3827` and `87731`. Here is the
missing `default` code from the `switch` statement:

```c
  default:

    // If it's a digit, scan the
    // literal integer value in
    if (isdigit(c)) {
      t->intvalue = scanint(c);
      t->token = T_INTLIT;
      break;
    }

    printf("Unrecognised character %c on line %d\n", c, Line);
    exit(1);
```

Once we hit a decimal digit character, we call the helper function `scanint()`
with this first character. It will return the scanned integer value. To
do this, it has to read each character in turn, check that it's a
legitimate digit, and build up the final number. Here is the code:

```c
// Scan and return an integer literal
// value from the input file. Store
// the value as a string in Text.
static int scanint(int c) {
  int k, val = 0;

  // Convert each character into an int value
  while ((k = chrpos("0123456789", c)) >= 0) {
    val = val * 10 + k;
    c = next();
  }

  // We hit a non-integer character, put it back.
  putback(c);
  return val;
}
```

We start with a zero `val` value. Each time we get a character
in the set `0` to `9` we convert this to an `int` value with
`chrpos()`. We make `val` 10 times bigger and then add this new
digit to it.

For example, if we have the characters `3`, `2`, `8`, we do:

 + `val= 0 * 10 + 3`, i.e. 3
 + `val= 3 * 10 + 2`, i.e. 32
 + `val= 32 * 10 + 8`, i.e. 328

Right at the end, did you notice the call to `putback(c)`?
We found a character that's not a decimal digit at this point.
We can't simply discard it, but luckily we can put it back
in the input stream to be consumed later.

You may also ask at this point: why not simply subtract the ASCII value of 
'0' from `c` to make it an integer? The answer is that, later on, we will
be able to do `chrpos("0123456789abcdef")` to convert hexadecimal digits
as well.

Here's the code for `chrpos()`:

```c
// Return the position of character c
// in string s, or -1 if c not found
static int chrpos(char *s, int c) {
  char *p;

  p = strchr(s, c);
  return (p ? p - s : -1);
}
```

And that's it for the lexical scanner code in `scan.c` for now.

## Putting the Scanner to Work

The code in `main.c` puts the above scanner to work. The `main()`
function opens up a file and then scans it for tokens:

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

And `scanfile()` loops while there is a new token and prints out the
details of the token:

```c
// List of printable tokens
char *tokstr[] = { "+", "-", "*", "/", "intlit" };

// Loop scanning in all the tokens in the input file.
// Print out details of each token found.
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

## Some Example Input Files

I've provided some example input files so you can see what tokens
the scanner finds in each file, and what input files the scanner rejects.

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

## Conclusion and What's Next

We've started small and we have a simple lexical scanner that recognises
the four main maths operators and also integer literal values. We saw
that we needed to skip whitespace and put back characters if we read
too far into the input.

Single character tokens are easy to scan, but multi-character tokens are
a bit harder. But at the end, the `scan()` function returns the next token
from the input file in a `struct token` variable:

```c
struct token {
  int token;
  int intvalue;
};
```

In the next part of our compiler writing journey, we will build
a recursive descent parser to interpret the grammar of our input
files, and calculate & print out the final value for each file. [Next step](../02_Parser/Readme.md)
