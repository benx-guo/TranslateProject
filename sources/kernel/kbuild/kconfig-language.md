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

配置数据库是以树状结构组织的配置项集合：

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

每个配置项都有自己的依赖关系。这些依赖关系用于确定配置项的可见性。
任何子项只有在其父项可见时才可见。

## TODO:菜单项

Most entries define a config option; all other entries help to organize
them. A single configuration option is defined like this:

TODO: 大多数配置项会定义一个可配置项；所有其他配置项用于帮助组织它们。
单个配置选项的定义如下：

    config MODVERSIONS
      bool "Set version information on all module symbols"
      depends on MODULES
      help
        Usually, modules have to be recompiled whenever you switch to a new
        kernel.  ...

Every line starts with a key word and can be followed by multiple
arguments. \"config\" starts a new config entry. The following lines
define attributes for this config option. Attributes can be the type of
the config option, input prompt, dependencies, help text and default
values. A config option can be defined multiple times with the same
name, but every definition can have only a single input prompt and the
type must not conflict.

TODO: 每行都以一个关键字开头，后面可以跟多个参数。\"config\" 开始一个新的配置项。
后续行定义该配置选项的属性。属性可以是配置选项的类型、输入提示、依赖关系、
帮助文本和默认值。一个配置选项可以用相同的名称定义多次，但每个定义只能有一个输入提示，
并且类型不能冲突。

## 菜单属性

A menu entry can have a number of attributes. Not all of them are
applicable everywhere (see syntax).

一个配置项可以有多个属性。并非所有属性都适用于任何地方（参见语法）。

-   类型定义：\"bool\"/\"tristate\"/\"string\"/\"hex\"/\"int\"

    每个配置选项都必须有一个类型。只有两种基本类型：tristate 和 string；其他类型都基于这两种。
    类型定义可以选择性地接受一个输入提示，因此以下两个示例是等价的：

        bool "Networking support"

    and:

        bool
        prompt "Networking support"

-   输入提示：\"prompt\" \<prompt\> \[\"if\" \<expr\>\]

    Every menu entry can have at most one prompt, which is used to
    display to the user. Optionally dependencies only for this prompt
    can be added with \"if\".

    每个菜单项最多只能有一个输入提示，用于向用户显示。可以选择性地使用 \"if\" 为该输入提示
    添加依赖关系（满足条件时显示）。

-   默认值：\"default\" \<expr\> \[\"if\" \<expr\>\]

    一个配置选项可以有任意数量的默认值。如果多个默认值可见，只有第一个定义的默认值有效。
    默认值不限于它们被定义的菜单项。这意味着默认值可以在其他地方定义，或被更早的定义覆盖。
    默认值只有在用户没有通过（上面的输入提示）设置其他值时才会分配给配置符号。如果输入
    提示可见，默认值会呈现给用户，并且可以被用户覆盖。可以选择性地使用 \"if\" 为该默认值添加
    依赖关系。

> 默认值被刻意设置为 \'n\'，以避免构建膨胀。除了少数例外，新的配置选项不应该改变这一默认行为。
> 目的是让 \"make oldconfig\" 在每次发布时尽可能少地向配置中添加内容。
>
> 注意：
>
> :   值得设置 \"default y/m\" 的情况包括：
>
>     a)  对于以前总是被构建的东西，新的 Kconfig 选项应该是 \"default y\"。
>     b)  新的总控型 Kconfig 选项，控制其他 Kconfig 选项的隐藏/显示(但自身不生成任何代码),
>         应该是 \"default y\"，以便用户能看到其他选项。
>     c)  对于一个 \"default n\" 驱动程序，其子驱动程序行为或类似选项可以被设置为默认启用。
>         以便该驱动启动时提供合理的默认值。
>     d)  每个人都期望的硬件或基础设施，例如 CONFIG_NET 或 CONFIG_BLOCK。
>         这些是罕见的例外。

-   类型定义 + 默认值：

        "def_bool"/"def_tristate" <expr> ["if" <expr>]

    这是类型定义加默认值的简写符号。可以选择性地使用 \"if\" 为该默认值添加依赖关系。

