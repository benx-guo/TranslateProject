---
status: translating
title: "Kconfig Language"
author: Linux Kernel Community
collector: tttturtle-russ
collected_date: 20240425
translator: benx-guo
translating_date: 20250917
priority: 10
link: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/kbuild/kconfig-language.rst
---

# Kconfig 语言

## 简介

配置数据库是一个由配置选项组成的树形结构：

    +- Code maturity level options
    |  +- Prompt for development and/or incomplete code/drivers
    +- General setup
    |  +- Networking support
    |  +- System V IPC
    |  +- BSD Process Accounting
    |  +- Sysctl support
    +- Loadable module support
    |  +- Enable loadable module support
    |     +- Set version information on all module symbols
    |     +- Kernel module loader
    +- ...

每个条目都有自己的依赖关系。这些依赖关系确定了该条目的可见性。
任何子条目只有在其父条目可见时才可见。

## 菜单条目

大多数条目会定义一个配置选项；所有其他条目用于帮助组织它们。
单个配置选项的定义如下：

    config MODVERSIONS
      bool "Set version information on all module symbols"
      depends on MODULES
      help
        Usually, modules have to be recompiled whenever you switch to a new
        kernel.  ...

每行都以关键字开头，后面可以跟多个参数。\"config\" 开始了一个新的配置条目。
后续的行定义该配置选项的属性。属性可以是配置选项的类型、输入提示、依赖关系、
帮助文本和默认值。一个配置选项可以用相同的名称定义多次，但每个定义只能有一个输入提示，
并且类型不能冲突。

## 菜单属性

一个菜单条目可以有多个属性。并非所有属性都适用于任意地方（参见语法）。

-   类型定义：\"bool\"/\"tristate\"/\"string\"/\"hex\"/\"int\"

    每个配置选项都必须指定类型。只有两种基本类型：tristate 和 string；
    其他类型都基于这两种。类型定义可以选择性地接受一个输入提示，因此以下两个示例是等价的：

        bool "Networking support"

    和:

        bool
        prompt "Networking support"

-   输入提示：\"prompt\" \<prompt\> \[\"if\" \<expr\>\]

    每个菜单条目最多只能有一个输入提示，用于向用户显示。
    可以选择性地使用 \"if\" 为该输入提示添加依赖关系（满足条件时显示）。

-   默认值：\"default\" \<expr\> \[\"if\" \<expr\>\]

    一个配置选项可以有任意数量的默认值。如果多个默认值可见，只有第一个定义的默认值有效。
    默认值并不局限于定义它的菜单条目中。这意味着默认值可以在其他地方定义，或被更早的定义覆盖。
    只有当用户尚未通过输入提示设置该配置符号时，默认值才会被赋予该配置符号。如果输入提示可见，
    默认值会显示给用户，并且可以被用户覆盖。可以选择性地使用 \"if\" 为该默认值添加依赖关系。

> 默认值被特意设置为 \'n\'，以避免构建体积膨胀。除了少数例外，新的配置选项不应该改变这一默认设置。
> 目的是让 \"make oldconfig\" 在不同版本之间尽可能少的向已有的配置中添加新内容。
>
> 注意：
>
> :   值得设置 \"default y/m\" 的情况包括：
>
>     a)  对于之前总是被构建的功能，新引入的 Kconfig 选项应该设置为 \"default y\"。
>     b)  作为总控开关（gatekeeping）的 Kconfig 选项（仅用于显示/隐藏其他选项，而自身不生成任何代码），
>         应该设置为 \"default y\"，以便用户能看到其他选项。
>     c)  对于一个 \"default n\" 驱动程序，其子驱动程序行为或类似选项可以被设置为默认启用。
>         以便该驱动启动时提供合理的默认值。
>     d)  每个人都期望的硬件或基础设施，例如 CONFIG_NET 或 CONFIG_BLOCK。
>         这些是罕见的例外。

