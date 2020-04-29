# Cpython源码阅读笔记

*该仓库中的assets文件夹用来储存markdown中的图片*

> Cpython代码命名规则:
>
> -  `Py` 前缀为公共函数, 非静态函数。 Py\_前缀保留用于Py_FatalError等全局服务例程。特定的对象（如特定的对象类型API）使用较长的前缀，例如PyString_用于字符串函数。
> - 公共函数使用大小写混合和下划线，如: `PyObject_GetAttr`, `Py_BuildValue`, `PyExc_TypeError`。
> - 有时，加载器必须能够看到内置函数。使用\_Py前缀，例如_PyObject_Dump。
> - `宏需要使用大小写混合的前缀然后使用大写`，如`PyString_AS_STRING`, `Py_PRINT_RAW`。





## 整体运行流程

![Python run swim lane diagram](assets/swim-lanes-chart-1.9fb3000aad85.png)

1. 

### 运行方式

#### -c

```
./python -c "print('hi')"
```

![Flow chart of pymain_run_command](assets/pymain_run_command.f5da561ba7d5.png)

1.  `pymain_run_command()`中调用`PyUnicode_FromWideChar()`把-c之后的参数从wchar_t *（int *)转化为str。

#### -m

## 初始配置 

### init
在执行任何Python代码之前，首先要建立基础的配置。
运行时的配置是在`Include/cpython/initconfig.h`中定义的数据结构PyConfig，其部分结构如下：
```
typedef struct {
    int _config_version;  /* Internal configuration version,
                             used for ABI compatibility */
    int _config_init;     /* _PyConfigInitEnum value */
    
    int faulthandler;
    int tracemalloc;
    int import_time;        /* PYTHONPROFILEIMPORTTIME, -X importtime */
    int show_ref_count;     /* -X showrefcount */
    int show_alloc_count;   /* -X showalloccount */
    int dump_refs;          /* PYTHONDUMPREFS */
    int malloc_stats;       /* PYTHONMALLOCSTATS */
    wchar_t *filesystem_encoding;
    wchar_t *filesystem_errors;
    wchar_t *pycache_prefix;  /* PYTHONPYCACHEPREFIX, -X pycache_prefix=PATH */
    int parse_argv;           /* Parse argv command line arguments? */

    PyWideStringList argv;
    wchar_t *program_name;
    PyWideStringList xoptions;     /* Command line -X options */
    PyWideStringList warnoptions;  /* Warnings options */
    int site_import;
    int bytes_warning;
    int inspect;
    int interactive;
    int optimization_level;
    int parser_debug;
    int write_bytecode;
    int verbose;
    int quiet;
    int user_site_directory;
    int configure_c_stdio;
    int buffered_stdio;
    wchar_t *stdio_encoding;
    wchar_t *stdio_errors;

#ifdef MS_WINDOWS
    int legacy_windows_stdio;
#endif

    wchar_t *check_hash_pycs_mode;

    int pathconfig_warnings;

    wchar_t *pythonpath_env; /* PYTHONPATH environment variable */
    wchar_t *home; 

    int module_search_paths_set;  /* If non-zero, use module_search_paths */
    PyWideStringList module_search_paths;  

    wchar_t *executable;        /* sys.executable */
    wchar_t *base_executable;   /* sys._base_executable */
    wchar_t *prefix;            /* sys.prefix */
    wchar_t *base_prefix;       /* sys.base_prefix */
    wchar_t *exec_prefix;       /* sys.exec_prefix */
    wchar_t *base_exec_prefix;  /* sys.base_exec_prefix */

    int skip_source_first_line;

    wchar_t *run_command;   /* -c command line argument */
    wchar_t *run_module;    /* -m command line argument */
    wchar_t *run_filename;  /* Trailing command line argument without -c or -m */

    ……

} PyConfig;
```

PyConfig中定义了运行时的基本配置，包括：
- faulthandler成员：是否支持错误处理
- bytes_warning成员：字节警告信息
- malloc_stats成员：内存分配状态
- use_environment、pythonpath_env成员：运行时设置的环境变量信息
- executable等成员：设置sys信息
- module_search_paths成员：设置sys.path信息
- 定义执行的方式
  - run_command成员：输入参数为-c对应command模式
  - run_module成员：输入参数为-m对应module模式
  - run_filename成员：除了-c和-m之外为filename模式
- 各种模式的运行时标志，例如调试和优化模式
- 提供了执行模式，例如是否传递文件名stdin或模块名称

配置数据结构的主要功能是在CPython运行时启用和禁用各种功能。

