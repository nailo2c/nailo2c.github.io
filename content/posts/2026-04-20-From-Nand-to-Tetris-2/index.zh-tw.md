---
title: "From Nand to Tetris (Part 2)"
date: "2026-04-20T00:00:00-00:00"
draft: true
description: ""
featuredImage: ""

tags: ["Study"]
categories: ["Study"]

math:
  enable: true
---

# Unit 1 - VM - Stack Arithmetric

VM是Stack Machine，主因是這樣VM跟Compiler都會變得簡潔，我認為是課堂需要。

Memory Segment將Memory切成數個區塊存放不同類別的資料例如 argument, local, static, constant

這堂課有八個virtual memory segment： local / argument / this / that / constant / static / pointer / temp

要注意 push xxx 1 是指複製 xxx 的數值到 stack，並不會消滅它 (xxx 不包含 constant)

pop xxx 2 是指從 stack 的頂層拿出數值後assign到 xxx 的 index 2 中 (xxx 不包含 constant)

## Pointer

D = *p 等價於下列三行:  
@p  
A=M  
D=M  

p-- 等價於 p = p - 1  
如果 RAM index p 的 value 是 257，那 p-- 後，該 value 會變 256

如果 RAM index q 的 value 是 1024，那 *q = 9 則是將 RAM index 1024 的 value 改成 9  
如果下一行執行 q++，則代表 RAM index q 的 value +1，也就是 1025

但題目又有點把我搞混，題目如下:

RAM[256] = 22
RAM[257] = 31
RAM[258] = 200
RAM[259] = 28

求:  
1  p1 = 256
2  p1 = p1 + 3
3  *p1 = *p1 + 3
4  p2 = p1 - 2
5  p1--
6  *(p2 + 1) = *p1 + *p2

第2行 -> p1 = 259  
第3行 -> *p1 = RAM[259] + 3 = 31 -> 代表寫31回RAM[259]  
第4行 -> p2 = 257  
第5行 -> p1 = 256  
第6行 -> *(p2+1) = *258 = *p1 + *p2 = 31 + 200 = 231 -> 代表寫231回RAM[258]  

看起來*p的意思就是要把資料寫進該address

*SP = 17  
SP++  

等價於下列:  
@17  // D=17  
D=A  
@SP  // *SP=D  
A=M  
M=D  
@SP  // SP++  
M=M+1  

## Memory Segment

1. Local - LCL
    1. LCL只記得base address，接著依序存放資料
    2. local有三個資料7, -91, 3，假設base address是1015，則RAM[1015]=7, RAM[1016]=-91, RAM[1017]=3
2. local / argument / this / that
    1. object -> this
    2. array -> that
    3. 上述四個都是有 base address，然後再依序取資料
    4. push 實作: addr = segmentPointer + i, *SP = *addr, SP++ (基本上push就是放到stack)
    5. pop 實作: addr = segmentPointer + i, SP--, *SP = *addr
3. constant
    1. 只有 push，沒有 pop
    2. *SP = i, SP++
4. static
    1. 他是全域變數，因此無法像上述segment可以dynamic的放在RAM裡
    2. 語法: static 5 -> @Foo.5
    3. 這堂課裡會把static放在RAM[16] ~ RAM[255]
5. temp
    1. Hack VM提供8個temp variables，RAM[5] ~ RAM[12]
    2. push 實作: addr = 5 + i, *SP = *addr, SP++
    3. pop 實作: addr = 5 + i, SP--, *SP = *addr
6. pointer
    1. 持續追蹤 this 跟 that 用，在compiler中比較能體現它的用處
    2. pointer 0 <=> THIS
    3. pointer 1 <=> THAT
    4. push 實作: *SP = THIS/THAT, SP++
    5. pop 實作: SP--, THIS/THAT = *SP
    6. "that7" = RAM[THAT + i]

## VM translator

VM translator目的是把VM code轉譯成Assembly Code

Unit 1的project先實作以下部分:

Arithmetic / Logical commands:  
`add`  
`sub`  
`neg`  
`eq`  
`gt`  
`lt`  
`and`  
`or`  
`not`  

Memory access commands:  
`pop segment i`  
`push segment i`  

# Unit 2 - VM - Program Control

## Branching

三種VM branching commands
1. label  
2. goto  
3. if-goto  

沒有branching，instructions就是從上而下。有了branching，instruction的執行順序就能跳來跳去的

+ Unconditional Branching
+ Conditional Branching

## Function
  
e.g. caller
```console
function main 0
    push constant 3
    push constant 8
    push constant 5
    call mult 2
    add
    return
```

e.g. callee
```
function mult 2
    push constant 0
    pop local 0
    push constant 1
    pop local 1
label LOOP
    push local 1
    push argument 1
    // ... computes the product into local 0
label END
    push local 0
    return
```

這邊mult 2代表callee這邊會init兩個LOCAL memory address  
例如 LOCAL 0 = result  
LOCAL 1 = counter  
然後caller丟的argument會存在ARGUMENT  
ARGUMENT 0 = 8  
ARGUMENT 1 = 5  

此時 counter 會執行回圈直到 ARGUMENT 1 = 5次，然後每次 result 會加上 ARGUMENT 0 = 8  

push constant 0 / pop local 0 代表把 local 0 = 0，通常要存乘法的結果。  
可以想像成 result = 0，然後進行以下步驟：
```
result = 0
result = result + 8
result = result + 8
result = result + 8
result = result + 8
result = result + 8
```

push constant 1 / pop local 1 代表把 local 1 = 1，通常是當counter，等local 1 = 6 時就停止。

### Function Call