-   依赖关系：\"depends on\" \<expr\>

    This defines a dependency for this menu entry. If multiple
    dependencies are defined, they are connected with \'&&\'.
    Dependencies are applied to all other options within this menu entry
    (which also accept an \"if\" expression), so these two examples are
    equivalent:

    这为该菜单项定义一个依赖关系。如果定义了多个依赖关系，它们通过 '&&' 连接。
    依赖关系应用于该菜单项内的所有其他选项（也接受 \"if\" 表达式），因此以下两个示例是
    等价的：

        bool "foo" if BAR
        default y if BAR

    and:

        depends on BAR
        bool "foo"
        default y

-   reverse dependencies: \"select\" \<symbol\> \[\"if\" \<expr\>\]

    反向依赖："select" \<symbol\> \["if" \<expr\>\]

    While normal dependencies reduce the upper limit of a symbol (see
    below), reverse dependencies can be used to force a lower limit of
    another symbol. The value of the current menu symbol is used as the
    minimal value \<symbol\> can be set to. If \<symbol\> is selected
    multiple times, the limit is set to the largest selection. Reverse
    dependencies can only be used with boolean or tristate symbols.

    虽然普通依赖关系降低符号的上限
    （见下文），但反向依赖可用于
    强制另一个符号的下限。
    当前菜单符号的值用作 \<symbol\>
    可以设置的最小值。如果 \<symbol\>
    被选择多次，限制设置为最大的
    选择。反向依赖只能用于布尔或
    三态符号。

    Note:

    注意：

    :   select should be used with care. select will force a symbol to a
        value without visiting the dependencies. By abusing select you
        are able to select a symbol FOO even if FOO depends on BAR that
        is not set. In general use select only for non-visible symbols
        (no prompts anywhere) and for symbols with no dependencies. That
        will limit the usefulness but on the other hand avoid the
        illegal configurations all over.

        select 应谨慎使用。select 会在
        不访问依赖关系的情况下强制将
        符号设为一个值。通过滥用
        select，即使 FOO 依赖于
        未设置的 BAR，您也能够选择
        符号 FOO。一般来说，仅对
        不可见符号（任何地方都没有提示）
        和没有依赖关系的符号使用 select。
        这会限制其实用性，但另一方面
        可以避免到处出现非法配置。

-   weak reverse dependencies: \"imply\" \<symbol\> \[\"if\" \<expr\>\]

    弱反向依赖："imply" \<symbol\> \["if" \<expr\>\]

    This is similar to \"select\" as it enforces a lower limit on
    another symbol except that the \"implied\" symbol\'s value may still
    be set to n from a direct dependency or with a visible prompt.

    这类似于 \"select\"，它对另一个
    符号强制执行下限，但 \"implied\"
    符号的值仍然可以通过直接依赖关系
    或可见提示设置为 n。

    Given the following example:

    给定以下示例：

        config FOO
        tristate "foo"
        imply BAZ

        config BAZ
        tristate "baz"
        depends on BAR

    The following values are possible:

    可能的值如下：

    > +---------------+---------------+-----------------+----------------+
    > | FOO           | BAR           | BAZ\'s default  | choice for BAZ |
    > +===============+===============+=================+================+
    > | n m y n m y y | y y y m m m n | n N/m/y m M/y/n |                |
    > |               |               | y Y/m/n n N/m m |                |
    > |               |               | M/n m M/n       |                |
    > |               |               | -   N           |                |
    > +---------------+---------------+-----------------+----------------+

    This is useful e.g. with multiple drivers that want to indicate
    their ability to hook into a secondary subsystem while allowing the
    user to configure that subsystem out without also having to unset
    these drivers.

    这在例如多个驱动程序希望指示
    它们能够挂钩到次要子系统，
    同时允许用户配置出该子系统而
    不必取消设置这些驱动程序时
    很有用。

    Note: If the combination of FOO=y and BAR=m causes a link error, you
    can guard the function call with IS_REACHABLE():

    注意：如果 FOO=y 和 BAR=m 的
    组合导致链接错误，您可以使用
    IS_REACHABLE() 保护函数调用：

        foo_init()
        {
            if (IS_REACHABLE(CONFIG_BAZ))
                baz_register(&foo);
            ...
        }

    Note: If the feature provided by BAZ is highly desirable for FOO,
    FOO should imply not only BAZ, but also its dependency BAR:

    注意：如果 BAZ 提供的功能对
    FOO 非常有益，FOO 不仅应该
    imply BAZ，还应该 imply
    其依赖项 BAR：

        config FOO
        tristate "foo"
        imply BAR
        imply BAZ

-   limiting menu display: \"visible if\" \<expr\>

    限制菜单显示："visible if" \<expr\>

    This attribute is only applicable to menu blocks, if the condition
    is false, the menu block is not displayed to the user (the symbols
    contained there can still be selected by other symbols, though). It
    is similar to a conditional \"prompt\" attribute for individual menu
    entries. Default value of \"visible\" is true.

    此属性仅适用于菜单块，
    如果条件为假，菜单块不会显示
    给用户（但其中包含的符号仍然
    可以被其他符号选择）。
    它类似于单个菜单项的条件
    \"prompt\" 属性。\"visible\"
    的默认值为 true。

-   numerical ranges: \"range\" \<symbol\> \<symbol\> \[\"if\"
    \<expr\>\]

    数值范围："range" \<symbol\> \<symbol\> \["if" \<expr\>\]

    This allows to limit the range of possible input values for int and
    hex symbols. The user can only input a value which is larger than or
    equal to the first symbol and smaller than or equal to the second
    symbol.

    这允许限制 int 和 hex 符号的
    可能输入值范围。用户只能输入
    大于或等于第一个符号且小于或
    等于第二个符号的值。