文件`Python/initconfig.c`是`initconfig.h`对应的C文件，建立了从环境变量和运行时命令行标志读取设置的逻辑。
下面介绍`initconfig.c`中部分重要的函数：
-  `config_read_env_vars` 函数读取环境变量并将其用于分配配置中设置的值。
```
static PyStatus
config_read_env_vars(PyConfig *config)
{
    PyStatus status;
    int use_env = config->use_environment;

    /* Get environment variables */
    _Py_get_env_flag(use_env, &config->parser_debug, "PYTHONDEBUG");
    _Py_get_env_flag(use_env, &config->verbose, "PYTHONVERBOSE");
    _Py_get_env_flag(use_env, &config->optimization_level, "PYTHONOPTIMIZE");
    _Py_get_env_flag(use_env, &config->inspect, "PYTHONINSPECT");

    ……

    return _PyStatus_OK();
}
```
- _PyConfig_Write函数-设置Py_xxx全局配置变量、初始化C标准流(stdin, stdout, stderr)
- config_read_cmdline函数，读取命令行参数，如-c、-m、-V，返回值为PyStatus
- config_parse_cmdline函数，根据输入从命令行参数设置处理模式，返回值为PyStatus
```
static PyStatus
config_parse_cmdline(PyConfig *config, PyWideStringList *warnoptions,
                     Py_ssize_t *opt_index)
{

        if (c == 'c') {
            if (config->run_command == NULL) {
                /* -c is the last option; following arguments
                   that look like options are left for the
                   command to interpret. */
                size_t len = wcslen(_PyOS_optarg) + 1 + 1;
                wchar_t *command = PyMem_RawMalloc(sizeof(wchar_t) * len);
                if (command == NULL) {
                    return _PyStatus_NO_MEMORY();
                }
                memcpy(command, _PyOS_optarg, (len - 2) * sizeof(wchar_t));
                command[len - 2] = '\n';
                command[len - 1] = 0;
                config->run_command = command;
            }
            break;
        }
        ……

        switch (c) {
        case 0:
            // Handle long option.
            assert(longindex == 0); // Only one long option now.
            if (wcscmp(_PyOS_optarg, L"always") == 0
                || wcscmp(_PyOS_optarg, L"never") == 0
                || wcscmp(_PyOS_optarg, L"default") == 0)
            {
                status = PyConfig_SetString(config, &config->check_hash_pycs_mode,
                                            _PyOS_optarg);
                if (_PyStatus_EXCEPTION(status)) {
                    return status;
                }
            } else {
                fprintf(stderr, "--check-hash-based-pycs must be one of "
                        "'default', 'always', or 'never'\n");
                config_usage(1, program);
                return _PyStatus_EXIT(2);
            }
            break;

        case 'b':
            config->bytes_warning++;
            break;

        ……

        default:
            /* unknown argument: parsing failed */
            config_usage(1, program);
            return _PyStatus_EXIT(2);
        }
    } while (1);

   ……

    return _PyStatus_OK();
}
```

## python的编译过程
在编译原理中我们学习到对于一种语言的编译，往往有词法分析，语法分析，语义处理，中间代码生成，代码优化，代码生成这几个阶段。
python作为一门解释性语言，在编译时往往是并不产生目标机器代码，而是生成一种叫做字节码的中间代码，并交给python虚拟机去执行。接下来我们将介绍python源代码是如何转化为字节码的。
对于python的编译过程，我们可以简单的分成以下这几部分：
- Tokenizer进行词法分析，把源程序分解为Token
- Parser根据Token创建CST
- CST被转换为AST
- AST被编译为字节码  

在语言分析的初步我们需要明确该语言的tokens和grammar
### Tokens
python内置的Token Type定义在Lib/token.py，部分内容如下：
```
ENDMARKER = 0
NAME = 1
NUMBER = 2
STRING = 3
NEWLINE = 4
INDENT = 5
DEDENT = 6
LPAR = 7
RPAR = 8
LSQB = 9
RSQB = 10
COLON = 11
COMMA = 12
SEMI = 13
PLUS = 14
MINUS = 15
STAR = 16
SLASH = 17
VBAR = 18
AMPER = 19
LESS = 20
……

EXACT_TOKEN_TYPES = {
    '!=': NOTEQUAL,
    '%': PERCENT,
    '%=': PERCENTEQUAL,
    '&': AMPER,
    '&=': AMPEREQUAL,
    '(': LPAR,
    ')': RPAR,
    '*': STAR,
    '**': DOUBLESTAR,
    '**=': DOUBLESTAREQUAL,
    ……
}
```
上述Tokens由`Tools/scripts/generate_token.py`工具自动生成