當我們 call function 時，有些caller's frame需要被存起來。

以上述的例子:
```console
function main 0
    push constant 3  // working stack of the caller
    push constant 8  // argument 0
    push constant 5  // argument 1
    call mult 2
    add
    return
```

然後會額外儲存
saved LCL：恢復 caller 原本的 local segment  
saved ARG：恢復 caller 原本的 argument segment  
saved THIS：恢復 caller 原本的 THIS   
saved THAT：恢復 caller 原本的 THAT  

### Call的實作

假設要實作 mult 2
```
...
8                  <- mult 的 argument 0  // ARG
5                  <- mult 的 argument 1
return address
saved LCL
saved ARG
saved THIS
saved THAT
                    <- SP現在在這  // LCL
```

這些建造出來後，會計算  
ARG = SP - 5 - nArgs  // 在這邊是 2  
LCL = SP  

ARG的目的是讓function知道自己的各個segment位置  
LCL SP的目的是LOCAL segment的起始位置，這樣才能開始init LOCAL 0, LOCAL 1 等  

e.g.2.

假設有 function foo 4，代表會建立 LOCAL 0 ~ 3  
call foo 2，代表會先push兩個constant到stack再建立那些必要的segment  
```
push constant 5  // ARG
push constant 8
call foo 2
return address
saved LCL
saved ARG
saved THIS
saved THAT
LOCAL 0 = 0
LOCAL 1 = 0
LOCAL 2 = 0
LOCAL 3 = 0
SP
```

### Return的實作

```
endFrame = LCL            // endFrame is a temporary variable
retAddr = *(endFrame - 5) // gets the return address
*ARG = pop()              // Repositions the return value for the caller
SP = ARG + 1              // Repositions SP of the caller
THAT = *(endFrame - 1)    // Restores THAT of the caller
THIS = *(endFrame - 2)    // Restores THIS of the caller
ARG = *(endFrame - 3)     // Restores ARG of the caller
LCL = *(endFrame - 4)     // Restores LCL of the caller
goto retAddr              // goes to return address in the caller's code
```

## VM translator

function Foo.main -> (Foo.main)  
function Foo.bla -> (Foo.bla)  

+ Main
    + fileName.vm -> fileName.asm
    + directoryName -> 含一個或多個 .vm files -> 全部compile成 .asm files
+ Parser
    + 跟 project7 一樣，但要多 handle goto / if-goto / label / call / fucntion / return
+ CodeWriter
    + API
        + setFileName - fileName(string)
        + writeInit
        + writeLabel - label(string)
        + writeGoto - label(string)
        + writeIf - label(string)
        + writeFunction - functionName(string) / numVars(int)
        + writeCall - functionName(string) / numArgs(int)
        + writeReturn

# Unit 3 - Jack Language

本章學習 Jack Language 的特性。Jack Language 不是重點，重點是學習如何建構一個語言，後續會實作compiler去compile它。

Hello World
```jack
/** Hello World program. */
class Main {
    function void main() {
        /* Prints some text using the standard library. */
        do Output.printString("Hello World!");
        do Output.println();
        return;
    }
}
```

+ Comments:
    + /** API block comment */
    + /* block comment */
    + // in-line coment
+ White space (ignored)

Procedural processing
```jack
// Inputs some numbers and compputes their average
class Main {
    function void main() {
        var Array a;
        var int length;
        var int i, sum;
        let length = keyboard.readInt("How many numbers? ");
        let a = Array.new(length); // constructs the array
        let i = 0;
        while (i < length) {
            let a[i] = keyboard.readInt("Enter a number: ");
            let sum = sum + a[i];
            let i = i + 1;
        }
        do Output.printString("The average is ");
        do Output.printInt(sum / length);
        return;
    }
}
```

+ Main class 一定得存在
+ Main class 至少要有一個function叫main
+ Program's entry point Main.main
+ Arrays
    + Array is implemented as part of the standard class library
    + Jack Array沒有type，意思是塞什麼進去都行
+ Jack data types:
    + Primitive
        + int
        + char
        + boolean
    + Class types
        + OS: array, String, ...
        + Program extensions: as needed

因為 Jack 只有 int/char/boolean，因此會需要OO programming來定義一些type例如 fractions

## OO programming: building a API

+ Fraction API
```jack
class Fraction {
    /** Constructs a (reduced) fraction from the given numerator and denominator */
    constructor Fraction new(int x, int y)

    /** Accessors. */
    method int getNumerator()
    method int getDenominator()

    /** Returns the sum of this fraction and the other one. */
    method Fraction plus(Fraction other)

    /** Disposes this fraction. */
    method void dispose()

    /** Prints this fraction in the format x/y. */
    method void print()
}
```

+ Example of using Fraction API
```jack
class Main {
    function void main() {
        var Fraction a, b, c;
        let a = Fraction.new(2, 3);
        let b = Fraction.new(1, 5);
        let c = a.plus(b);
        do c.print();
        return;
    }
}
```

## OO programming: building a class (Abstract)