-   类型定义 + 默认值：

        "def_bool"/"def_tristate" <expr> ["if" <expr>]

    这是类型定义加默认值的简写符号。可以选择性地使用 \"if\" 为该默认值添加依赖关系。

-   依赖关系：\"depends on\" \<expr\> ::Tag Here

    这为该菜单条目定义一个依赖关系。如果定义了多个依赖关系，它们通过 '&&' 连接。
    依赖关系应用于该菜单条目下的所有其他选项（也接受 \"if\" 表达式），
    因此以下两个示例是等价的：

        bool "foo" if BAR
        default y if BAR

    和:

        depends on BAR
        bool "foo"
        default y

-   反向依赖：\"select\" \<symbol\> \[\"if\" \<expr\>\]

    虽然普通依赖关系降低符号的上限（见下文），但反向依赖可用于强制另一个符号的下限。
    当前菜单符号的值用作 \<symbol\> 可以设置的最小值。如果 \<symbol\> 被选择多次，
    限制设置为最大的选择。反向依赖只能用于 bool 或 tristate。

    注意：

    ：select 应谨慎使用。select 会在不访问依赖关系的情况下强制将符号设为一个值。
    滥用 select 时，即使 FOO 依赖于 BAR（ BAR 没有启用），FOO 仍会被启用。
    一般来说，仅对不可见符号（任何地方都没有提示）和没有依赖关系的符号使用 select。
    虽然这样会限制其实用性，但可以避免出现大量非法配置。

-   弱反向依赖：\"imply\" \<symbol\> \[\"if\" \<expr\>\]

    这类似于 \"select\"，它对另一个符号强制执行下限，但 \"implied\" 符号的值
    仍然可以通过直接依赖关系或可见提示设置为 n。

    给定以下示例：

        config FOO
        tristate "foo"
        imply BAZ

        config BAZ
        tristate "baz"
        depends on BAR

    可能的值如下：

    > +---------------+---------------+-----------------+----------------+
    > | FOO           | BAR           | BAZ\'s default  | choice for BAZ |
    > +===============+===============+=================+================+
    > | n m y n m y y | y y y m m m n | n N/m/y m M/y/n |                |
    > |               |               | y Y/m/n n N/m m |                |
    > |               |               | M/n m M/n       |                |
    > |               |               | -   N           |                |
    > +---------------+---------------+-----------------+----------------+

    这在例如多个驱动程序希望指示它们能够挂钩到次要子系统，同时允许用户配置出该子系统而
    不必取消设置这些驱动程序时很有用。

    注意：如果 FOO=y 和 BAR=m 的组合导致链接错误，您可以使用 IS_REACHABLE() 保护函数调用：

        foo_init()
        {
            if (IS_REACHABLE(CONFIG_BAZ))
                baz_register(&foo);
            ...
        }

    注意：如果 BAZ 提供的功能对 FOO 非常有益，FOO 不仅应该 imply BAZ，还应该 imply
    其依赖项 BAR：

        config FOO
        tristate "foo"
        imply BAR
        imply BAZ

-   限制菜单显示：\"visible if\" \<expr\>

    此属性仅适用于菜单块，如果条件为假，菜单块不会显示给用户（但其中包含的符号仍然
    可以被其他符号选择）。它类似于单个菜单项的条件 \"prompt\" 属性。\"visible\"
    的默认值为 true。

-   数值范围：\"range\" \<symbol\> \<symbol\> \[\"if\"
    \<expr\>\]

    这允许限制 int 和 hex 符号的可能输入值范围。用户只能输入大于或等于第一个符号且
    小于或等于第二个符号的值。

-   帮助文本：\"help\"

    这定义帮助文本。帮助文本的结束由缩进级别确定，这意味着它在第一个缩进小于帮助文本
    第一行的行处结束。

-   模块属性："modules" 这声明符号用作 MODULES 符号，它为所有配置符号启用第三个模块化
    状态。最多一个符号可以设置 \"modules\"选项。

## 菜单依赖关系

