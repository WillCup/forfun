title: antlr简记
date: 2016-07-16 22:49:04
tags: [antlr]
---

要素
---
 - source code：面向编程人员的语言代码；
 - Lexical Analyser ：词汇解析。将source code解析成为某个语言能理解的有意义的形式；
 - Syntax Analyser: 语法解析，检查语言的正确性。
 - Semantic Analyser: 语义解析；
 - Generate/Improve Code: 生成机器能够读懂的执行代码。

![](/imgs/antlr/general_phrases_of_compiler.png)


参考：https://www.youtube.com/watch?v=PooQrbFrd_U


环境准备
---
#### 环境准备
使用eclipse进行开发。

 - IDE环境准备：eclipse classic 3.5.0
 - eclipse 插件
    - GEF-ALL-3.8.2
    - dltk-core-S-3.0.1-201108261011
    - antlr3. http://antlrv3ide.sourceforge.net/updates

#### 插件设置
主要就是preferences里的antlr->builder里添加一个installed package，只需要一个jar包即可，但是规范是把这个jar包放到某个文件夹里。

我的地址是：E:\tools\antlr3.2。里面放置了antlr-3.2.jar文件。
![添加antlr的jar包给插件调用](/imgs/antlr/antlr-install-package-eclipse.png)
还可以在code generator配置一下代码产生的位置。
![代码产生位置](/imgs/antlr/antlr-code-generator-eclipse.png)

#### IDE简介
##### grammer编辑界面
有些自动提示啥的，保存后会自动编译，并在前面指定位置生成lexer和parser代码。
![antlr编辑器](/imgs/antlr/antlr-editor-eclipse.png)

##### 测试执行界面
完成gramemr的编辑之后，可以编辑一些测试脚本，验证grammer的正确性。跟测试连接的真紧密啊，貌似antlr想不测试驱动开发都难，哈哈~
![antlr Intercepter](/imgs/antlr/antlr-intercepter-eclipse.png)
**注意** : 有的时候直接点击右上角的执行按钮，并不能成功解析。可以再试一下执行的下来按钮里的Run(java)的方式，它等同于执行以下代码：
```java
```

##### RailRoad 视图
展示每个rule的路线图。点击左边的rule，右边就自动呈现出来了。可以形象化的展示我们定义的每个rule。
![antlr RailRoad界面](/imgs/antlr/antlr-railroad-eclipse.png)
  
  
  
  
#### 创建项目
创建普通java项目，之后转为antlr项目，.project：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<projectDescription>
	<name>anltr-learning</name>
	<comment></comment>
	<projects>
	</projects>
	<buildSpec>
		<buildCommand>
			<name>org.eclipse.dltk.core.scriptbuilder</name>
			<arguments>
			</arguments>
		</buildCommand>
		<buildCommand>
			<name>org.eclipse.jdt.core.javabuilder</name>
			<arguments>
			</arguments>
		</buildCommand>
	</buildSpec>
	<natures>
		<nature>org.eclipse.jdt.core.javanature</nature>
		<nature>org.deved.antlride.core.nature</nature>
	</natures>
</projectDescription>

```

在根目录创建一个lib文件夹，然后把刚才那个install package里的jar包copy到lib里来，加入当前project的build path。


#### 概念
逻辑流程图
![逻辑流程](/imgs/antlr/antlr-process-concepts.png)
物理流程图
![物理流程](/imgs/antlr/antlr-process-code.png)

- Lexer解析出重要的Token发送给Parser
- Parser生成AST Tree，
- TreeParser负责结合AST Tree里的context生成代码



#### 创建antlr文件
创建Sample.g文件，开始真正编辑antlr文件。经测试，.g文件并不支持中文注释，会报某种NullPointer的错误。下面的注释只为解释，如果copy的话，自行注意去除。
```antlr
grammar Sample;// 类似类名似的，必须跟文件同名

options {
  language = Java;
}

@header {
    package com.will;
}


@lexer::header {
    package com.will;
}