+ Fraction class
```jack
/** Represents the Fraction type and related operations. */
class Fraction {

    field int numerator, denominator;

    /** Constructs a (reduced) fraction from the given numerator and denominator. */
    constructor Fraction new(int x, int y) {
        let numerator = x;
        let denominator = y;
        do reduce();
        return this;
    }

    // Reduces this fraction.
    method void reduce() {
        var int g;
        let g = Fraction.gcd(numerator, denominator);
        if (g > 1) {let numerator = numerator / g; let denominator = denominator / g;}
        return;
    }

    // Computes the greatest common divisor of the given integers.
    function int gcd(int a, int b) {
        var int r;
        while (~(b = 0)) {                // applies Euclid's algorithm.
            let r = a - (b * (a / b));    // r = remainder of the integer division a/b
            let a = b; let b = r; }
        return a;
        }
    }

    /** Accessors. */
    method int getNumerator() {
        return numerator;
    }

    method int getDenominator() {
        return denominator;
    }

    /** Returns the sum of this fraction and the other one. */
    method Fraction plus(Fraction other) {
        var int sumNumerators;
        let sumNumerators = (numerator * other.getDenominator()) +
                            (other.getNumerator() * denominator);
        return Fraction.new(sumNumberators, denominator * other.getDenominator());
    }

    /** Prints this fraction in the format x/y. */
    method void print() {
        do Output.printInt(numerator); do Output.printString("/");
        do Output.printInt(denominator);
        return;
    }

    /** Disposes this fraction. */
    method void dispose() {
        do Memory.deAlloc(this);
        return;
    }
}  // Fraction class
```

Dispose 在 Jack 中是必要的因為 Jack 沒有 GC。

## List processing

這堂課的List是用linked-list style呈現

(2, (3, (5, null))) -> commonly abbreviated as (2, 3, 5)

+ List API
```jack
/** Represents a linked list of integers. */
class List {
    /** Creates a List */
    constructor List new(int car, List cdr)

    /** Prints this list */
    method void print() {}

    /** Disposes this list */
    method void dispose() {}
}
```

+ Main
```jack
// Build, prints, and dispose a list
class Main {
    function void main() {
        // Creates and manipulates the list (2, (3, (5, null))),
        // commonly referred to as (2,3,5)
        var List v;
        let v = List.new(5, null);
        let v = List.new(3, v);
        let v = List.new(2, v);
        do v.print();
        do v.dispose();  // disposes the list
        return;
    }
}
```

+ List class
```jack
class List {
    field int data;  // a list consists of a data field,
    field List next; // followed by a list.

    /* Creates a List. */
    constructor List new(int car, List cdr) {
        let data = car;
        let next = cdr;
        return this;
    }

    /** Prints this list. */
    method void print() {
        var List current;
        let current = this;
        while (~(current == null)) {
            do Output.printInt(current.getData());
            do Output.printChar(32);  // prints a space
            do current = current.getNext();
        }
        return;
    }

    /** Disposes this list */
    // by recursively disposing its tail
    method void dispost() {
        if (~(next == null)) {
            do next.dispost()
        }
        // Uses an OS routine to recycle this list
        do Memory.deAlloc(this);
        return;
    }
}
```

List在RAM中這樣存

![List RAM](List_RAM.png)

## Jack Syntax

| Syntax | Description |
| --- | --- |
| `class`, `constructor`, `method`, `function` | Program components |
| `int`, `boolean`, `char`, `void` | Primitive types |
| `var`, `static`, `field` | Variable declarations |
| `let`, `do`, `if`, `else`, `while`, `return` | Statements |
| `true`, `false`, `null` | Constant values |
| `this` | Object reference |

## Jack Symbols

| Symbol | Description |
| --- | --- |
| `()` | Used for grouping arithmetic expressions and for enclosing parameter-lists and argument-lists |
| `[]` | Used for array indexing |
| `{}` | Used for grouping program units and statements |
| `,` | Variable list separator |
| `;` | Statement terminator |
| `=` | Assignment and comparison operator |
| `.` | Class membership |
| <code>+</code>, <code>-</code>, <code>*</code>, <code>/</code>, <code>&amp;</code>, <code>&#124;</code>, <code>~</code>, <code>&lt;</code>, <code>&gt;</code> | Operators |

## Type conversions

1. Char跟Int可以互換
```jack
var char c;
let c = 65;  // 'A'
// Equivalently:
var String s;
let s = "A";
let c = s.charAt(0);
// Note that the idiom c = 'A' is not supported by the Jack language
```

2. Int可以當作memory address被assigned
```jack
var Array arr;     // creates a pointer variable
let arr = 5000;    // sets arr to the memory address 5000
let arr[100] = 17; // sets memory address 5100 to 17
```

3. Object可以被轉成Array，反之亦然
```jack
var Fraction x; // a Fraction object has two int fields: numerator and denominator.
var Array arr;
let arr = Array.new(2);
let arr[0] = 2;
let arr[1] = 5;
let x = arr;    // sets x to the base address of the memory block representing the array [2, 5]
do x.print();   // will print 2/5
```

## Classes

之前在 Fraction 做的 gcd 可以抽到 `Math` class 當 Library 來用

+ Jack Standard Library

| OS class | Services |
| --- | --- |
| `Math` | Common mathematical operations: `multiply(int, int)`, `sqrt(int)`, etc. |
| `String` | Represents string objects and related methods: `length()`, `charAt(int)`, etc. |
| `Array` | Represents array objects and related operations: `new(int)`, `dispose()`. |
| `Output` | Supports text output to the screen:<br>`printString(String)`, `printInt(int)`, `println()`, etc. |
| `Screen` | Supports graphics output to the screen: `drawPixel(int, int)`, `setColor(boolean)`, `drawCircle(int, int, int)`, etc. |
| `Keyboard` | Supports input from the keyboard: `readLine(String)`, `readInt(String)`, etc. |
| `Memory` | Facilitates access to the host RAM:<br>`peek(int)`, `poke(int, int)`, `alloc(int)`, `deAlloc(Array)`. |
| `Sys` | Supports execution-related services: `halt()`, `wait(int)`, etc. |

## Methods

一段可被呼叫、可重複利用的程式稱為subroutines

