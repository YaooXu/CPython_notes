# Cpython源码阅读笔记

![icon](assets/py.png)
*该仓库中的assets文件夹用来储存markdown中的图片*


## Grammer
pegn通过Grammer文件生成解析表，如果修改了Grammer文件，需要重新生成解析表。

pege: EBNF -> NFA -> DFA。

## Tokens
python实现的tokenize位于 Lib/tokenize.py。

