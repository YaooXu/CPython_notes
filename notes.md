# Cpython源码阅读笔记

*该仓库中的assets文件夹用来储存markdown中的图片*

> Cpython代码命名规则:
>
> -  `Py` 前缀为公共函数, 非静态函数. Py\_前缀保留用于Py_FatalError等全局服务例程。特定的对象（如特定的对象类型API）使用较长的前缀，例如PyString_用于字符串函数。
> - 公共函数使用大小写混合和下划线，如: `PyObject_GetAttr`, `Py_BuildValue`, `PyExc_TypeError`.
> - 有时，加载器必须能够看到内置函数。使用\_Py前缀，例如_PyObject_Dump。
> - 宏需要使用大小写混合的前缀然后使用大写，如`PyString_AS_STRING`, `Py_PRINT_RAW`.


## Grammer
pegn通过Grammer文件生成解析表，如果修改了Grammer文件，需要重新生成解析表。

pege: EBNF -> NFA -> DFA。

## Tokens
python实现的tokenize位于 Lib/tokenize.py。

## 运行流程

![Python run swim lane diagram](assets/swim-lanes-chart-1.9fb3000aad85.png)

### init

1. Python/initconfig.c中 `config_read_env_vars` 函数读取环境变量。

### 运行方式

#### -c

```
./python -c "print('hi')"
```

![Flow chart of pymain_run_command](assets/pymain_run_command.f5da561ba7d5.png)

1.  `pymain_run_command()`中调用`PyUnicode_FromWideChar()`把-c之后的参数从wchar_t *（int *)转化为str。

#### -m



## 编译

​	整体流程分为两步：

1. 遍历树并创建一个控制流图，它表示执行的逻辑顺序
2. 将CFG中的节点转换为更小的可执行语句，即字节码



**步骤**

1. Python/pythonrun.c/`PyRun_FileExFlags()`
   1. `PyParser_ASTFromFileObject()`把FILE句柄转化为mod, 类型为[mod_ty](#mod_ty)。
   2. `run_mod()`通过`PyAST_CompileObject`把mod转化为PyCodeObject，然后把其送入`run_eval_code_obj`函数。



## 部分类型的含义

### <a name="mod_ty">mod_ty</a>

Python中5种模块类型之一的容器结构，是AST的实例，包含有：

1. `Module`
2. `Interactive`
3. `Expression`
4. `FunctionType`
5. `Suite`

### <a name="PyObject">PyObject</a>

所有Python的对象都继承于PyObject，但是C不是面向对象的语言，因此PyObject是Python对象内存开头的数据结构。



## 部分宏的含义

### <a name="ADDOP_JABS">ADDOP_JABS</a>

**ADD** **O**peration with **J**ump to an **ABS**olute position

### ADDOP_JREL

**ADD** **O**peration with **J**ump to a **REL**ative position

### 