+ Constructors
    + 0, 1 or more in a class
    + Common name: new
    + The constructor's type must be the name of the constructor's class
    + The constructor must return a reference to an object of the class type
+ Variables
    + static / field / local / prarmeter
+ Statement
    + let / if / while / do / return
+ Expressions
    + A constant / A variable name in scope / this / Arr[expression] / A subroutine call
    + `-` or `~`
    + `()`
+ Strings
+ Arrays
    + 在Jack中array不是個type，是個object
    + multi-array要在array中塞array
+ Peculiar Features
    + let - must be used in assignments: `let x = 0;`
    + do - must be used for calling a method or a function outside an expression: `do reduce();`
    + Body of statement 必須包在 brackets: `if (a > 0) {return a} else {return -a};`
    + All subroutine must end with a `return`
    + 沒有先乘除後加減: 不能 `2 + 3 * 4`，在Jack必須寫成 `2 + (3 * 4)`
    + Jack is weakly typed

# Unit 4 - Compiler I: Syntax Analysis

## Tokenizing

+ 看起來像是把code切開by whitespace
+ Token的定義 - A token is a string of characters that has a meaning
+ Token有以下類別
    + Keyword
    + Symbol
    + intergenConstant
    + StringConstant
    + identifier

![](./token.png)

e.g.
```jack
if (x < 0) {
    // prints the sign
    let sign = "negative";
}
```

outptu:
```bash
<keyword> if </keyword>
<symbol> ( </symbol>
<identifier> x </identifier>
<symbod> < </symbol>
<intConst> 0 </intConst>
...
```

## Grammar

![](./grammar.png)

## Parser Logic

Parse是一個recursion，底下演示一個 while 語句如何轉換為 XML。  
`while ( count < 100 ) { let count = count + 1 ; }`
```xml
<whileStatement>
    <keyword> while </keyword>
    <symbol> ( </symbol>
    <expression>
        <term>
            <identifier> count </identifier>
        </term>
        <symbol> < </symbol>
        <term>
            <integerConstant> 100 </integerConstant>
        </term>
    </expression>
    <symbol> ) </symbol>
    <symbol> { </symbol>
    <statements>
        <letStatement>
            <keyword> let </keyword>
            <identifier> count </identifier>
            <symbol> = </symbol>
            <expression>
                <term>
                    <identifier> count </identifier>
                </term>
                <symbol> + </symbol>
                <term>
                    <integerConstant> 1 </integerConstant>
                </term>
            </expression>
            <symbol> ; </symbol>
        </letStatement>
    </statements>
    <symbol> } </symbol>
</whileStatement>
```

這邊使用 LL grammar，然後做 LL(1)，意思是一次只往前看一個

## Jack grammar

e.g.  
`parameterList: ( ( type varName )   ( ‘,’ type varName ) * ) ?`

這邊的意思是  
1. (...)? 代表括號內可以出現0或1次，因此 `foo()` 這樣的function是能被解析的
2. (type varName) 代表可以長這樣 `int x`
3. (',' type varName)* 代表後面可接 0 ~ N 個參數
    1. 例如: (int x, int y) / (int x, int y, boolean flag) / etc

### Lexical Elements

The Jack language includes five types of terminal elements, or tokens.

| Element | Rule |
|---|---|
| `keyword` | `'class' \| 'constructor' \| 'function' \| 'method' \| 'field' \| 'static' \| 'var' \| 'int' \| 'char' \| 'boolean' \| 'void' \| 'true' \| 'false' \| 'null' \| 'this' \| 'let' \| 'do' \| 'if' \| 'else' \| 'while' \| 'return'` |
| `symbol` | `'{' \| '}' \| '(' \| ')' \| '[' \| ']' \| '.' \| ',' \| ';' \| '+' \| '-' \| '*' \| '/' \| '&' \| '\|' \| '<' \| '>' \| '=' \| '~'` |
| `integerConstant` | A decimal number in the range `0..32767`. |
| `stringConstant` | A sequence of Unicode characters not including double quote or newline, enclosed in double quotes. |
| `identifier` | A sequence of letters, digits, and underscore `_`, not starting with a digit. |

---

### Program Structure

A Jack program is a collection of classes, each appearing in a separate file.  
The compilation unit is a class.

| Non-terminal | Rule |
|---|---|
| `class` | `'class' className '{' classVarDec* subroutineDec* '}'` |
| `classVarDec` | `('static' \| 'field') type varName (',' varName)* ';'` |
| `type` | `'int' \| 'char' \| 'boolean' \| className` |
| `subroutineDec` | `('constructor' \| 'function' \| 'method') ('void' \| type) subroutineName '(' parameterList ')' subroutineBody` |
| `parameterList` | `((type varName) (',' type varName)*)?` |
| `subroutineBody` | `'{' varDec* statements '}'` |
| `varDec` | `'var' type varName (',' varName)* ';'` |
| `className` | `identifier` |
| `subroutineName` | `identifier` |
| `varName` | `identifier` |

---

### Statements

| Non-terminal | Rule |
|---|---|
| `statements` | `statement*` |
| `statement` | `letStatement \| ifStatement \| whileStatement \| doStatement \| returnStatement` |
| `letStatement` | `'let' varName ('[' expression ']')? '=' expression ';'` |
| `ifStatement` | `'if' '(' expression ')' '{' statements '}' ('else' '{' statements '}')?` |
| `whileStatement` | `'while' '(' expression ')' '{' statements '}'` |
| `doStatement` | `'do' subroutineCall ';'` |
| `returnStatement` | `'return' expression? ';'` |

---

### Expressions