依赖关系定义菜单项的可见性，也可以减少 tristate 符号的输入范围。表达式中使用的三态逻辑比普通
 boolean 逻辑多一个状态来表示模块状态。依赖表达式具有以下语法：

    <expr> ::= <symbol>                           (1)
             <symbol> '=' <symbol>                (2)
             <symbol> '!=' <symbol>               (3)
             <symbol1> '<' <symbol2>              (4)
             <symbol1> '>' <symbol2>              (4)
             <symbol1> '<=' <symbol2>             (4)
             <symbol1> '>=' <symbol2>             (4)
             '(' <expr> ')'                       (5)
             '!' <expr>                           (6)
             <expr> '&&' <expr>                   (7)
             <expr> '||' <expr>                   (8)

表达式按优先级递减顺序列出。

(1) 将符号转换为表达式。bool 和 tristate 简单地转换为相应的表达式值。所有其他符号类型的
    结果为 'n'。

(2) 如果两个符号的值相等，返回 'y'，否则返回 'n'。

(3) 如果两个符号的值相等，返回 'n'，否则返回 'y'。

(4) 如果 \<symbol1\> 的值分别小于、大于、小于等于或大于等于 \<symbol2\> 的值，
    返回 \'y\'，否则返回 \'n\'。

(5) 返回表达式的值。用于覆盖优先级。

(6) 返回 (2-/expr/) 的结果。

(7) 返回 min(/expr/, /expr/) 的结果。

(8) 返回 max(/expr/, /expr/) 的结果。

表达式可以具有 \'n\'，\'m\' 或 \'y\' 的值（或分别为 0，1，2 用于计算）。
当菜单项的表达式求值为 \'m\' 或 \'y\' 时，该菜单项变为可见。

有两种类型的符号：常量符号和非常量符号。非常量符号是最常见的，使用 \'config\' 语句定义。
非常量符号完全由字母数字字符或下划线组成。常量符号只是表达式的一部分。常量符号总是被单引号或
双引号包围。在引号内，允许任何其他字符，引号可以使用 \'\' 转义。

## 菜单结构

菜单项在树中的位置通过两种方式确定。首先可以明确指定：

    menu "Network device support"
      depends on NET

    config NETDEVICES
      ...

    endmenu

\"menu\" \... \"endmenu\" 块内的所有项成为 \"Network device support\" 的子菜单。
所有子项从菜单项继承依赖关系，例如这意味着依赖关系 \"NET\" 被添加到配置选项 NETDEVICES 
的依赖列表中。

生成菜单结构的另一种方法是通过分析依赖关系来完成的。如果一个菜单项以某种方式依赖于前一个
项，它可以成为其子菜单。首先，前一个（父）符号必须是依赖列表的一部分，然后以下两个条件之一
必须为真：

-   如果父级设置为 \'n\'，子项必须变为不可见

-   子项只有在父级可见时才可见：

        config MODULES
        bool "Enable loadable module support"

        config MODVERSIONS
        bool "Set version information on all module symbols"
        depends on MODULES

        comment "module support disabled"
        depends on !MODULES

MODVERSIONS 直接依赖于 MODULES，这意味着它只有在 MODULES 不是 \'n\' 时才可见。
另一方面，提示文字仅在 MODULES 设置为 \'n\' 时可见。

## Kconfig 语法

配置文件描述一系列菜单项，每一行以一个关键字开头（帮助文本除外）。以下关键字
结束一个菜单项：

-   config
-   menuconfig
-   choice/endchoice
-   comment
-   menu/endmenu
-   if/endif
-   source

前五种关键字也都用于开始定义一个菜单项。

config:

    "config" <symbol>
    <config options>

这定义一个配置符号 \<symbol\> 并接受上述任意属性作为选项。

menuconfig:

    "menuconfig" <symbol>
    <config options>