-   help text: \"help\"

    帮助文本："help"

    This defines a help text. The end of the help text is determined by
    the indentation level, this means it ends at the first line which
    has a smaller indentation than the first line of the help text.

    这定义帮助文本。帮助文本的结束
    由缩进级别确定，这意味着它在
    第一个缩进小于帮助文本第一行的
    行处结束。

-   module attribute: \"modules\" This declares the symbol to be used as
    the MODULES symbol, which enables the third modular state for all
    config symbols. At most one symbol may have the \"modules\" option
    set.

    模块属性："modules" 这声明符号
    用作 MODULES 符号，它为所有
    配置符号启用第三个模块化状态。
    最多一个符号可以设置 \"modules\"
    选项。

## Menu dependencies

## 菜单依赖关系

Dependencies define the visibility of a menu entry and can also reduce
the input range of tristate symbols. The tristate logic used in the
expressions uses one more state than normal boolean logic to express the
module state. Dependency expressions have the following syntax:

依赖关系定义菜单项的可见性，
也可以减少三态符号的输入范围。
表达式中使用的三态逻辑比普通
布尔逻辑多一个状态来表示模块
状态。依赖表达式具有以下语法：

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

Expressions are listed in decreasing order of precedence.

表达式按优先级递减顺序列出。

(1) Convert the symbol into an expression. Boolean and tristate symbols
    are simply converted into the respective expression values. All
    other symbol types result in \'n\'.

    将符号转换为表达式。布尔和
    三态符号简单地转换为相应的
    表达式值。所有其他符号类型的
    结果为 'n'。

(2) If the values of both symbols are equal, it returns \'y\', otherwise
    \'n\'.

    如果两个符号的值相等，返回 'y'，否则返回 'n'。

(3) If the values of both symbols are equal, it returns \'n\', otherwise
    \'y\'.

    如果两个符号的值相等，返回 'n'，否则返回 'y'。

(4) If value of \<symbol1\> is respectively lower, greater,
    lower-or-equal, or greater-or-equal than value of \<symbol2\>, it
    returns \'y\', otherwise \'n\'.

    如果 \<symbol1\> 的值分别低于、
    大于、小于或等于或大于或等于
    \<symbol2\> 的值，返回 'y'，
    否则返回 'n'。

(5) Returns the value of the expression. Used to override precedence.

    返回表达式的值。用于覆盖优先级。

(6) Returns the result of (2-/expr/).

    返回 (2-/expr/) 的结果。

(7) Returns the result of min(/expr/, /expr/).

    返回 min(/expr/, /expr/) 的结果。

(8) Returns the result of max(/expr/, /expr/).

    返回 max(/expr/, /expr/) 的结果。

An expression can have a value of \'n\', \'m\' or \'y\' (or 0, 1, 2
respectively for calculations). A menu entry becomes visible when its
expression evaluates to \'m\' or \'y\'.

表达式可以具有 'n'、'm' 或 'y'
的值（或分别为 0、1、2 用于
计算）。当菜单项的表达式求值为
'm' 或 'y' 时，该菜单项变为
可见。

There are two types of symbols: constant and non-constant symbols.
Non-constant symbols are the most common ones and are defined with the
\'config\' statement. Non-constant symbols consist entirely of
alphanumeric characters or underscores. Constant symbols are only part
of expressions. Constant symbols are always surrounded by single or
double quotes. Within the quote, any other character is allowed and the
quotes can be escaped using \'\'.

有两种类型的符号：常量符号和
非常量符号。非常量符号是最常见的，
使用 'config' 语句定义。
非常量符号完全由字母数字字符或
下划线组成。常量符号只是表达式的
一部分。常量符号总是被单引号或
双引号包围。在引号内，允许任何
其他字符，引号可以使用 '' 转义。

## Menu structure

## 菜单结构

The position of a menu entry in the tree is determined in two ways.
First it can be specified explicitly:

菜单项在树中的位置通过两种
方式确定。首先可以明确指定：

    menu "Network device support"
      depends on NET

    config NETDEVICES
      ...

    endmenu

All entries within the \"menu\" \... \"endmenu\" block become a submenu
of \"Network device support\". All subentries inherit the dependencies
from the menu entry, e.g. this means the dependency \"NET\" is added to
the dependency list of the config option NETDEVICES.

\"menu\" ... \"endmenu\" 块内的
所有项成为 \"Network device
support\" 的子菜单。所有子项从
菜单项继承依赖关系，例如这意味着
依赖关系 \"NET\" 被添加到配置
选项 NETDEVICES 的依赖列表中。

The other way to generate the menu structure is done by analyzing the
dependencies. If a menu entry somehow depends on the previous entry, it
can be made a submenu of it. First, the previous (parent) symbol must be
part of the dependency list and then one of these two conditions must be
true:

生成菜单结构的另一种方法是通过
分析依赖关系来完成的。如果一个
菜单项以某种方式依赖于前一个
项，它可以成为其子菜单。首先，
前一个（父）符号必须是依赖列表的
一部分，然后以下两个条件之一
必须为真：