| Non-terminal | Rule |
|---|---|
| `expression` | `term (op term)*` |
| `term` | `integerConstant \| stringConstant \| keywordConstant \| varName \| varName '[' expression ']' \| subroutineCall \| '(' expression ')' \| unaryOp term` |
| `subroutineCall` | `subroutineName '(' expressionList ')' \| (className \| varName) '.' subroutineName '(' expressionList ')'` |
| `expressionList` | `(expression (',' expression)*)?` |
| `op` | `'+' \| '-' \| '*' \| '/' \| '&' \| '\|' \| '<' \| '>' \| '='` |
| `unaryOp` | `'-' \| '~'` |
| `keywordConstant` | `'true' \| 'false' \| 'null' \| 'this'` |

## Implementation

+ JackTokenizer
    + Handle Lexical elelemts
    + Modules: hasMoreTokens() / advance() / tokenType()
+ CompilationEngine
    + Handle Program structure / Statemets / Expressions
    + compileStatements() / compileIfStatement() / compileWhileStatement()
        + Detail in Unit 4.5
+ JackAnalyzer
    + input: file / directory
    + output: xml / xml for all files in directory
    + Uses the services of a JackTokenizer (check Jack grammar)

### JackTokenizer API

**JackTokenizer**: Ignores all comments and white space in the input stream, and serializes it into Jack-language tokens. The token types are specified according to the Jack grammar.

| Routine | Arguments | Returns | Function |
|---|---|---|---|
| `Constructor` | input file / stream | - | Opens the input `.jack` file and gets ready to tokenize it. |
| `hasMoreTokens` | - | `boolean` | Are there more tokens in the input? |
| `advance` | - | - | Gets the next token from the input, and makes it the current token.<br><br>This method should be called only if `hasMoreTokens` is true.<br><br>Initially there is no current token. |
| `tokenType` | - | `KEYWORD`<br>`SYMBOL`<br>`IDENTIFIER`<br>`INT_CONST`<br>`STRING_CONST` | Returns the type of the current token, as a constant. |
| `keyword` | - | `CLASS`<br>`METHOD`<br>`FUNCTION`<br>`CONSTRUCTOR`<br>`INT`<br>`BOOLEAN`<br>`CHAR`<br>`VOID`<br>`VAR`<br>`STATIC`<br>`FIELD`<br>`LET`<br>`DO`<br>`IF`<br>`ELSE`<br>`WHILE`<br>`RETURN`<br>`TRUE`<br>`FALSE`<br>`NULL`<br>`THIS` | Returns the keyword which is the current token, as a constant.<br><br>This method should be called only if `tokenType` is `KEYWORD`. |
| `symbol` | - | `char` | Returns the character which is the current token. Should be called only if `tokenType` is `SYMBOL`. |
| `identifier` | - | `string` | Returns the identifier which is the current token. Should be called only if `tokenType` is `IDENTIFIER`. |
| `intVal` | - | `int` | Returns the integer value of the current token. Should be called only if `tokenType` is `INT_CONST`. |
| `stringVal` | - | `string` | Returns the string value of the current token, without the two enclosing double quotes. Should be called only if `tokenType` is `STRING_CONST`. |

### CompilationEngine API

**CompilationEngine**: generates the compiler's output.

| Routine | Arguments | Returns | Function |
|---|---|---|---|
| `CompileExpression` | -- | -- | Compiles an expression. |
| `CompileTerm` | -- | -- | Compiles a term. If the current token is an identifier, the routine must distinguish between a variable, an array entry, or a subroutine call. A single look-ahead token, which may be one of `[`, `(`, or `.`, suffices to distinguish between the possibilities. Any other token is not part of this term and should not be advanced over. |
| `CompileExpressionList` | -- | -- | Compiles a possibly empty comma-separated list of expressions. |


# Unit 5 - Compiler II: Code Generation


### 5.1

+ 一次只編譯一個 class，為了簡化複雜度 + 模組化
+ 可以只編譯 file 裡的一個 subroutine，目的也是模組化

### 5.2

+ source code 裡的變數，例如 x，在轉換成 vm code 時會需要知道它是 field, static, local, argument 以及 memory 位置，因為 vm code 沒有變數。
+ 變數有四種資訊要記錄: name, type, kind, scope
    + 例子: `field int x;` 其中 name=x, type=int, kind=field, scope=class level
+ 用 Symbol tables 紀錄: 包含 name, type, kind, # (memory address)
    + Symbol tables 有 class-level 與 subroutine-level
+ method 裡會隱含 this，因為 method 可能用到 class-level 的變數例如 x，則這個 x 其實就是 this.x
+ 如果一個 class 有兩個 subroutine，則會有一個 class-level 的 symbol table，以及兩個 subroutine-level 的 symbol table。只是在進行第二個 subroutine 時，就會把第一個 subroutine-level 的 symbol table 丟棄

### 5.3

+ infix  : a * (b + c)
+ prefix : * a + b c
+ postfix: a b c + *

這個 unit 的目標是將 code 從 infix-style 轉為 postfix-style

Project 11 的 compiler 目標是將 source code 轉為 VM code

e.g.  
```
let x = a + b - c;
->
push a
push b
+
push c
-
pop x
```

### 5.4

這個 unit 談論 `while` & `if` statements，因為其他的 `let`, `do`, `return` 相對單純。

+ `if` statement
    + 先做用 not else，這樣條件為 false 時會先跳到 label L1 來執行 else 的邏輯
    + 如果不用 not 寫法，vm code會比較長一點

```
compiled expression
not
if-goto L1

compiled statement1
goto L2

label L1
compiled statement2

label L2
```