这类似于上面的简单配置项，但它还向前端提示，所有子选项应该显示为单独的选项列表。为了确保
所有子选项真正显示在 menuconfig 项下而不是在其外部，\<config options\> 列表中的每个
项目都必须依赖于 menuconfig 符号。实际上，这是通过使用以下两种构造之一来实现的：

    (1):
    menuconfig M
    if M
        config C1
        config C2
    endif

    (2):
    menuconfig M
    config C1
        depends on M
    config C2
        depends on M

在以下示例 (3) 和 (4) 中，C1 和 C2 仍然具有 M 依赖关系，但由于 C0 不依赖于 M，
它们不会再出现在 menuconfig M 下：

    (3):
    menuconfig M
        config C0
    if M
        config C1
        config C2
    endif

    (4):
    menuconfig M
    config C0
    config C1
        depends on M
    config C2
        depends on M

choices:

    "choice"
    <choice options>
    <choice block>
    "endchoice"

这定义一个选择组并接受上述任何属性作为选项。选择只能是 bool 或 tristate 类型。
如果没有为选择指定类型，其类型将由组中第一个选择元素的类型确定，或者如果没有任何选择
元素指定类型，则保持未知。

虽然 bool 选择只允许选择单个配置项，但 tristate 选择还允许将任意数量的配置项设置为 \'m\'。
如果单个硬件存在多个驱动程序，并且只有一个驱动程序可以编译/加载到内核中，但所有驱动程序
都可以编译为模块，则可以使用此功能。

选择接受另一个选项 \"optional\"，它允许将选择设置为 \'n\'，并且不需要选择任何项。

comment:

    "comment" <prompt>
    <comment options>

这定义在配置过程中向用户显示的提示文字，并且也回显到输出文件。唯一可能的选项是依赖关系。

menu:

    "menu" <prompt>
    <menu options>
    <menu block>
    "endmenu"

这定义一个菜单块，有关更多信息，请参见上面的 \"Menu structure\"。唯一可能的选项是
依赖关系和 \"visible\" 属性。

if:

    "if" <expr>
    <if block>
    "endif"

这定义一个 if 块。依赖表达式 \<expr\> 被附加到所有包含的菜单项。

source:

    "source" <prompt>

读取指定的配置文件。该文件始终被解析。

mainmenu:

    "mainmenu" <prompt>

如果配置程序选择使用它，这将设置配置程序的标题栏。它应该放在配置的顶部，在任何其他语句之前。

\'#\' Kconfig 源文件注释：

源文件行中任何位置的未加引号的 \'#\' 字符表示源文件注释的开始。该行的其余部分是注释。

## Kconfig 提示

这是 Kconfig 技巧的集合，其中大多数乍一看并不明显，而且大多数已成为多个 Kconfig 
文件中的惯用法。

### 添加通用功能并且使用法可配置

实现与某些架构相关但并非所有架构相关的功能/特性是一种常见的惯用法。推荐的做法是使用名为
[HAVE]()\* 的配置变量，该变量在公共 Kconfig 文件中定义并由相关架构选择。一个例子是
通用 IOMAP 功能。

我们会在 lib/Kconfig 中看到：

    # Generic IOMAP is used to ...
    config HAVE_GENERIC_IOMAP

    config GENERIC_IOMAP
      depends on HAVE_GENERIC_IOMAP && FOO

在 lib/Makefile 中我们会看到：

    obj-$(CONFIG_GENERIC_IOMAP) += iomap.o

对于使用通用 IOMAP 功能的每个架构，我们会看到：

    config X86
      select ...
      select HAVE_GENERIC_IOMAP
      select ...

注意：我们使用现有的配置选项并避免创建新的配置变量来选择 HAVE_GENERIC_IOMAP。

注意：使用内部配置变量 HAVE_GENERIC_IOMAP 是为了突破 select 的限制，它会强制将配置
选项设置为 'y'，而不管依赖关系如何。依赖关系被移动到符号 GENERIC_IOMAP，我们避免了 
select 强制将符号设置为 'y' 的情况。