-   the child entry must become invisible, if the parent is set to \'n\'

    如果父级设置为 'n'，子项必须变为不可见

-   the child entry must only be visible, if the parent is visible:

    子项只有在父级可见时才可见：

        config MODULES
        bool "Enable loadable module support"

        config MODVERSIONS
        bool "Set version information on all module symbols"
        depends on MODULES

        comment "module support disabled"
        depends on !MODULES

MODVERSIONS directly depends on MODULES, this means it\'s only visible
if MODULES is different from \'n\'. The comment on the other hand is
only visible when MODULES is set to \'n\'.

MODVERSIONS 直接依赖于 MODULES，
这意味着它只有在 MODULES 不是 'n'
时才可见。另一方面，注释仅在
MODULES 设置为 'n' 时可见。

## Kconfig syntax

## Kconfig 语法

The configuration file describes a series of menu entries, where every
line starts with a keyword (except help texts). The following keywords
end a menu entry:

配置文件描述一系列菜单项，
每一行以一个关键字开头
（帮助文本除外）。以下关键字
结束一个菜单项：

-   config
-   menuconfig
-   choice/endchoice
-   comment
-   menu/endmenu
-   if/endif
-   source

The first five also start the definition of a menu entry.

前五个关键字也开始菜单项的定义。

config:

    "config" <symbol>
    <config options>

This defines a config symbol \<symbol\> and accepts any of above
attributes as options.

这定义一个配置符号 \<symbol\>
并接受上述任何属性作为选项。

menuconfig:

    "menuconfig" <symbol>
    <config options>

This is similar to the simple config entry above, but it also gives a
hint to front ends, that all suboptions should be displayed as a
separate list of options. To make sure all the suboptions will really
show up under the menuconfig entry and not outside of it, every item
from the \<config options\> list must depend on the menuconfig symbol.
In practice, this is achieved by using one of the next two constructs:

这类似于上面的简单配置项，
但它还向前端提示，所有子选项应该
显示为单独的选项列表。为了确保
所有子选项真正显示在 menuconfig
项下而不是在其外部，
\<config options\> 列表中的每个
项目都必须依赖于 menuconfig 符号。
实际上，这是通过使用以下两种
构造之一来实现的：

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

In the following examples (3) and (4), C1 and C2 still have the M
dependency, but will not appear under menuconfig M anymore, because of
C0, which doesn\'t depend on M:

在以下示例 (3) 和 (4) 中，C1 和
C2 仍然具有 M 依赖关系，但由于
C0 不依赖于 M，它们不会再出现在
menuconfig M 下：

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

This defines a choice group and accepts any of the above attributes as
options. A choice can only be of type bool or tristate. If no type is
specified for a choice, its type will be determined by the type of the
first choice element in the group or remain unknown if none of the
choice elements have a type specified, as well.

这定义一个选择组并接受上述任何
属性作为选项。选择只能是 bool 或
tristate 类型。如果没有为选择
指定类型，其类型将由组中第一个
选择元素的类型确定，或者如果没有
任何选择元素指定类型，则保持未知。

While a boolean choice only allows a single config entry to be selected,
a tristate choice also allows any number of config entries to be set to
\'m\'. This can be used if multiple drivers for a single hardware exists
and only a single driver can be compiled/loaded into the kernel, but all
drivers can be compiled as modules.

虽然布尔选择只允许选择单个配置
项，但三态选择还允许将任意数量的
配置项设置为 'm'。如果单个
硬件存在多个驱动程序，并且只有
一个驱动程序可以编译/加载到内核中，
但所有驱动程序都可以编译为模块，
则可以使用此功能。

A choice accepts another option \"optional\", which allows to set the
choice to \'n\' and no entry needs to be selected.

选择接受另一个选项 \"optional\"，
它允许将选择设置为 'n'，并且不需要
选择任何项。

comment:

    "comment" <prompt>
    <comment options>

This defines a comment which is displayed to the user during the
configuration process and is also echoed to the output files. The only
possible options are dependencies.

这定义在配置过程中向用户显示的
注释，并且也回显到输出文件。
唯一可能的选项是依赖关系。

menu:

    "menu" <prompt>
    <menu options>
    <menu block>
    "endmenu"

This defines a menu block, see \"Menu structure\" above for more
information. The only possible options are dependencies and \"visible\"
attributes.

这定义一个菜单块，有关更多信息，
请参见上面的 \"Menu structure\"。
唯一可能的选项是依赖关系和
\"visible\" 属性。

if:

    "if" <expr>
    <if block>
    "endif"

This defines an if block. The dependency expression \<expr\> is appended
to all enclosed menu entries.

这定义一个 if 块。依赖表达式
\<expr\> 被附加到所有包含的
菜单项。

source:

    "source" <prompt>

This reads the specified configuration file. This file is always parsed.

这读取指定的配置文件。该文件始终被解析。