python实现的词法分析器位于`Lib/tokenize.py`，其作用如下：
tokenize(readline)是一个生成器，它将一个字节流分解为Python的token。它根据PEP-0263对字节进行解码。
它接受一个类似readline的方法，该方法被用于反复调用以获取下一行输入(直到EOF)。它用这些生成5元组，5元组结构如下：
- the token type (详情见token.py)
- the token (a string)
- the starting (row, column) indices of the token (a 2-tuple of ints)
- the ending (row, column) indices of the token (a 2-tuple of ints)
- the original line (string)
部分内容如下：
```
 # Parse the arguments and options
    parser = argparse.ArgumentParser(prog='python -m tokenize')
    parser.add_argument(dest='filename', nargs='?',
                        metavar='filename.py',
                        help='the file to tokenize; defaults to stdin')
    parser.add_argument('-e', '--exact', dest='exact', action='store_true',
                        help='display token names using the exact type')
    args = parser.parse_args()

    try:
        # Tokenize the input
        if args.filename:
            filename = args.filename
            with _builtin_open(filename, 'rb') as f:
                tokens = list(tokenize(f.readline))
        else:
            filename = "<stdin>"
            tokens = _tokenize(sys.stdin.readline, None)

        # Output the tokenization
        for token in tokens:
            token_type = token.type
            if args.exact:
                token_type = token.exact_type
            token_range = "%d,%d-%d,%d:" % (token.start + token.end)
            print("%-20s%-15s%-15r" %
                  (token_range, tok_name[token_type], token.string))
    except IndentationError as err:
        line, column = err.args[1][1:3]
        error(err.args[0], filename, (line, column))
    except TokenError as err:
        line, column = err.args[1]
        error(err.args[0], filename, (line, column))
    except SyntaxError as err:
        error(err, filename)
    except OSError as err:
        error(err)
    except KeyboardInterrupt:
        print("interrupted\n")
    except Exception as err:
        perror("unexpected error: %s" % err)
        raise
```

### Grammer

Python 的语法文件使用具有正则表达式语法的 Extended-BNF（EBNF）规范。

Cpython所支持的Grammar和Token具体内容位于的Grammar文件夹下的两个文件中。

其中，Grammer/Grammar部分内容如下：
```
stmt: simple_stmt | compound_stmt
simple_stmt: small_stmt (';' small_stmt)* [';'] NEWLINE
small_stmt: (expr_stmt | del_stmt | pass_stmt | flow_stmt |
             import_stmt | global_stmt | nonlocal_stmt | assert_stmt)
expr_stmt: testlist_star_expr (annassign | augassign (yield_expr|testlist) |
                     [('=' (yield_expr|testlist_star_expr))+ [TYPE_COMMENT]] )
annassign: ':' test ['=' (yield_expr|testlist_star_expr)]
testlist_star_expr: (test|star_expr) (',' (test|star_expr))* [',']
augassign: ('+=' | '-=' | '*=' | '@=' | '/=' | '%=' | '&=' | '|=' | '^=' |
            '<<=' | '>>=' | '**=' | '//=')
# For normal and annotated assignments, additional restrictions enforced by the interpreter
del_stmt: 'del' exprlist
pass_stmt: 'pass'
flow_stmt: break_stmt | continue_stmt | return_stmt | raise_stmt | yield_stmt
break_stmt: 'break'
continue_stmt: 'continue'
return_stmt: 'return' [testlist_star_expr]
yield_stmt: yield_expr
raise_stmt: 'raise' [test ['from' test]]
import_stmt: import_name | import_from
import_name: 'import' dotted_as_names
……
```
对应的Grammar/Tokens的部分内容如下：
```
LPAR                    '('
RPAR                    ')'
LSQB                    '['
RSQB                    ']'
COLON                   ':'
COMMA                   ','
SEMI                    ';'
PLUS                    '+'
MINUS                   '-'
STAR                    '*'
SLASH                   '/'
VBAR                    '|'
AMPER                   '&'
LESS                    '<'
GREATER                 '>'
EQUAL                   '='
DOT                     '.'
PERCENT                 '%'
LBRACE                  '{'
RBRACE                  '}'
EQEQUAL                 '=='
NOTEQUAL                '!='
LESSEQUAL               '<='
GREATEREQUAL            '>='
TILDE                   '~'
CIRCUMFLEX              '^'
LEFTSHIFT               '<<'
RIGHTSHIFT              '>>'
DOUBLESTAR              '**'
PLUSEQUAL               '+='
……
```
### pgen工具的使用