### 添加需要编译器支持的功能

有几个功能需要编译器支持。描述对编译器功能依赖的推荐方法是使用 \"depends on\" 后跟测试宏：

    config STACKPROTECTOR
      bool "Stack Protector buffer overflow detection"
      depends on $(cc-option,-fstack-protector)
      ...

如果您需要向 makefiles 和/或 C 源文件公开编译器能力，[CC_HAS\_]{.title-ref} 是配置
选项的推荐前缀：

    config CC_HAS_FOO
      def_bool $(success,$(srctree)/scripts/cc-check-foo.sh $(CC))

### 仅作为模块构建

要限制组件构建仅作为模块，请用 \"depends on m\" 限定其配置符号。例如：

    config FOO
      depends on BAR && m

将 FOO 限制为模块 (=m) 或禁用 (=n)。

### 编译测试

如果配置符号具有依赖关系，但即使不满足依赖关系，由配置符号控制的代码仍然可以编译，则建议
通过向依赖关系添加 \"\|\| COMPILE_TEST\" 子句来增加构建覆盖率。这对于更奇特硬件的
驱动程序特别有用，因为它允许持续集成系统在更常见的系统上编译测试代码，并以这种方式检测错误。
请注意，编译测试的代码在不满足依赖关系的系统上运行时应避免崩溃。

### 架构和平台依赖关系

由于存在桩代码，大多数驱动程序现在可以在大多数架构上编译。但是，这并不意味着在所有地方都
提供所有驱动程序是合理的，因为实际硬件可能仅存在于特定架构和平台上。对于片上系统 IP 核
尤其如此，它们可能仅限于特定供应商或 SoC 系列。

为了防止向用户询问在用户正在编译内核的系统上无法使用的驱动程序，并且如果有意义，
控制驱动程序编译的配置符号应包含适当的依赖关系，将符号的可见性限制为驱动程序可用的平台
（的超集）。依赖关系可以是架构（例如 ARM）或平台（例如ARCH_OMAP4）依赖关系。这不仅使
发行版配置所有者的生活更简单，而且对于配置内核的每个开发人员或用户也是如此。

这样的依赖关系可以通过与上面的编译测试规则结合来放宽，从而导致：

> config FOO
>
> :   bool \"Support for foo hardware\" depends on ARCH_FOO_VENDOR \|\|
>     COMPILE_TEST

### 可选依赖关系

某些驱动程序能够选择性地使用另一个模块的功能或在禁用该模块的情况下干净地构建，但在尝试从
内置驱动程序使用该可加载模块时会导致链接失败。

在 Kconfig 逻辑中表达此可选依赖关系的最常见方法使用稍微违反直觉的方式：

    config FOO
      tristate "Support for foo hardware"
      depends on BAR || !BAR

这意味着要么存在对 BAR 的依赖关系，不允许 FOO=y 与 BAR=m 的组合，要么 BAR 完全禁用。
对于有多个驱动程序具有相同依赖关系的更正式的方法，可以使用辅助符号，例如：

    config FOO
      tristate "Support for foo hardware"
      depends on BAR_OPTIONAL

    config BAR_OPTIONAL
      def_tristate BAR || !BAR

### Kconfig 递归依赖限制

如果您遇到 Kconfig 错误："检测到递归依赖"，则您遇到了 Kconfig 的递归依赖问题，递归依赖
可以总结为循环依赖。kconfig 工具需要确保 Kconfig 文件符合指定的配置要求。为了做到这一点，
kconfig 必须确定所有 Kconfig 符号可能的值，如果两个或更多 Kconfig 符号之间存在循环关系，
目前无法做到这一点。有关更多详细信息，请参阅下面的 "简单的 Kconfig 递归问题"小节。
Kconfig 不进行递归依赖解析；这对 Kconfig 文件编写者有一些影响。我们将首先解释为什么存在
此问题，然后提供一个示例技术限制，这给 Kconfig 开发人员带来了影响。希望尝试解决此限制的
热心开发人员应阅读下一小节。