mainmenu:

    "mainmenu" <prompt>

This sets the config program\'s title bar if the config program chooses
to use it. It should be placed at the top of the configuration, before
any other statement.

如果配置程序选择使用它，这将设置
配置程序的标题栏。它应该放在
配置的顶部，在任何其他语句之前。

\'#\' Kconfig source file comment:

'#' Kconfig 源文件注释：

An unquoted \'#\' character anywhere in a source file line indicates the
beginning of a source file comment. The remainder of that line is a
comment.

源文件行中任何位置的未加引号的
'#' 字符表示源文件注释的开始。
该行的其余部分是注释。

## Kconfig hints

## Kconfig 提示

This is a collection of Kconfig tips, most of which aren\'t obvious at
first glance and most of which have become idioms in several Kconfig
files.

这是 Kconfig 技巧的集合，
其中大多数乍一看并不明显，
而且大多数已成为多个 Kconfig
文件中的惯用法。

### Adding common features and make the usage configurable

### 添加通用功能并使用法可配置

It is a common idiom to implement a feature/functionality that are
relevant for some architectures but not all. The recommended way to do
so is to use a config variable named [HAVE]()\* that is defined in a
common Kconfig file and selected by the relevant architectures. An
example is the generic IOMAP functionality.

实现与某些架构相关但并非所有
架构相关的功能/特性是一种常见的
惯用法。推荐的做法是使用名为
[HAVE]()\* 的配置变量，该变量在
公共 Kconfig 文件中定义并由相关
架构选择。一个例子是通用 IOMAP
功能。

We would in lib/Kconfig see:

我们会在 lib/Kconfig 中看到：

    # Generic IOMAP is used to ...
    config HAVE_GENERIC_IOMAP

    config GENERIC_IOMAP
      depends on HAVE_GENERIC_IOMAP && FOO

And in lib/Makefile we would see:

在 lib/Makefile 中我们会看到：

    obj-$(CONFIG_GENERIC_IOMAP) += iomap.o

For each architecture using the generic IOMAP functionality we would
see:

对于使用通用 IOMAP 功能的每个架构，我们会看到：

    config X86
      select ...
      select HAVE_GENERIC_IOMAP
      select ...

Note: we use the existing config option and avoid creating a new config
variable to select HAVE_GENERIC_IOMAP.

注意：我们使用现有的配置选项并
避免创建新的配置变量来选择
HAVE_GENERIC_IOMAP。

Note: the use of the internal config variable HAVE_GENERIC_IOMAP, it is
introduced to overcome the limitation of select which will force a
config option to \'y\' no matter the dependencies. The dependencies are
moved to the symbol GENERIC_IOMAP and we avoid the situation where
select forces a symbol equals to \'y\'.

注意：使用内部配置变量
HAVE_GENERIC_IOMAP 是为了克服
select 的限制，它会强制将配置
选项设置为 'y'，而不管依赖关系
如何。依赖关系被移动到符号
GENERIC_IOMAP，我们避免了 select
强制将符号设置为 'y' 的情况。

### Adding features that need compiler support

### 添加需要编译器支持的功能

There are several features that need compiler support. The recommended
way to describe the dependency on the compiler feature is to use
\"depends on\" followed by a test macro:

有几个功能需要编译器支持。
描述对编译器功能依赖的推荐方法是
使用 \"depends on\" 后跟测试宏：

    config STACKPROTECTOR
      bool "Stack Protector buffer overflow detection"
      depends on $(cc-option,-fstack-protector)
      ...

If you need to expose a compiler capability to makefiles and/or C source
files, [CC_HAS\_]{.title-ref} is the recommended prefix for the config
option:

如果您需要向 makefile 和/或
C 源文件公开编译器能力，
[CC_HAS\_]{.title-ref} 是配置
选项的推荐前缀：

    config CC_HAS_FOO
      def_bool $(success,$(srctree)/scripts/cc-check-foo.sh $(CC))

### Build as module only

### 仅作为模块构建

To restrict a component build to module-only, qualify its config symbol
with \"depends on m\". E.g.:

要限制组件构建仅作为模块，
请用 \"depends on m\" 限定其
配置符号。例如：

    config FOO
      depends on BAR && m

limits FOO to module (=m) or disabled (=n).

将 FOO 限制为模块 (=m) 或禁用 (=n)。

### Compile-testing

### 编译测试

If a config symbol has a dependency, but the code controlled by the
config symbol can still be compiled if the dependency is not met, it is
encouraged to increase build coverage by adding an \"\|\| COMPILE_TEST\"
clause to the dependency. This is especially useful for drivers for more
exotic hardware, as it allows continuous-integration systems to
compile-test the code on a more common system, and detect bugs that way.
Note that compile-tested code should avoid crashing when run on a system
where the dependency is not met.