+ `while` statement
    + 也是用 not 邏輯

```
lable L1
    compiled (expression)
    not
    if-goto L2
    compiled (statements)
    goto L1
label L2
...
```

+ 多個 `if` 跟 `while` statements
    + label should be unique
    + 結構時常會是 nested


### 5.5

+ High-level 的 jack code 有 object 概念
+ Mid-level 的 VM programs 只有 local / argument / this / that / pointer / constant / static / temp
+ Low-level 只有 RAM address
+ Compiler 弭平了各個 level 間的 gap
+ host RAM 在 stack 段存放 local & argument，在 heap 段存放 objects & arrays
    + 用 this/that 去存取 objects & arrays

| VM code (commands) | VM implementation (resulting effect) |
| ------------------ | ------------------------------------ |
| push 8000 <br> pop pointer 0 | <br> sets THIS to 8000     |
| push/pop this 0 <br> push/pop this 1 <br> ... <br> push/pop this i | accessing RAM[8000] <br> accessing RAM[8001] <br> ... <br> accessing RAM[8000+i] |

### 5.6

```jack
...
var Point p1;
...
let p1 = Point.new(2, 3);
...
```

p1 會存到 local variable (stack) 中，然後 object 的細節則會存到 heap 中。

Init 一個 object 時，會使用 `Memory.alloc` 來確認哪一段 memory 是有足夠的空間來初始化這個 object。

範例 (class) :
```jack
class Point {
    field int x, y;
    static int pointCount;
    ...
    /** Constructs a new point */
    constructor Point new(int ax, int ay) {
        let x = ax;
        let y = ay;
        let pointCount = pointCount + 1;
        return this;
    }
}
```

compiled code 如下:
```vm
// constructor Point new(int ax, int ay)

// The compiler creates the subroutine's symbol table.
// The compiler figures out the size of an object of this class (n), and writes code that calls Memory.alloc(n).
// This OS method finds a memory block of the required size, and returns its base address.

push 2  // two 16-bit words are required (x and y) p.s. one word = 16 bits
call Memory.alloc 1  // one argument. assume it's RAM[6012], RAM[6013], it would return 6012 to pointer
pop pointer 0  // anchors this at the base address, which means 6012 would be pop and put it to pointer 0

// let x = ax; let y = ay;
push argument 0  // when symbol table be created, ax has been put on argument 0
pop this 0       // pop stack top (ax), and put it to this 0
pusn argument 1
pop this 1

// let pointCount = pointCount + 1;
push static 0
push 1
add
pop static 0

// return this
push pointer 0
return
```

### 5.7

這一個 Unit 會解釋怎麼操作 Object。

+ 將 OO style 轉為 Procedural style
    + `p1.distance(p2) -> distance(p1,p2)`
    + `p1.getx() -> getx(p1)`
    + `obj.foo(x1, x2, ...) -> foo(obj, x1, x2, ...)`

```vm
// obj.foo(x1, x2, ...)
push obj
push x1
push x2
push ...
call foo

// let d = p1.distance(p2);
...
push p1
push p2
call distance
...
```

+ 如何 compiling methods?
```jack
class Point {
    field int x, y;
    static int pointCount;
    ...
    method int distance(Point other) {
        var int dx, dy
        let dx = x - other.getx();
        let dy = y - other.gety();
        return Math.sqrt((dx*dx) + (dy*dy));
    }
    ...
}
```

VM code
```
// method int distance(Point other)
// var int dx, dy
push argument 0
pop pointer 0  // THIS = argument 0

// let dx = x - other.getx()
push this 0   // this 0 store the first field, in this case is `x`
push argument 1
call Point.getx 1
sub
pop local 0
// let dy = y - other.gety()
// Similar, code omitted.

// return Math.sqrt((dx*dx) + (dy*dy))
push local 0
push local 0
call Math.multiply 2
push local 1
push local 1
call Math.multiply 2
add
call Math.sqrt 1
return
```

補充:  
push argument 0  → 拿到 object address  
push this 0      → 拿到 object 的第 0 個 field，也就是 x  

+ 如何 compiling void methods
```
// method void print()
push argument 0
pop pointer 0
// compiled code omitted
// Methods must return a value
push constant 0
return
```

這堂課我們約定 method 一定要有值 return，因此 void 我們 push constant 0，然後在外層（caller層），用 pop temp 0 把 stack top 的 constant 0 給 drop 掉。

### 5.8

跟 object 一樣， array 被 init 時會在 stack (local) 一個地方紀錄 heap 的 address。例如 `Array.new(5)` 就會在 heap 找五個地址。

array 通常用 `THAT` memory segment，使用 `pop pointer 1` 來 set。

e.g.
```vm
// arr[2] = 17
push arr  // e.g. 8056
push 2    
add       // 8058
pop pointer 1  // store 8058 to THAT 0
push 17
pop that 0  // store 17 to RAM[8058]
```

pointer 1 代表 THAT，每次 arr index 變動時，都會重新把對應的 memory address 存到 THAT，再把 value 存到 RAM[THAT] 即可。

當更複雜的情境出現時 `a[i] = b[j]`，我們會需要 temp variable。
```vm
// a[i] = b[j]
push a
push i
add
push b
push j
add            // state 1
pop pointer 1
push that 0
pop temp 0     // state 2
pop pointer 1
push temp 0    // state 3
pop that 0
```

第一個 pop pointer 1 是指說把Stack top (也就是 b[j] 的memory address) pop 到 pointer 1  
接下來 push that 0，就是把剛剛存到 pointer 1 的 value 丟到 stack top  
然後 pop temp 0，就會把 b[j] 的 value 存到 temp 0   

