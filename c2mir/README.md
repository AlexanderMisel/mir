# C到MIR编译器
  * 实现一个小型的C11（2011年ANSI C标准）到MIR（中间表示）编译器
    * 不支持可选的标准特性：可变大小数组、复数、原子操作
    * 支持以下C扩展功能：
      * `\e` 转义序列
      * 以 `0b` 或 `0B` 前缀开始的二进制数字
      * 宏 `__has_include`
      * 空结构体、联合体和初始化列表
      * 范围 case `case <start>...<finish>`
      * 零大小数组
      * 语句表达式
  * 编译器代码依赖性最小。没有使用额外的工具（如yacc/flex）
  * 简单易懂的实现优先于速度，以便于学习和维护代码
    * 通过四个阶段将编译过程划分为可管理的子任务：
      1. 预处理阶段生成标记
      2. 解析阶段生成AST（抽象语法树）。为了尽快接近ANSI标准的语法，使用了[**PEG**](https://en.wikipedia.org/wiki/Parsing_expression_grammar)手动解析器
      3. 上下文检查阶段检查上下文约束并扩充AST
      4. 生成阶段产生MIR

  ![C to MIR](c2mir.svg)

  C到MIR编译器可以作为库的一部分，并将其作为您的代码的一部分使用。该编译器也可以像通常的C编译器一样作为单独的程序使用。

  为了识别由C到MIR编译器进行的编译，可以使用特定于编译器的宏`__mirc__`和`__MIRC__`，将其定义为1。

  关于C到MIR编译器的其他信息可以在[这篇博客文章](https://developers.redhat.com/blog/2021/04/27/the-mir-c-interpreter-and-just-in-time-jit-compiler)中找到。
  
## 将C转换为MIR的编译器，类似于普通的C编译器

该项目的makefile构建了一个名为`c2m`的程序，可以编译在命令行上指定的C和MIR文件，并产生MIR代码或执行它：

- 编译器`c2m`具有与其他C编译器相同的选项`-E`、`-c`、`-S`和`-o`：
  - `-E`选项在预处理之后停止编译器，并将预处理后的文件输出到标准输出或选项`-o`后指定的文件中。
  - `-S`选项在生成MIR代码后停止编译器，并输出C源文件的MIR文本表示和具有`.bmir`后缀的二进制MIR文件。
  - `-c`选项在生成MIR代码后也停止编译器，并输出C源文件的MIR二进制表示和具有`.mir`后缀的文本MIR文件。
  - `-S`和`-c`选项的输出文件在当前目录中以源文件的名称创建，分别使用相应的后缀`.mir`和`.bmir`。
  - 如果只有一个源文件，还可以使用`-o`选项设置输出文件。
- 您可以通过使用`-s`选项和随后的字符串在命令行上提供C源代码。
- 您可以通过使用`-i`选项从标准输入读取C源代码。
- 如果未指定`-E`、`-c`或`-S`选项，所有生成的MIR代码将被链接，并检查是否存在`main`函数。整个生成的代码将作为二进制MIR文件`a.bmir`输出，或者作为`-o`选项指定的文件。
- 您可以通过使用`-ei`、`-eg`或`-el`选项来执行程序，而不是输出链接文件：
  - `-ei`表示通过MIR解释器执行代码。
  - `-eg`表示执行MIR生成器生成的机器代码。MIR生成器在解释器之前处理所有的MIR代码。
  - `-el`表示延迟代码生成。它类似于`-eg`，但函数代码在首次调用函数时生成。因此，对于从未使用的函数，不会生成机器代码。
  - 在`-ei`、`-eg`或`-el`选项之后的命令行参数不会被C转MIR编译器处理。这些参数将传递给生成和执行的MIR程序。
  - 执行的程序可以使用`libc`和`libm`库中的函数。它们始终可用。
    - `-lxxx`选项使得`libxxx`库在程序执行时可用。
    - `-Lxxx`选项将库目录`xxx`添加到由`-lxxx`选项指定的库的搜索路径中。搜索从标准库目录开始，并按照命令行上先前的`-L`选项的顺序在相应的目录中继续进行。
    - 要生成独立可执行文件，请参阅`mir-utils`目录中`b2ctab`实用程序的描述。
  - `-D`和`-U`选项与其他C编译器中用于命令行上的宏操作的选项类似。
  - `-I`选项用于添加包含目录，与其他C编译器类似。
  - `-fpreprocessed`选项表示跳过C文件的预处理过程。
  - `-fsyntax-only`选项表示在解析和语义检查C文件后停止，而不生成MIR代码。
  - `-w`选项表示关闭报告所有警告。
  - `-pedantic`选项用于更严格地检查C标准的一致性。这在C2MIR实现了一些GCC扩展时可能很有用。
  - `-O<n>`选项用于设置MIR生成器的优化级别。有关优化级别的描述，请参阅MIR生成器API函数`MIR_gen_set_optimize_level`的文档。
  - `-dg[<level>]`选项用于调试MIR生成器。根据调试级别，它会将有关MIR生成器工作的调试信息转储到`stderr`。如果省略级别，则表示最大级别。
  - 除了C文件外，还可以在命令行上指定具有`.mir`后缀的MIR文本文件和具有`.bmir`后缀的MIR二进制文件。在这种情况下，这些MIR文件将被读取并添加到生成的MIR代码中。
  - 这是有关编译器用法和执行C程序的简单示例：
```
	c2m -c part1.c && c2m -S part2.c && c2m part1.bmir part2.mir -eg # variant 1
	c2m part1.c part2.c && c2m a.bmir -eg                            # variant 2
	c2m part1.c part2.c -eg                                          # variant 3
```

## 将C编译器作为库的MIR编译器
该编译器可以作为库使用，并成为您程序的一部分。它可以从文件或内存中获取C代码。整个编译器的代码包含在文件`c2mir.c`中。其接口描述在文件`c2mir.h`中：
* 函数`c2mir_init(MIR_context ctx)`用于初始化编译器，以在上下文`ctx`中生成MIR代码。
* 函数`c2mir_finish(MIR_context ctx)`用于完成编译器在上下文`ctx`中的工作。它释放编译器在上下文`ctx`中使用的一些公共内存。
* 函数`c2mir_compile(MIR_context_t ctx, struct c2mir_options *ops, int (*getc_func)(void *), void *getc_data, const char *source_name, FILE *output_file)`用于编译一个C代码文件。函数在成功编译时返回true（非零）。它释放用于编译文件的所有内存。因此，您可以在同一上下文中编译许多文件而无需增加程序内存。函数`getc_func`提供对编译的C代码的访问，该代码可以在文件或内存中。该函数的每次调用都会将`getc_data`作为参数传递。诊断使用的源文件的名称由参数`source_name`给出。参数`output_file`类似于`c2m`的`-o`选项。参数`ops`是一个指向定义编译器选项的结构体的指针：
* 成员`message_file`定义报告错误和警告的位置。如果其值为NULL，则不会有任何输出。
* 成员`macro_commands_num`和`macro_commands`将编译器指示为`c2m`的`-D`和`-U`选项。
* 成员`include_dirs_num`和`include_dirs`将编译器指示为`-I`选项。
* 成员`debug_p`、`verbose_p`、`ignore_warnings_p`、`no_prepro_p`、`prepro_only_p`、`syntax_only_p`、`pedantic_p`、`asm_p`和`object_p`将编译器指示为`c2m`的`-d`、`-v`、`-w`、`-fpreprocessed`、`-E`、`-fsyntax-only`、`-pedantic`、`-S`和`-c`选项。如果`prepro_only_p`、`syntax_only_p`、`asm_p`和`object_p`的所有值都为零，则不会生成输出文件，只会将生成的MIR模块保留在上下文`ctx`的内存中。
* 成员`module_num`定义生成的MIR模块名称的索引（如果有的话）。
    
## Current state C to MIR compiler
  * **On Oct 25 we achieved a successful bootstrap**
    * `c2m` compiles own sources and generate binary MIR, this binary
      MIR compiles `c2m` sources again and generate another binary
      MIR, and the two binary MIR files are identical
    * The bootstrap test takes about CPU 10 sec (for comparison GCC minimal bootstrap takes about 2 CPU hours)    
  * ABI compliant calls (multiple return regs, passing structures through regs) has implemented for x86-64, ppc64,
    aarch64, and s390x