如果配置符号具有依赖关系，但即使
不满足依赖关系，由配置符号控制的
代码仍然可以编译，则建议通过向
依赖关系添加 \"\|\| COMPILE_TEST\"
子句来增加构建覆盖率。这对于更
奇特硬件的驱动程序特别有用，
因为它允许持续集成系统在更常见的
系统上编译测试代码，并以这种方式
检测错误。请注意，编译测试的代码在
不满足依赖关系的系统上运行时应
避免崩溃。

### Architecture and platform dependencies

### 架构和平台依赖关系

Due to the presence of stubs, most drivers can now be compiled on most
architectures. However, this does not mean it makes sense to have all
drivers available everywhere, as the actual hardware may only exist on
specific architectures and platforms. This is especially true for on-SoC
IP cores, which may be limited to a specific vendor or SoC family.

由于存在桩代码，大多数驱动程序
现在可以在大多数架构上编译。
但是，这并不意味着在所有地方都
提供所有驱动程序是合理的，因为
实际硬件可能仅存在于特定架构和
平台上。对于片上系统 IP 核尤其
如此，它们可能仅限于特定供应商或
SoC 系列。

To prevent asking the user about drivers that cannot be used on the
system(s) the user is compiling a kernel for, and if it makes sense,
config symbols controlling the compilation of a driver should contain
proper dependencies, limiting the visibility of the symbol to (a
superset of) the platform(s) the driver can be used on. The dependency
can be an architecture (e.g. ARM) or platform (e.g. ARCH_OMAP4)
dependency. This makes life simpler not only for distro config owners,
but also for every single developer or user who configures a kernel.

为了防止向用户询问在用户正在
编译内核的系统上无法使用的
驱动程序，并且如果有意义，
控制驱动程序编译的配置符号应
包含适当的依赖关系，将符号的
可见性限制为驱动程序可用的平台
（的超集）。依赖关系可以是架构
（例如 ARM）或平台（例如
ARCH_OMAP4）依赖关系。这不仅使
发行版配置所有者的生活更简单，
而且对于配置内核的每个开发人员或
用户也是如此。

Such a dependency can be relaxed by combining it with the
compile-testing rule above, leading to:

这样的依赖关系可以通过与上面的
编译测试规则结合来放宽，
从而导致：

> config FOO
>
> :   bool \"Support for foo hardware\" depends on ARCH_FOO_VENDOR \|\|
>     COMPILE_TEST

### Optional dependencies

### 可选依赖关系

Some drivers are able to optionally use a feature from another module or
build cleanly with that module disabled, but cause a link failure when
trying to use that loadable module from a built-in driver.

某些驱动程序能够选择性地使用
另一个模块的功能或在禁用该模块的
情况下干净地构建，但在尝试从
内置驱动程序使用该可加载模块时会
导致链接失败。

The most common way to express this optional dependency in Kconfig logic
uses the slightly counterintuitive:

在 Kconfig 逻辑中表达此可选
依赖关系的最常见方法使用稍微
违反直觉的方式：

    config FOO
      tristate "Support for foo hardware"
      depends on BAR || !BAR

This means that there is either a dependency on BAR that disallows the
combination of FOO=y with BAR=m, or BAR is completely disabled. For a
more formalized approach if there are multiple drivers that have the
same dependency, a helper symbol can be used, like:

这意味着要么存在对 BAR 的依赖关系，
不允许 FOO=y 与 BAR=m 的组合，
要么 BAR 完全禁用。对于有多个
驱动程序具有相同依赖关系的更正式的
方法，可以使用辅助符号，例如：

    config FOO
      tristate "Support for foo hardware"
      depends on BAR_OPTIONAL

    config BAR_OPTIONAL
      def_tristate BAR || !BAR

### Kconfig recursive dependency limitations

### Kconfig 递归依赖限制

If you\'ve hit the Kconfig error: \"recursive dependency detected\"
you\'ve run into a recursive dependency issue with Kconfig, a recursive
dependency can be summarized as a circular dependency. The kconfig tools
need to ensure that Kconfig files comply with specified configuration
requirements. In order to do that kconfig must determine the values that
are possible for all Kconfig symbols, this is currently not possible if
there is a circular relation between two or more Kconfig symbols. For
more details refer to the \"Simple Kconfig recursive issue\" subsection
below. Kconfig does not do recursive dependency resolution; this has a
few implications for Kconfig file writers. We\'ll first explain why this
issues exists and then provide an example technical limitation which
this brings upon Kconfig developers. Eager developers wishing to try to
address this limitation should read the next subsections.

如果您遇到 Kconfig 错误：
"检测到递归依赖"，则您遇到了
Kconfig 的递归依赖问题，递归依赖
可以总结为循环依赖。kconfig 工具
需要确保 Kconfig 文件符合指定的
配置要求。为了做到这一点，
kconfig 必须确定所有 Kconfig
符号可能的值，如果两个或更多
Kconfig 符号之间存在循环关系，
目前无法做到这一点。有关更多
详细信息，请参阅下面的
"Simple Kconfig recursive issue"
小节。Kconfig 不进行递归依赖解析；
这对 Kconfig 文件编写者有一些
影响。我们将首先解释为什么存在
此问题，然后提供一个示例技术限制，
这给 Kconfig 开发人员带来了影响。
希望尝试解决此限制的热心开发人员
应阅读下一小节。