// 以下这些在antlr里都是rule了。就是定义一些语法规则。
// IDENT其实也可以看成rule，只是只包含常量的rule而已。
// 这个program是最外层的rule了，其实也可以叫pppp，随便。
program
    : 'program' IDENT '='
        (constrant | variable)*
        'begin'
        statement*
        'end' IDENT '.'
    ;

constrant
    : 'constrant' IDENT ':' type ':=' Integer ';'
    ;
    
type
    : 'Integer'
    ;
    
variable
    : 'var' IDENT (',' IDENT)* ':' type ';'
    ;
    
    
// fun time
// 运算符优先级设计实现，低优先级expression调用高优先级的expression
term
    : Integer
    | '(' expression ')'
    | IDENT
    ;

negation
    : 'not'* term
    ;
    
unary
    : ('+' |'-')* negation
    ;

mult
    : unary (('*' | '/' | 'mode') unary)*
    ;
    
add
    : mult (('+' | '-') mult)*
    ;
    
relation
    : add (('=' | '/=' | '<' | '<=' | '>' | '>=') add)*
    ;

expression
    : relation (('and' | 'or') relation)*
    ;

statement
    : assignmentStatement
    | ifStatement
    | loopStatement
    ;
exitStatement
    : 'exit' 'when' expression
    ;

assignmentStatement
    : IDENT ':=' expression ';'
    ;
    
ifStatement
    : 'if' expression 'then' statement+
    ('elif' expression 'then' statement+)*
    ('else' statement+)?
    'end' 'if' ';'
    ;
    
loopStatement
    : 'loop' statement* 
    exitStatement
    'end' 'loop' ';'
    ;
 

// 静态常量rule
Integer: '0'..'9'+;

IDENT  : ('a'..'z' | 'A'..'Z' | '0'..'9')+;
WS:(' ' | '\t' |'\n'|'\r'|'\f')+ {$channel = HIDDEN;};

// comment
COMMENT: '//' .* ('\n' | '\r') {$channel = HIDDEN;};
MUTICOMMENT: '/*' .* '*/' {$channel = HIDDEN;};
```



上面的文件中有个要提示的地方，就是优先级设置：
![优先级定义](/imgs/antlr/antlr-exec-level.png)
直接copy的人家的图，大概就是从上到下优先级递减。在grammer中的实现就是低优先级的expression调用高优先级的expression，这样在调用某个expression的时候，就先执行位于叶子节点的最高优先级的expression了。

对应的测试代码：
```will-lang
program XLSample1 = 
constrant one:Integer :=1;
constrant two:Integer :=2;
var x, y, z:Integer;
begin
x := 30 + 2 * 9 -10;
y := 10;
	if x=2 then
		y:= 9 * 7 -1;
		if 9 > d then
			x:= 90;
		else
			x:=0;
		end if;
	elif x>9 and 8 < 2 then
//		x:=90;
		y:=100;
	else
		x:= 99;
		z:=100;
		y:=100;
	end if;

loop
	x:=0;
	y:=99;
	exit when x =0
end loop;
end XLSample1.
```
虽然就这么几个简单的句子，但是跑出来的AST简直是太大了，就不截图了，感兴趣自己运行了看吧。


#### 小结
动态的rule使用小写，静态常量rule就只用大写字母。

##### 表达式
基本跟正则差不太多。
表达式 | 解释
---|---
(a \| b)| 就是a和b可以以任意次序出现，不严格限定a先于b,或者b先于a。
(a \| b)* | 两者都可以有0或多个
(a \| b)+ | 两者至少有一个，多个的话，次序可乱
\| | 或者
'0'..'9' | 从'0'至'9'，字母也一样，生成的java代码都是直接使用>  <啥的限定范围的
'not'? term | 'not'可有可无，但是有的话只能有一个，多个就+或者*
{$channel = HIDDEN;} | 设置当前元素在AST中不可见。应该有待挖掘，可能更多个性化的东西，都可以在这里处理