接下來把 a[i] 的 mem address 丟到 pointer 1  
push temp 0 也就是把 b[j] value 丟到 stack top  
接著把 b[j] value assign到 that 0，也就是 a[i] 的 memory address  

### 5.9

這個 Unit 是個大總結:

```markdown
# Jack Compiler Standard Mapping over VM

## 1. File mapping

// 每個 Jack class file 會被編譯成同名的 VM file
Main.jack   -> Main.vm
Point.jack  -> Point.vm
Square.jack -> Square.vm


## 2. Subroutine mapping

// Jack 裡的 constructor / function / method，到 VM 層都會變成 function
constructor Point new(...) -> function Point.new
function int foo(...)      -> function ClassName.foo
method int bar(...)        -> function ClassName.bar


## 3. Function / Constructor arguments

// function 和 constructor 有幾個 Jack arguments，VM 就有幾個 arguments
function int foo(int x, int y)

x -> argument 0
y -> argument 1


constructor Point new(int ax, int ay)

ax -> argument 0
ay -> argument 1


## 4. Method arguments

// method 會多一個隱含的 this argument
method int distance(Point other)

this  -> argument 0
other -> argument 1


// 呼叫 method 時，caller 會把 object address 當成第一個 argument 傳入
p1.distance(p2)

argument 0 = p1 object address
argument 1 = p2 object address


## 5. Variable mapping

// Jack variable kind 對應到 VM segment
local    -> local
argument -> argument
static   -> static
field    -> this


## 6. Local variables

// subroutine 裡的 var 會對應到 local segment
var int x, y;

x -> local 0
y -> local 1


## 7. Argument variables

// subroutine 參數會對應到 argument segment
function void foo(int x, int y)

x -> argument 0
y -> argument 1


## 8. Static variables

// static 是 class-level 共用變數，對應到 static segment
static int pointCount;

pointCount -> static 0


## 9. Field variables

// field 是 object-level 變數，對應到 this segment
field int x, y;

x -> this 0
y -> this 1


// 但使用 this 之前，必須先讓 pointer 0 指向目前 object
push argument 0
pop pointer 0

pointer 0 = THIS
this 0 = currentObject.x
this 1 = currentObject.y


## 10. Method setup

// method 一開始要把 argument 0 設成 THIS
method int distance(Point other)

push argument 0
pop pointer 0


// 之後 method 裡的 field access 才會正確
x -> this 0
y -> this 1


## 11. Method call

// 呼叫 method 時，要先 push 呼叫者 object，再 push 明確參數
p1.distance(p2)

push p1
push p2
call Point.distance 2


// 進入 Point.distance 後
argument 0 = p1 = this
argument 1 = p2 = other


## 12. Constructor setup

// constructor 要配置新 object 的 heap 空間
class Point {
    field int x, y;
}

Point object size = 2 words


// 呼叫 Memory.alloc 配置 2 words，並讓 this 指向新 object
push constant 2
call Memory.alloc 1
pop pointer 0


// 初始化 fields
let x = ax;
let y = ay;

push argument 0
pop this 0

push argument 1
pop this 1


// constructor 最後回傳 this，也就是新 object 的 heap address
return this;

push pointer 0
return


## 13. Object construction from caller side

// 宣告 object variable 不會建立 object，只是建立一個變數存 address
var Point p1;

p1 -> local 0


// 呼叫 constructor 才會真的建立 object
let p1 = Point.new(2, 3);

push constant 2
push constant 3
call Point.new 2
pop local 0


// 結果
local 0 = p1 object address

p1 本身在 local segment
p1 指向的 object data 在 heap


## 14. Array declaration

// 宣告 array variable 不會建立 array，只是建立一個變數存 address
var Array arr;

arr -> local 0


## 15. Array construction

// Array.new(n) 才會真的在 heap 配置 array 空間
let arr = Array.new(n);

push n
call Array.new 1
pop local 0


// 結果
local 0 = arr base address

arr 本身在 local segment
arr 的內容在 heap


## 16. Array access

// arr[i] 的核心做法是先算出 arr + i
arr[i] address = arr base address + i


// 然後把 pointer 1 設成 arr[i] 的 address
push arr
push i
add
pop pointer 1


// pointer 1 = THAT
// that 0 = arr[i]
push that 0


## 17. Array assignment

// let arr[i] = value;
push arr
push i
add
pop pointer 1

push value
pop that 0


## 18. Complex array assignment

// let a[i] = b[j];

// 先算 a[i] 的 address，留在 stack
push a
push i
add

// 再算 b[j] 的 address
push b
push j
add

// 讓 THAT 指向 b[j]
pop pointer 1

// 讀出 b[j] 的 value，暫存在 temp 0
push that 0
pop temp 0

// 讓 THAT 改指向 a[i]
pop pointer 1

// 把 b[j] 的 value 寫進 a[i]
push temp 0
pop that 0


## 19. Void subroutine

// VM 規定每個 function 都要 return 一個 value
// 所以 void subroutine 也要回傳 dummy value
method void print()

push constant 0
return


## 20. Void method call

// caller 不需要 void method 的回傳值，所以要把 dummy value 丟掉
do p1.print();

push p1
call Point.print 1
pop temp 0


## 21. do statement

// do statement 表示呼叫 subroutine，但不使用回傳值
do something();

call Something
pop temp 0


## 22. return statement

// return expression;
compile expression
return


// return;
push constant 0
return


## 23. true / false / null

// false 和 null 都是 0
false -> push constant 0
null  -> push constant 0


// true 是 -1
true -> push constant 1
        neg


## 24. Arithmetic operators

// VM 有些 arithmetic command 可以直接用
+  -> add
-  -> sub
&  -> and
|  -> or
<  -> lt
>  -> gt
=  -> eq
~  -> not


// multiplication / division 需要呼叫 OS function
* -> call Math.multiply 2
/ -> call Math.divide 2


## 25. String constants

// String constant 要透過 String.new 和 String.appendChar 建立
"abc"

push constant 3
call String.new 1

push constant 97
call String.appendChar 2

push constant 98
call String.appendChar 2

push constant 99
call String.appendChar 2


## 26. Object allocation

// 建立 object 時，constructor 會呼叫 Memory.alloc
object size = number of fields

push constant objectSize
call Memory.alloc 1
pop pointer 0


## 27. Memory deallocation

// 回收 object / array 時，使用 Memory.deAlloc
push objectAddress
call Memory.deAlloc 1


## 28. OS classes

// Jack OS 會提供一些 VM classes
Math.vm
Memory.vm
Array.vm
String.vm
Output.vm
Screen.vm
Keyboard.vm
Sys.vm


// compiler 產生的 VM code 可以直接呼叫 OS functions
call Math.multiply 2
call Math.divide 2
call Memory.alloc 1
call String.new 1
call Output.printString 1


# Core Summary

class file    -> VM file
subroutine    -> VM function

local         -> local segment
argument      -> argument segment
static        -> static segment
field         -> this segment

method        -> argument 0 is this
constructor   -> Memory.alloc + pointer 0 + return this
array access  -> pointer 1 + that 0
void return   -> push constant 0
do call       -> pop temp 0

pointer 0     -> THIS
pointer 1     -> THAT

this i        -> current object's field i
that 0        -> currently selected array entry

object variable -> stores heap address
array variable  -> stores heap address
object data     -> stored in heap
array data      -> stored in heap
```