### Simple Kconfig recursive issue

### 简单的 Kconfig 递归问题

Read: Documentation/kbuild/Kconfig.recursion-issue-01

阅读：Documentation/kbuild/Kconfig.recursion-issue-01

Test with:

测试方法：

    make KBUILD_KCONFIG=Documentation/kbuild/Kconfig.recursion-issue-01 allnoconfig

### Cumulative Kconfig recursive issue

### 累积的 Kconfig 递归问题

Read: Documentation/kbuild/Kconfig.recursion-issue-02

阅读：Documentation/kbuild/Kconfig.recursion-issue-02

Test with:

测试方法：

    make KBUILD_KCONFIG=Documentation/kbuild/Kconfig.recursion-issue-02 allnoconfig

### Practical solutions to kconfig recursive issue

### kconfig 递归问题的实用解决方案

Developers who run into the recursive Kconfig issue have two options at
their disposal. We document them below and also provide a list of
historical issues resolved through these different solutions.

遇到递归 Kconfig 问题的开发人员
有两个选择。我们在下面记录它们，
并提供通过这些不同解决方案解决的
历史问题列表。

> a)  Remove any superfluous \"select FOO\" or \"depends on FOO\"
>
>     删除任何多余的 \"select FOO\" 或 \"depends on FOO\"
>
> b)  Match dependency semantics:
>
>     匹配依赖语义：
>
> > b1) Swap all \"select FOO\" to \"depends on FOO\" or,
> >
> >     将所有 \"select FOO\" 交换为 \"depends on FOO\"，或
> >
> > b2) Swap all \"depends on FOO\" to \"select FOO\"
> >
> >     将所有 \"depends on FOO\" 交换为 \"select FOO\"

The resolution to a) can be tested with the sample Kconfig file
Documentation/kbuild/Kconfig.recursion-issue-01 through the removal of
the \"select CORE\" from CORE_BELL_A_ADVANCED as that is implicit
already since CORE_BELL_A depends on CORE. At times it may not be
possible to remove some dependency criteria, for such cases you can work
with solution b).

a) 的解决方案可以通过示例
Kconfig 文件 Documentation/kbuild/
Kconfig.recursion-issue-01 测试，
通过从 CORE_BELL_A_ADVANCED 中
删除 \"select CORE\"，因为由于
CORE_BELL_A 依赖于 CORE，这已经是
隐含的。有时可能无法删除某些
依赖标准，对于这种情况，您可以
使用解决方案 b)。

The two different resolutions for b) can be tested in the sample Kconfig
file Documentation/kbuild/Kconfig.recursion-issue-02.

b) 的两种不同解决方案可以在示例
Kconfig 文件 Documentation/kbuild/
Kconfig.recursion-issue-02 中测试。

Below is a list of examples of prior fixes for these types of recursive
issues; all errors appear to involve one or more \"select\" statements
and one or more \"depends on\".

以下是这些类型递归问题的先前
修复示例列表；所有错误似乎都涉及
一个或多个 \"select\" 语句和一个或
多个 \"depends on\"。

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

(1) Partial (or no) quote of error.

    部分（或没有）错误引用。

(2) That seems to be the gist of that fix.

    这似乎是该修复的要点。

(3) Same error.

    相同错误。

### Future kconfig work

### 未来的 kconfig 工作

Work on kconfig is welcomed on both areas of clarifying semantics and on
evaluating the use of a full SAT solver for it. A full SAT solver can be
desirable to enable more complex dependency mappings and / or queries,
for instance one possible use case for a SAT solver could be that of
handling the current known recursive dependency issues. It is not known
if this would address such issues but such evaluation is desirable. If
support for a full SAT solver proves too complex or that it cannot
address recursive dependency issues Kconfig should have at least clear
and well defined semantics which also addresses and documents
limitations or requirements such as the ones dealing with recursive
dependencies.

欢迎在澄清语义和评估使用完整
SAT 求解器方面对 kconfig 进行
工作。完整的 SAT 求解器可能是
理想的，以启用更复杂的依赖映射
和/或查询，例如 SAT 求解器的一个
可能用例可能是处理当前已知的递归
依赖问题。目前尚不清楚这是否会
解决此类问题，但这种评估是理想的。
如果对完整 SAT 求解器的支持证明
过于复杂或无法解决递归依赖问题，
Kconfig 至少应该具有清晰且定义
明确的语义，这些语义还应解决和
记录限制或要求，例如处理递归
依赖的限制或要求。

Further work on both of these areas is welcomed on Kconfig. We elaborate
on both of these in the next two subsections.

欢迎在 Kconfig 上对这两个领域
进行进一步的工作。我们在接下来的
两个小节中详细说明这两个方面。