### 简单的 Kconfig 递归问题

阅读：Documentation/kbuild/Kconfig.recursion-issue-01

测试方法：

    make KBUILD_KCONFIG=Documentation/kbuild/Kconfig.recursion-issue-01 allnoconfig

### 累积的 Kconfig 递归问题

阅读：Documentation/kbuild/Kconfig.recursion-issue-02

测试方法：

    make KBUILD_KCONFIG=Documentation/kbuild/Kconfig.recursion-issue-02 allnoconfig

### kconfig 递归问题的实用解决方案

遇到递归 Kconfig 问题的开发人员有两个选择。我们在下面记录它们，并提供
通过这些不同解决方案解决的历史问题列表。

> a)  删除任何多余的 \"select FOO\" 或 \"depends on FOO\"
>
> b)  匹配依赖语义：
>
> > b1) 将所有 \"select FOO\" 交换为 \"depends on FOO\"，或
> >
> > b2) 将所有 \"depends on FOO\" 交换为 \"select FOO\"
> >
a) 的解决方案可以通过示例 Kconfig 文件 
Documentation/kbuild/Kconfig.recursion-issue-01 测试，通过从 CORE_BELL_A_ADVANCED 
中删除 \"select CORE\"，因为由于 CORE_BELL_A 依赖于 CORE，这已经是隐含的。
有时可能无法删除某些依赖标准，对于这种情况，您可以使用解决方案 b)。

b) 的两种不同解决方案可以在示例 Kconfig 文件 
Documentation/kbuild/Kconfig.recursion-issue-02 中测试。

以下是这些类型递归问题的先前修复示例列表；所有错误似乎都涉及一个或多个 \"select\" 语句和
一个或多个 \"depends on\"。

  commit         fix
  -------------- -------------------------------
  06b718c01208   select A -\> depends on A
  c22eacfe82f9   depends on A -\> depends on B
  6a91e854442c   select A -\> depends on A
  118c565a8f2e   select A -\> select B
  f004e5594705   select A -\> depends on A
  c7861f37b4c6   depends on A -\> (null)
  80c69915e5fb   select A -\> (null) (1)
  c2218e26c0d0   select A -\> depends on A (1)
  d6ae99d04e1c   select A -\> depends on A
  95ca19cf8cbf   select A -\> depends on A
  8f057d7bca54   depends on A -\> (null)
  8f057d7bca54   depends on A -\> select A
  a0701f04846e   select A -\> depends on A
  0c8b92f7f259   depends on A -\> (null)
  e4e9e0540928   select A -\> depends on A (2)
  7453ea886e87   depends on A \> (null) (1)
  7b1fff7e4fdf   select A -\> depends on A
  86c747d2a4f0   select A -\> depends on A
  d9f9ab51e55e   select A -\> depends on A
  0c51a4d8abd6   depends on A -\> select A (3)
  e98062ed6dc4   select A -\> depends on A (3)
  91e5d284a7f1   select A -\> (null)

(1) 部分（或没有）错误引用。

(2) 这似乎是该修复的要点。

(3) 相同错误。

### 未来的 kconfig 工作

欢迎在澄清语义和评估使用完整 SAT 解析器方面对 kconfig 进行工作。完整的 SAT 解析器可能是
理想的，以启用更复杂的依赖映射和/或查询，例如 SAT 解析器的一个可能用例可能是处理当前已知的
递归依赖问题。目前尚不清楚这是否会解决此类问题，但这种评估是理想的。如果对完整 SAT 解析器的
支持证明过于复杂或无法解决递归依赖问题，Kconfig 至少应该具有清晰且定义明确的语义，这些语义
还应解决和记录限制或要求，例如处理递归依赖的限制或要求。

欢迎在 Kconfig 上对这两个领域进行进一步的工作。我们在接下来的两个小节中详细说明这两个方面。

### Kconfig 的语义