### 5.10

```
# Project 11 Additions

## Output target

// Project 11 輸出 VM code，不再輸出 XML
Main.jack  -> Main.vm
Point.jack -> Point.vm


## SymbolTable

// 新增 symbol table 管理 identifier metadata
name
type
kind
index


## SymbolTable scopes

// 只需要兩層 scope
classScope
subroutineScope


// classScope：每個 class reset
static
field


// subroutineScope：每個 subroutine reset
argument
local


## SymbolTable API

// 建立 / 重設 scope
new()
startSubroutine()


// 定義 symbol
define(name, type, kind)


// 查詢 symbol count
varCount(kind)


// 查詢 symbol metadata
kindOf(name)
typeOf(name)
indexOf(name)


## Symbol index rule

// 同 kind 內從 0 開始遞增
static 0
static 1

field 0
field 1

argument 0
argument 1

local 0
local 1


## VMWriter

// 新增 VM output abstraction
writePush(segment, index)
writePop(segment, index)
writeArithmetic(command)
writeLabel(label)
writeGoto(label)
writeIf(label)
writeCall(name, nArgs)
writeFunction(name, nLocals)
writeReturn()


## CompilationEngine changes

// 從 XML emission 改成 VM emission
XML tags -> VMWriter calls


// compileXXX contract
read XXX
advance tokenizer beyond XXX
emit VM code for XXX


## Expression contract

// compileExpression / compileTerm 產生的 VM code
// 必須把 expression result 留在 stack top
expression -> stack top


## Code generation dependencies

// CompilationEngine 需要使用
JackTokenizer
SymbolTable
VMWriter


## Identifier handling

// identifier lookup order
subroutineScope
classScope


// 找不到時，可能是 class name 或 subroutine name
symbol not found -> className / subroutineName


## Implementation hint

// SymbolTable 可用 hash table 實作
classScope      -> HashMap
subroutineScope -> HashMap
```

### 5.11

```
# Project 11 Overview

## Goal

// 把 syntax analyzer 擴充成 full compiler
Jack source code -> VM code


## Stage 1: SymbolTable

// 先讓 compiler 理解 identifier 的語意
identifier -> category
identifier -> kind
identifier -> index
identifier -> define / use


// identifier category
var
argument
static
field
class
subroutine


// var = local
var -> local


## Stage 1 Testing

// 先用 enhanced XML 測試 SymbolTable
Project 10 XML
+ identifier metadata


// 目的
test SymbolTable in isolation


## Stage 2: Code Generation

// 不再輸出 XML，改成輸出 VM code
XML output -> VM output


// 核心工作
CompilationEngine -> VMWriter calls


## Testing Rule

// 測試程式本身是正確的
// 如果執行錯，是 compiler bug
test failure -> fix compiler


## Test Program 1: Seven

// 測試最基本 VM code generation
constant arithmetic
do statement
return statement


## Test Program 2: ConvertToBin

// 測試基本 statements 和 simple expressions
let
if
while
do
return
simple expression


## Test Program 3: Square

// 測試 object-oriented basics
constructor
method
method call expression
object manipulation


## Test Program 4: Average

// 測試 arrays 和 strings
array access
array assignment
String.new
String.appendChar


## Test Program 5: Pong

// 測試完整 OOP application
multiple classes
objects
methods
constructors
static variables
interactive program


## Test Program 6: ComplexArrays

// 測試複雜 array expression
complex index expression
nested array access
array manipulation


## Development Strategy

// 按測試程式順序逐步實作 compiler
Seven -> ConvertToBin -> Square -> Average -> Pong -> ComplexArrays


## Execution Flow

// 每個測試程式的流程
compile .jack directory
generate .vm files
load directory into VM Emulator
run generated VM code
inspect output / RAM / screen
fix compiler if wrong
```