### Semantics of Kconfig

### Kconfig 的语义

The use of Kconfig is broad, Linux is now only one of Kconfig\'s users:
one study has completed a broad analysis of Kconfig use in 12
projects[^1]. Despite its widespread use, and although this document
does a reasonable job in documenting basic Kconfig syntax a more precise
definition of Kconfig semantics is welcomed. One project deduced Kconfig
semantics through the use of the xconfig configurator[^2]. Work should
be done to confirm if the deduced semantics matches our intended Kconfig
design goals. Another project formalized a denotational semantics of a
core subset of the Kconfig language[^3].

Kconfig 的使用范围很广，
Linux 现在只是 Kconfig 的用户
之一：一项研究完成了对 12 个项目中
Kconfig 使用的广泛分析[^1]。
尽管它被广泛使用，尽管本文档在
记录基本 Kconfig 语法方面做得
相当好，但仍欢迎更精确的 Kconfig
语义定义。一个项目通过使用 xconfig
配置器推导出 Kconfig 语义[^2]。
应该进行工作以确认推导出的语义
是否与我们预期的 Kconfig 设计
目标相匹配。另一个项目形式化了
Kconfig 语言核心子集的指称
语义[^3]。

Having well defined semantics can be useful for tools for practical
evaluation of dependencies, for instance one such case was work to
express in boolean abstraction of the inferred semantics of Kconfig to
translate Kconfig logic into boolean formulas and run a SAT solver on
this to find dead code / features (always inactive), 114 dead features
were found in Linux using this methodology[^4] (Section 8: Threats to
validity). The kismet tool, based on the semantics in[^5], finds abuses
of reverse dependencies and has led to dozens of committed fixes to
Linux Kconfig files[^6].

具有明确定义的语义对于实际评估
依赖关系的工具很有用，例如，
一个这样的案例是在 Kconfig 推断
语义的布尔抽象中表达，将 Kconfig
逻辑转换为布尔公式，并在此基础上
运行 SAT 求解器以查找死代码/功能
（始终不活动），使用此方法在
Linux 中发现了 114 个死功能[^4]
（第 8 节：有效性威胁）。基于[^5]中
语义的 kismet 工具发现反向依赖的
滥用，并导致对 Linux Kconfig 文件
进行了数十次提交的修复[^6]。

Confirming this could prove useful as Kconfig stands as one of the
leading industrial variability modeling languages[^7][^8]. Its study
would help evaluate practical uses of such languages, their use was only
theoretical and real world requirements were not well understood. As it
stands though only reverse engineering techniques have been used to
deduce semantics from variability modeling languages such as
Kconfig[^9].

确认这一点可能会很有用，因为
Kconfig 是领先的工业可变性建模
语言之一[^7][^8]。其研究将有助于
评估此类语言的实际用途，它们的
使用仅是理论上的，现实世界的
需求尚未得到充分理解。然而，
目前只有逆向工程技术被用来从
可变性建模语言（如 Kconfig）中
推导语义[^9]。

### Full SAT solver for Kconfig

### Kconfig 的完整 SAT 求解器

Although SAT solvers[^10] haven\'t yet been used by Kconfig directly, as
noted in the previous subsection, work has been done however to express
in boolean abstraction the inferred semantics of Kconfig to translate
Kconfig logic into boolean formulas and run a SAT solver on it[^11].
Another known related project is CADOS[^12] (former VAMOS[^13]) and the
tools, mainly undertaker[^14], which has been introduced first
with[^15]. The basic concept of undertaker is to extract variability
models from Kconfig and put them together with a propositional formula
extracted from CPP #ifdefs and build-rules into a SAT solver in order to
find dead code, dead files, and dead symbols. If using a SAT solver is
desirable on Kconfig one approach would be to evaluate repurposing such
efforts somehow on Kconfig. There is enough interest from mentors of
existing projects to not only help advise how to integrate this work
upstream but also help maintain it long term. Interested developers
should visit:

尽管如前一小节中所述，Kconfig 尚未
直接使用 SAT 求解器[^10]，但已经
完成了在布尔抽象中表达 Kconfig 的
推断语义以将 Kconfig 逻辑转换为
布尔公式并在其上运行 SAT 求解器的
工作[^11]。另一个已知的相关项目是
CADOS[^12]（前身为 VAMOS[^13]）
及其工具，主要是 undertaker[^14]，
它首次在[^15]中引入。undertaker
的基本概念是从 Kconfig 中提取
可变性模型，并将它们与从 CPP
#ifdefs 和构建规则中提取的命题
公式一起放入 SAT 求解器，以便
查找死代码、死文件和死符号。
如果在 Kconfig 上使用 SAT 求解器是
理想的，一种方法是评估以某种方式在
Kconfig 上重新利用此类工作。
现有项目的导师有足够的兴趣不仅
帮助建议如何将这项工作集成到上游，
而且还帮助长期维护它。感兴趣的
开发人员应该访问：

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