Kconfig 的使用范围很广，Linux 现在只是 Kconfig 的用户之一：一项研究完成了对 12 个项目中
Kconfig 使用的广泛分析[^1]。尽管它被广泛使用，尽管本文档在记录基本 Kconfig 语法方面做得
相当好，但仍欢迎更精确的 Kconfig 语义定义。一个项目通过使用 xconfig 配置器推导出 Kconfig 
语义[^2]。应该进行工作以确认推导出的语义是否与我们预期的 Kconfig 设计目标相匹配。另一个项目
形式化了 Kconfig 语言核心子集的指称语义[^3]。

具有明确定义的语义对于实际评估依赖关系的工具很有用，例如，一个这样的案例是在 Kconfig 推断
语义的布尔抽象中表达，将 Kconfig逻辑转换为布尔公式，并在此基础上运行 SAT 解析器以查找
死代码/功能（始终不活动），使用此方法在 Linux 中发现了 114 个死功能[^4]（第 8 节：有效性威胁）。
基于[^5]中语义的 kismet 工具发现反向依赖的滥用，并导致对 Linux Kconfig 文件进行了数十次
提交的修复[^6]。

确认这一点可能会很有用，因为 Kconfig 是领先的工业可变性建模语言之一[^7][^8]。其研究将有助于
评估此类语言的实际用途，它们的使用仅是理论上的，现实世界的需求尚未得到充分理解。然而，目前只有
逆向工程技术被用来从可变性建模语言（如 Kconfig）中推导语义[^9]。

### Kconfig 的完整 SAT 解析器

尽管如前一小节中所述，Kconfig 尚未直接使用 SAT 解析器[^10]，但已经完成了在布尔抽象中
表达 Kconfig 的推断语义以将 Kconfig 逻辑转换为布尔公式并在其上运行 SAT 解析器的工作[^11]。
另一个已知的相关项目是CADOS[^12]（前身为 VAMOS[^13]）及其工具，主要是 undertaker[^14]，
它首次在[^15]中引入。undertaker 的基本概念是从 Kconfig 中提取可变性模型，并将它们与从 CPP
#ifdefs 和构建规则中提取的命题公式一起放入 SAT 解析器，以便查找死代码、死文件和死符号。
如果在 Kconfig 上使用 SAT 解析器是理想的，一种方法是评估以某种方式在 Kconfig 上重新利用
此类工作。现有项目的导师有足够的兴趣不仅帮助建议如何将这项工作集成到上游，而且还帮助长期维护它。
感兴趣的开发人员应该访问：

<https://kernelnewbies.org/KernelProjects/kconfig-sat>

[^1]: <https://www.eng.uwaterloo.ca/~shshe/kconfig_semantics.pdf>

[^2]: <https://gsd.uwaterloo.ca/sites/default/files/vm-2013-berger.pdf>

[^3]: <https://paulgazzillo.com/papers/esecfse21.pdf>

[^4]: <https://gsd.uwaterloo.ca/sites/default/files/vm-2013-berger.pdf>

[^5]: <https://paulgazzillo.com/papers/esecfse21.pdf>

[^6]: <https://github.com/paulgazz/kmax>

[^7]: <https://gsd.uwaterloo.ca/sites/default/files/vm-2013-berger.pdf>

[^8]: <https://gsd.uwaterloo.ca/sites/default/files/ase241-berger_0.pdf>

[^9]: <https://gsd.uwaterloo.ca/sites/default/files/icse2011.pdf>

[^10]: <https://www.cs.cornell.edu/~sabhar/chapters/SATSolvers-KR-Handbook.pdf>

[^11]: <https://gsd.uwaterloo.ca/sites/default/files/vm-2013-berger.pdf>

[^12]: <https://cados.cs.fau.de>

[^13]: <https://vamos.cs.fau.de>

[^14]: <https://undertaker.cs.fau.de>

[^15]: <https://www4.cs.fau.de/Publications/2011/tartler_11_eurosys.pdf>