[pgen](https://python-history.blogspot.com/2018/05/the-origins-of-pgen.html)即语法分析生成器（parser generator），是Guido van Rossum（Python之父）为python写的第一段代码。在pgen中使用了`LL(1)`的分析方法以及一套类似`EBNF`的语法符号。

Grammar文件本身不会被Python编译器直接使用，而是使用一个名为 pgen 的工具，来创建的解析器表。在整个编译的流程当中，pgen使用grammar文件输出的是一个解析树，但是这个解析树并不直接用作代码生成器的输入：它首先会被转换成抽象语法树（AST），然后再被编译成字节码。  

pgen工具将语法文件`Grammar/Grammar`使用的`Extended-BNF（EBNF）`语句转化为非确定有限自动机NFA，然后再将NFA转化为确定有限自动机DFA，生成解析器表。如果我们修改了语法文件，则需要重新生成解析器表并重新编译Python。

下面演示一个修改语法文件的简单样例：

在python中，原pass语句定义为：`pass_stmt: 'pass'`

若要增加新的可以接受的关键字`proceed`。则改为：`pass_stmt: 'pass' | 'proceed'`

重新编译语法文件，以生成新的解析器表：（不同的操作系统对应命令如下）

- 在macOS和Linux上，使用命令`make regen-grammar`以运行`pgen`更改后的语法文件。
- 在Windows上，需用PCbuild文件夹下的批处理脚本运行，使用命令`build.bat --regen`。

执行完上述命令之后，生成新的`Include/graminit.h`和`Python/graminit.c`，即基于新的语法文件生成的解析表。

然后重新编译CPython，即完成具有新的语法的python解释器，此时`proceed`也会作为python的关键词，效果同`pass`一致。

### 词法分析(Lexing)
词法分析的主要部分在`Parser/tokenizer.c`文件中，使用`PyTokenizer_FromFile()`实例化标记化器状态`tok_state`结构体,然后使用`tok_get()`函数在DFA的不同状态中转化并获得最终tokens。这与我们编译原理中学习的内容类似，我们以一个识别`DOT`部分为例
```
static int
tok_get(struct tok_state *tok, char **p_start, char **p_end)
{
...
       /* Period or number starting with period? */
    if (c == '.') {
        c = tok_nextc(tok);
        if (isdigit(c)) {
            goto fraction;
        } else if (c == '.') {
            c = tok_nextc(tok);
            if (c == '.') {
                *p_start = tok->start;
                *p_end = tok->cur;
                return ELLIPSIS;
            }
            else {
                tok_backup(tok, c);
            }
            tok_backup(tok, '.');
        }
        else {
            tok_backup(tok, c);
        }
        *p_start = tok->start;
        *p_end = tok->cur;
        return DOT;
    }
...
}
```
可以看到在这个状态下判断该符号是否为`DOT`时需要看下一个符号是否为数字等符号，通过不断调用`token_get`函数能够获得python中所有的tokens。  
在这个例子里，`DOT`是一个tokens，其值在`Include/token.h`中定义。所有标记都是常量int值，并且在我们之前使用pgen时运行`make regen-grammar`生成了`Include/token.h`文件。 

### 语法分析(Parsing)
#### 生成语法图
语法分析部分首先会通过对上面提到的`pgen`工具进行调用，自动生成`Graminit.c`和`Graminit.h`两文件。
我们可以简单查看下`Graminit.c`文件，它定义了Python进行语法分析所需要的静态数据。实际上它构成了整个语法图。
```
/* Generated by Parser/pgen */

#include "grammar.h"
grammar _PyParser_Grammar;
static const arc arcs_0_0[3] = {
    {2, 1},
    {3, 2},
    {4, 1},
};
static const arc arcs_0_1[1] = {
    {0, 1},
};
static const arc arcs_0_2[1] = {
    {2, 1},
};
static state states_0[3] = {
    {3, arcs_0_0},
    {1, arcs_0_1},
    {1, arcs_0_2},
};
...

static const dfa dfas[92] = {
    {256, "single_input", 3, states_0,
     "\344\377\377\377\377\017\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000"},
    {257, "file_input", 2, states_1,
     "\344\377\377\377\377\057\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000"},
...
static const label labels[184] = {
    {0, "EMPTY"},
    {256, 0},
    {4, 0},
    {295, 0},
    {270, 0},
...
```
这里的几种数据类型都在grammar.h中进行了定义
-   `label`是从状态转移到另外一个状态所经过的边所对应的符号，可以      是非终结符，也可以是终结符。`label`一定依附于一条或者多条边。
    ```
    typedef struct {
    int          lb_type;
    const char  *lb_str;
    } label;
    ```
    `Lb_type`代表符号的类型，如终结符`NAME`,`Lb_str`代表具体符号的内容。比如，`label (NAME, “if”)`表示当`parser`处于某个状态，如果遇到了`’if’`这个符号，则移动另外一个状态。
- `arc`代表DFA中一个状态到另一个状态的弧/边。
    ```
    typedef struct {
    short       a_lbl;          /* Label of this arc */
    short       a_arrow;        /* State where this arc goes to */
    } arc;

    ```
    `A_lbl`代表`arc`所对应的`label`，而`a_arrow`记录了`arc`的目标状态。因为`arc`是属于某个状态的，因此不用记录`arc`的起始状态。
- `State`代表着DFA中的状态节点。
    ```
    typedef struct {
    int          s_narcs;
    const arc   *s_arc;         /* Array of arcs */

    /* Optional accelerators */
    int          s_lower;       /* Lowest label index */
    int          s_upper;       /* Highest label index */
    int         *s_accel;       /* Accelerator */
    int          s_accept;      /* Nonzero for accepting state */
    } state;
    ```
    每个`state`记录了从该`state`出发的边的集合，存放在`s_arc`中。
- `dfa`结构中记录了起始状态`d_initial`和所有状态的集合`d_state`。
    ```
    typedef struct {
        int          d_type;        /* Non-terminal this represents */
        char        *d_name;        /* For printing */
        int          d_nstates;
        state       *d_state;       /* Array of states */
        bitset       d_first;
    } dfa;
    ```
    `d_first`记录了该`dfa`所对应的非终结符的first集合，也就是说，当遇到first集合中的终结符的时候，便需要跳转到此`dfa`中。  

在了解这些结构体的作用后我们就可以理解上面的`graminit.c`文件了。
`arcs_0_0`代表DFA0中从状态0出发的所有`arc`，`arcs_0_1`代表DFA0中从状态1出发的所有`arc`，依此类推。`arcs_0_0`中`{ 2, 1 }`代表一条边从状态0开始到状态1，`Label`为2（可以在后面查到`label`为2代表NEWLINE，即换行符）。`states_0`记录了DFA0中所有的状态节点上面的所有边。
当定义完所有的DFA的状态和边的信息之后，接下来定义了所有的DFA的数组
```
static const dfa dfas[92] = {
    {256, "single_input", 3, states_0,
     "\344\377\377\377\377\017\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000"},
```
拿第一个元素举例，256在`graminit.h`中可以查到代表`single_input`，也就是交互模式下单条语句。初始状态为0，共有3个状态，对应的状态和边的信息存在`states_0`中，最后的一个很长的字符串代表了该非终结符的first集合，每个字节对应着`label`的ID。
最后我们看一看他的labels数组
```
static const label labels[184] = {
    {0, "EMPTY"},
    {256, 0},
    {4, 0},
```
`{ 0, “EMPTY” }`是一条特殊的边，表示该状态是accept状态，代表DFA的结束。`{ 256, 0 }` 则代表该label对应的是符号`256`，也就是`single_input`，无对应字符串描述。由于每个关键字在语法中是直接出现的，因此在Label中定义了每个关键字。
在整个python3.8中，通过pgen解析grammar文件，生成了92种DFA状态，184个Label，起始Label符号为`256`(`single_input`)的语法图。

#### 通过语法图生成CST

CST (Concrete Syntax Tree) 和AST (Abstract Syntax Tree) 类似，都是语法分析所获得的中间结果。CST有很多东西是使语言明确可解析所必需的，但却没有实际意义,即CST相对于AST而言，CST是直接生成的，含有大量冗余信息的中间结果。
CST生成的核心代码在`Parser/parsetok.c`文件中，我们可以简单的看一看其关键部分
```
    for (;;) {
        char *a, *b;
        int type;
        size_t len;
        char *str;
        col_offset = -1;
        int lineno;
        const char *line_start;
        type = PyTokenizer_Get(tok, &a, &b);
        ...
        if ((err_ret->error =
             PyParser_AddToken(ps, (int)type, str,
                               lineno, col_offset, tok->lineno, end_col_offset,
                               &(err_ret->expected))) != E_OK) {
            if (err_ret->error != E_DONE) {
                PyObject_FREE(str);
                err_ret->token = type;
            }
            break;
        }
    }
```
在这段代码中，程序不断调用PyParser_AddToken根据PyTokenizer所获得的token和当前所处的dfa状态，跳转到下一个状态，并添加到CST中。生成一个完整的具体语法树。
我们一颗通过一个简单的例子查看python的具体语法树长什么样。在python中运行以下代码
```
from pprint import pprint
import parser
st = parser.expr('a + 1')
pprint(parser.st2list(st))
```
运行结果：
```
[258,
 [331,
  [305,
   [309,
    [310,
     [311,
      [312,
       [315,
        [316,
         [317,
          [318,
           [319,
            [320, [321, [322, [323, [324, [1, 'a']]]]]],
            [14, '+'],
            [320, [321, [322, [323, [324, [2, '1']]]]]]]]]]]]]]]]],
 [4, ''],
 [0, '']]
```
为了便于理解，我们也可以获取symbol和token模块中的所有数字,将它们放入字典中，并使用名称递归替换parser.st2list()输出的值。在python中运行以下程序：
```
import symbol
import token
import parser
from pprint import pprint

def lex(expression):
    symbols = {v: k for k, v in symbol.__dict__.items() if isinstance(v, int)}
    tokens = {v: k for k, v in token.__dict__.items() if isinstance(v, int)}
    lexicon = {**symbols, **tokens}
    st = parser.expr(expression)
    st_list = parser.st2list(st)

    def replace(l: list):
        r = []
        for i in l:
            if isinstance(i, list):
                r.append(replace(i))
            else:
                if i in lexicon:
                    r.append(lexicon[i])
                else:
                    r.append(i)
        return r

    return replace(st_list)

pprint(lex('a + 1'))
```
运行结果：
```
['eval_input',
 ['testlist',
  ['test',
   ['or_test',
    ['and_test',
     ['not_test',
      ['comparison',
       ['expr',
        ['xor_expr',
         ['and_expr',
          ['shift_expr',
           ['arith_expr',
            ['term',
             ['factor', ['power', ['atom_expr', ['atom', ['NAME', 'a']]]]]],
            ['PLUS', '+'],
            ['term',
             ['factor',
              ['power', ['atom_expr', ['atom', ['NUMBER', '1']]]]]]]]]]]]]]]]],
 ['NEWLINE', ''],
 ['ENDMARKER', '']]
```
#### 将CST转化为AST

#### 将AST转化为字节码







#### 编译结果 PyCodeObject

​	PyCodeObject为Python编译后的结果，结构如下：

```C
typedef struct {
    PyObject_HEAD
    int co_argcount;            /* #arguments, except *args */
    int co_posonlyargcount;     /* #positional only arguments */
    int co_kwonlyargcount;      /* #keyword only arguments */
    int co_nlocals;             /* #local variables */
    int co_stacksize;           /* #entries needed for evaluation stack */
    int co_flags;               /* CO_..., see below */
    int co_firstlineno;         /* first source line number */
    PyObject *co_code;          /* instruction opcodes */
    PyObject *co_consts;        /* list (constants used) 常量*/
    PyObject *co_names;         /* list of strings (names used) */
    PyObject *co_varnames;      /* tuple of strings (local variable names) 局部变量名集合*/
    PyObject *co_freevars;      /* tuple of strings (free variable names) 实现闭包所需要的东西*/
    PyObject *co_cellvars;      /* tuple of strings (cell variable names) 内部嵌套函数所引用的局部变量名集合*/
    /* The rest aren't used in either hash or comparisons, except for co_name,
       used in both. This is done to preserve the name and line number
       for tracebacks and debuggers; otherwise, constant de-duplication
       would collapse identical functions/lambdas defined on different lines.
    */
    Py_ssize_t *co_cell2arg;    /* Maps cell vars which are arguments. */
    PyObject *co_filename;      /* unicode (where it was loaded from) */
    PyObject *co_name;          /* unicode (name, for reference) */
    PyObject *co_lnotab;        /* string (encoding addr<->lineno mapping) See
                                   Objects/lnotab_notes.txt for details. */
    void *co_zombieframe;       /* for optimization only (see frameobject.c) */
    PyObject *co_weakreflist;   /* to support weakrefs to code objects */
    /* Scratch space for extra data relating to the code object.
       Type is a void* to keep the format private in codeobject.c to force
       people to go through the proper APIs. */
    void *co_extra;

    /* Per opcodes just-in-time cache
     *
     * To reduce cache size, we use indirect mapping from opcode index to
     * cache object:
     *   cache = co_opcache[co_opcache_map[next_instr - first_instr] - 1]
     */

    // co_opcache_map is indexed by (next_instr - first_instr).
    //  * 0 means there is no cache for this opcode.
    //  * n > 0 means there is cache in co_opcache[n-1].
    unsigned char *co_opcache_map;
    _PyOpcache *co_opcache;
    int co_opcache_flag;  // used to determine when create a cache.
    unsigned char co_opcache_size;  // length of co_opcache.
} PyCodeObject;
```

​	编译过程中，每个code block都对应着一个PyCodeObject对象，即PyCodeObject包含该code block经过编译后得到的byte code序列。（Python规定每进入一个新的名字空间就视为进入一个新的code block。）

​	TODO：名字空间的解释（可参考《源码剖析》8.2），.pyc文件的简单介绍



## 执行

#### 执行环境（栈帧模拟）

​	虽然PyCodeObject包含了字节码序列和其他信息，但是却没有包含程序运行时的动态信息，因此在python执行的时候, 虚拟机实际上面对的不是一个 `PyCodeObject` 对象, 而是另一个 `PyFrameObject` 对象，它就是我们说的执行环境, 也是python在高级层次上对栈帧的模拟，x86上程序运行方式如下：

![1588048529337](assets/1588048529337-1588048529499.png)

​	PyFrameObject结构如下：

```C

typedef struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;      /* previous frame, or NULL 使新的栈帧在结束之后可以顺利回到旧的栈帧中 */
    PyCodeObject *f_code;       /* code segment */
    PyObject *f_builtins;       /* builtin symbol table (PyDictObject) */
    PyObject *f_globals;        /* global symbol table (PyDictObject) */
    PyObject *f_locals;         /* local symbol table (any mapping) */
    PyObject **f_valuestack;    /* 运行时栈的栈底位置 points after the last local */
    /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
       Frame evaluation usually NULLs it, but a frame that yields sets it
       to the current stack top. */
    PyObject **f_stacktop;      /* 运行时栈的栈顶位置 */
    PyObject *f_trace;          /* Trace function */
    char f_trace_lines;         /* Emit per-line trace events? */
    char f_trace_opcodes;       /* Emit per-opcode trace events? */

    /* Borrowed reference to a generator, or NULL */
    PyObject *f_gen;

    int f_lasti;                /* Last instruction if called */
    /* Call PyFrame_GetLineNumber() instead of reading this field
       directly.  As of 2.3 f_lineno is only valid when tracing is
       active (i.e. when f_trace is set).  At other times we use
       PyCode_Addr2Line to calculate the line from the current
       bytecode index. */
    int f_lineno;               /* Current line number */
    int f_iblock;               /* index in f_blockstack */
    char f_executing;           /* whether the frame is still executing */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */

    // 动态内存，维护 局部变量+cell对象集合+free对象集合+运行时栈 所需的空间
    PyObject *f_localsplus[1];  /* locals+stack, dynamically sized */
} PyFrameObject;
```

​	相比于x86下的栈帧，PyFrameObject所包含的信息更多，下面依次分析其中的部分关键成员变量。

​	f_code：存放着待执行的PyCodeObject，可以看到每一个PyFrameObject对象都维护一个PyCodeObject对象，说明一个PyFrameObject是对应着一个code block（*code block的范围可大可小. 可以是整个py文件, 可以是class, 可以是函数*）。

​	f_back：指向上一个PyFrameObject，在实际执行过程中，会把很多个PyFrameObject连接起来，使得新的栈帧在结束之后可以顺利回到旧的栈帧中

​	f_builtins、f_globals、f_locals 分别代表内置symbol table，全局symbol table和局部symbol table，他们均为PyDictObject。

​	运行时环境组织如下，与x86运行时连续的内存空间不同，Python虚拟机运行的时候是离散的内存空间，通过链表串连起来。

![1588057893605](assets/1588057893605-1588057893801.png)

#### 虚拟机框架（模拟CPU）

​	执行流程为模拟CPU，进入for循环, 取出第一条字节码之后, 判断指令后执行, 然后一条接一条的从字节流中获取。

​	TODO：稍微结合代码说明一下

#### 运行环境（线程、进程）

​	Python在执行时，可能会有多个线程存在。Python虚拟机是对CPU的模拟可以把他看做软CPU，Python中的所有线程都使用这个软CPU来完成计算工作。真实机器上的任务切换机制对应到Python中，就是使不同的线程轮流使用虚拟机的机制。
​	CPU切换任务时需要保存线程运行环境。对于Python来说，在切换线程之前，同样需要保存关于当前线程的信息。线程状态信息的抽象是通过 `PyThreadState` 对象来实现的, 一个线程将拥有一个`PyThreadState`对象。 `PyThreadState`不是对线程的模拟, 而是对线程状态的抽象。 python的线程仍然使用操作系统的原生线程。对于进程的抽象, 由 `PyInterPreterState` 对象来实现。

​	通常情况下, python只有一个interpreter, 其中维护了一个或多个PyThreadState对象, 这些对象对应的线程轮流使用上面提到的软CPU。 为了实现线程同步, python通过一个全局解释器锁GIL。

​	TODO：稍微介绍一下GIL，包括提出的原因，优缺点等。

​	PyThreadState结构体定义如下：

```C
struct _ts {
    /* See Python/ceval.c for comments explaining most fields */

    struct _ts *prev;
    struct _ts *next;
    PyInterpreterState *interp;

    struct _frame *frame;   // 模拟线程中的函数调用堆栈
    int recursion_depth;
    char overflowed; /* The stack has overflowed. Allow 50 more calls
                        to handle the runtime error. */
    char recursion_critical; /* The current calls must not cause
                                a stack overflow. */
    int stackcheck_counter;

    /* 'tracing' keeps track of the execution depth when tracing/profiling.
       This is to prevent the actual trace/profile code from being recorded in
       the trace/profile. */
    int tracing;
    int use_tracing;

    Py_tracefunc c_profilefunc;
    Py_tracefunc c_tracefunc;
    PyObject *c_profileobj;
    PyObject *c_traceobj;

    /* The exception currently being raised */
    PyObject *curexc_type;
    PyObject *curexc_value;
    PyObject *curexc_traceback;

    /* The exception currently being handled, if no coroutines/generators
     * are present. Always last element on the stack referred to be exc_info.
     */
    _PyErr_StackItem exc_state;

    /* Pointer to the top of the stack of the exceptions currently
     * being handled */
    _PyErr_StackItem *exc_info;

    PyObject *dict;  /* Stores per-thread state */

    int gilstate_counter;

    PyObject *async_exc; /* Asynchronous exception to raise */
    unsigned long thread_id; /* Thread id where this tstate was created */

    int trash_delete_nesting;
    PyObject *trash_delete_later;

    /* Called when a thread state is deleted normally, but not when it
     * is destroyed after fork().
     * Pain:  to prevent rare but fatal shutdown errors (issue 18808),
     * Thread.join() must wait for the join'ed thread's tstate to be unlinked
     * from the tstate chain.  That happens at the end of a thread's life,
     * in pystate.c.
     * The obvious way doesn't quite work:  create a lock which the tstate
     * unlinking code releases, and have Thread.join() wait to acquire that
     * lock.  The problem is that we _are_ at the end of the thread's life:
     * if the thread holds the last reference to the lock, decref'ing the
     * lock will delete the lock, and that may trigger arbitrary Python code
     * if there's a weakref, with a callback, to the lock.  But by this time
     * _PyRuntime.gilstate.tstate_current is already NULL, so only the simplest
     * of C code can be allowed to run (in particular it must not be possible to
     * release the GIL).
     * So instead of holding the lock directly, the tstate holds a weakref to
     * the lock:  that's the value of on_delete_data below.  Decref'ing a
     * weakref is harmless.
     * on_delete points to _threadmodule.c's static release_sentinel() function.
     * After the tstate is unlinked, release_sentinel is called with the
     * weakref-to-lock (on_delete_data) argument, and release_sentinel releases
     * the indirectly held lock.
     */
    void (*on_delete)(void *);
    void *on_delete_data;

    int coroutine_origin_tracking_depth;

    PyObject *async_gen_firstiter;
    PyObject *async_gen_finalizer;

    PyObject *context;
    uint64_t context_ver;

    /* Unique thread state id. */
    uint64_t id;

    /* XXX signal handlers should also be here */

};
```

​	下面依次分析其中的部分关键成员变量。

​	frame: 栈帧列表，模拟线程中函数调用堆栈。

​	当Python虚拟机开始执行时，会将当前线程状态对象中的frame设置为当前的执行环境（frame）。

​	

​	PyInterPreterState结构定义如下：

```C
struct _is {

    struct _is *next;
    struct _ts *tstate_head; // 模拟进程环境中的线程集合

    int64_t id;
    int64_t id_refcount;
    int requires_idref;
    PyThread_type_lock id_mutex;

    int finalizing;

    PyObject *modules;
    PyObject *modules_by_index;
    PyObject *sysdict;
    PyObject *builtins;
    PyObject *importlib;

    /* Used in Python/sysmodule.c. */
    int check_interval;

    /* Used in Modules/_threadmodule.c. */
    long num_threads;
    /* Support for runtime thread stack size tuning.
       A value of 0 means using the platform's default stack size
       or the size specified by the THREAD_STACK_SIZE macro. */
    /* Used in Python/thread.c. */
    size_t pythread_stacksize;

    PyObject *codec_search_path;
    PyObject *codec_search_cache;
    PyObject *codec_error_registry;
    int codecs_initialized;

    /* fs_codec.encoding is initialized to NULL.
       Later, it is set to a non-NULL string by _PyUnicode_InitEncodings(). */
    struct {
        char *encoding;   /* Filesystem encoding (encoded to UTF-8) */
        char *errors;     /* Filesystem errors (encoded to UTF-8) */
        _Py_error_handler error_handler;
    } fs_codec;

    PyConfig config;
#ifdef HAVE_DLOPEN
    int dlopenflags;
#endif

    PyObject *dict;  /* Stores per-interpreter state */

    PyObject *builtins_copy;
    PyObject *import_func;
    /* Initialized to PyEval_EvalFrameDefault(). */
    _PyFrameEvalFunction eval_frame;

    Py_ssize_t co_extra_user_count;
    freefunc co_extra_freefuncs[MAX_CO_EXTRA_USERS];

#ifdef HAVE_FORK
    PyObject *before_forkers;
    PyObject *after_forkers_parent;
    PyObject *after_forkers_child;
#endif
    /* AtExit module */
    void (*pyexitfunc)(PyObject *);
    PyObject *pyexitmodule;

    uint64_t tstate_next_unique_id;

    struct _warnings_runtime_state warnings;

    PyObject *audit_hooks;
};
```





​	进程, 线程, 栈帧关系大致如下:

![1588063672226](assets/1588063672226-1588063672523.png)



TODO：

表达式

控制流

函数机制

类机制

运行环境初始化

模块加载机制

多线程机制

内存管理机制



## 部分类型的含义

### <a name="mod_ty">mod_ty</a>

Python中5种模块类型之一的容器结构，是AST的实例，包含有：

1. `Module`
2. `Interactive`
3. `Expression`
4. `FunctionType`
5. `Suite`

### <a name="PyObject">PyObject</a>

所有Python的对象都是PyObject的拓展，包含着把一个指针转化为对象的所有信息。所有对象都可以转化为PyObject*，对对象成员的访问需要使用 Py_TYPE和Py_REFCNT。



## 部分宏的含义

### <a name="ADDOP_JABS">ADDOP_JABS</a>

**ADD** **O**peration with **J**ump to an **ABS**olute position

### ADDOP_JREL

**ADD** **O**peration with **J**ump to a **REL**ative position

### <a name="Py_TYPE()">Py_TYPE</a>

用于访问`PyObject`中的 `ob_type` 

```
(((PyObject*)(o))->ob_type)
```

### Py_REFCNT(o)

用于访问`PyObject`中的 `ob_refcnt`

```
(((PyObject*)(o))->ob_refcnt)
```