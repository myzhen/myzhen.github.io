---
layout: post
title: PostgreSQL函数管理器模块分析1-README
categories: FMGR
description: the README of FMGR 
keywords: git MR PR
---

# PostgreSQL函数管理器模块分析1-README

在分析PG某个模块的代码之前，最好都先阅读一下它的README文件(如果有)，PG的代码布局非常规范和清晰，几乎所有规模化的模块子系统都有自己单独的子目录，我们在深入了解之前可以通过README来提前了解个大概。

```wiki
src/backend/utils/fmgr/README

Function Manager
================

[This file originally explained the transition from the V0 to the V1
interface.  Now it just explains some internals and rationale for the V1
interface, while the V0 interface has been removed.]
【该文件最初解释了从 V0 到 V1 接口的过渡。现在它只是解释了 V1 接口的一些内部原理和基本原理，
 而 V0 接口已被删除。】
笔记：
	从我接触PG开始就是V1了，经常从PG的代码和手册中看到现在PG版本使用的版本1的函数调用约定，
英文称其为：version-1 calling convention。比如，我在开发一个extension时编写的C函数就被
要求遵循VERSION-1调用约定风格来声明函数，extension的language C函数声明：
PG_FUNCTION_INFO_V1(charin);	
PG_FUNCTION_INFO_V1(charout);
有关版本1调用约定详见官方文档：
https://www.postgresql.org/docs/14/xfunc-c.html#id-1.8.3.13.7

The V1 Function-Manager Interface
【版本1风格的函数管理器接口】
---------------------------------

The core of the design is data structures for representing the result of a
function lookup and for representing the parameters passed to a specific
function invocation.  (We want to keep function lookup separate from
function call, since many parts of the system apply the same function over
and over; the lookup overhead should be paid once per query, not once per
tuple.)
【设计的核心是用于表示函数查找结果和用于表示传递给特定函数调用的参数的数据结构。（我们希望将
 表示函数查找的数据结构与函数调用的数据结构分开，因为系统的许多部分一遍又一遍地应用相同的函数，
 查找开销应该为每个查询支付一次，而不是每个元组一次。】
笔记：
	比如，一个SELECT to_char(col) from test;返回10条元组，to_char函数被调用10次来计算
	并返回每一个查询结果元组对应的函数值，而FMGR将表示函数调用的结构和表示函数查找结果的结构分开
	的设计，可以保证在每个查询中函数只查找一次，这样性能上必然是更优的。这也是PG将函数查找结果
	数据单独使用一个结构体来表示的好处。

When a function is looked up in pg_proc, the result is represented as
【在 pg_proc 中查找一个函数时，其结果表示为FmgrInfo】

typedef struct
{
    PGFunction  fn_addr;    /* pointer to function or handler to be called */
    Oid         fn_oid;     /* OID of function (NOT of handler, if any) */
    short       fn_nargs;   /* number of input args (0..FUNC_MAX_ARGS) */
    bool        fn_strict;  /* function is "strict" (NULL in => NULL out) */
    bool        fn_retset;  /* function returns a set (over multiple calls) */
    unsigned char fn_stats; /* collect stats if track_functions > this */
    void       *fn_extra;   /* extra space for use by handler */
    MemoryContext fn_mcxt;  /* memory context to store fn_extra in */
    Node       *fn_expr;    /* expression parse tree for call, or NULL */
} FmgrInfo;

For an ordinary built-in function, fn_addr is just the address of the C
routine that implements the function.  Otherwise it is the address of a
handler for the class of functions that includes the target function.
The handler can use the function OID and perhaps also the fn_extra slot
to find the specific code to execute.  (fn_oid = InvalidOid can be used
to denote a not-yet-initialized FmgrInfo struct.  fn_extra will always
be NULL when an FmgrInfo is first filled by the function lookup code, but
a function handler could set it to avoid making repeated lookups of its
own when the same FmgrInfo is used repeatedly during a query.)  fn_nargs
is the number of arguments expected by the function, fn_strict is its
strictness flag, and fn_retset shows whether it returns a set; all of
these values come from the function's pg_proc entry.  fn_stats is also
set up to control whether or not to track runtime statistics for calling
this function.
【对于普通的内置函数，fn_addr 只是实现该函数的 C 例程的地址。否则，它是包含目标函数的
 函数类的call handler的地址。call handler可以使用函数 OID 以及 fn_extra 插槽来查
 找要执行的特定代码。（fn_oid = InvalidOid 可用于表示尚未初始化的 FmgrInfo 结构。
 当 FmgrInfo 首次由函数查找代码填充时，fn_extra 将始终为 NULL，但一个函数的handler
 可以设置它以避免重复查找它自己，当在查询期间重复使用相同的 FmgrInfo 时。） fn_nargs 
 是函数期望的参数数量，fn_strict 是它的严格性标志，fn_retset 显示它是否返回一个集合；
 所有这些值都来自函数的 pg_proc 条目。fn_stats 还设置为控制是否跟踪调用此函数的运行时
 统计信息。】

If the function is being called as part of a SQL expression, fn_expr will
point to the expression parse tree for the function call; this can be used
to extract parse-time knowledge about the actual arguments.  Note that this
field really is information about the arguments rather than information
about the function, but it's proven to be more convenient to keep it in
FmgrInfo than in FunctionCallInfoBaseData where it might more logically go.
【如果函数作为 SQL 表达式的一部分被调用，fn_expr 将指向函数调用的表达式分析树；这可用于
 提取有关实际参数的解析时知识。请注意，此字段实际上是有关参数的信息而不是有关函数的信息，
 但事实证明，将其保存在 FmgrInfo 中比保存在 FunctionCallInfoBaseData 中更方便，因为
 它可能更符合逻辑。】

During a call of a function, the following data structure is created
and passed to the function:
【在一个函数的调用期间，会创建以下数据结构并将其传递给这个函数：】
笔记：
	可以简单为函数调用时传递的实参数据结构。

typedef struct
{
    FmgrInfo   *flinfo;         /* ptr to lookup info used for this call */
    Node       *context;        /* pass info about context of call */
    Node       *resultinfo;     /* pass or return extra info about result */
    Oid         fncollation;    /* collation for function to use */
    bool        isnull;         /* function must set true if result is NULL */
    short       nargs;          /* # arguments actually passed */
    NullableDatum args[];       /* Arguments passed to function */
} FunctionCallInfoBaseData;
typedef FunctionCallInfoBaseData* FunctionCallInfo;

flinfo points to the lookup info used to make the call.  Ordinary functions
will probably ignore this field, but function class handlers will need it
to find out the OID of the specific function being called.
【flinfo 指向用于进行（执行）调用的查找信息。普通函数可能会忽略此字段，
 但函数类处理程序（call handler）将需要它来找出被调用的特定函数的OID。】
 笔记：
 	函数调用信息的结构体（eg，表示函数调用时传递的参数）中的第一个字段是FGMR中的另一个结构体，
 	即表示函数查找结果的结构体flinfo，FGMR的核心思想一共就这2个结构体。
 	对于函数类，比如pl语言的函数，需要这个字段，正如上面介绍的那样，plinfo的fn_addr中记载
 	的是pl语言的call handler的地址，但是注意，plinfo的fn_oid记录的可不是call handler
 	自身的函数OID，而是由pl语言编写的用户自定义函数的OID，即将要被调用的特定函数的OID，这是
 	与内置函数不同的地方，内置函数的fn_addr就是C函数的地址，fn_oid就是fn_addr表示的内置函
 	数自己的OID。
 	对于pl函数，fn_addr存储的call handler通过fn_oid和fn_extra来调用特定的用户自定义的
 	过程语言函数。

context is NULL for an "ordinary" function call, but may point to additional
info when the function is called in certain contexts.  (For example, the
trigger manager will pass information about the current trigger event here.)
If context is used, it should point to some subtype of Node; the particular
kind of context is indicated by the node type field.  (A callee should
always check the node type before assuming it knows what kind of context is
being passed.)  fmgr itself puts no other restrictions on the use of this
field.
【对于“普通”函数调用，context为 NULL，但是，当函数在某些上下文中被调用时可能指向额外的信息。
 （例如，触发器管理器将在此处传递有关当前触发器事件的信息。）如果context被使用，它应该指向Node
  的某个子类型；特定类型的context由节点类型字段指示。（被调用者在假设它知道正在传递什么样的
  上下文之前，应该总是检查节点类型。） fmgr本身对该字段的使用没有其他限制。】

resultinfo is NULL when calling any function from which a simple Datum
result is expected.  It may point to some subtype of Node if the function
returns more than a Datum.  (For example, resultinfo is used when calling a
function that returns a set, as discussed below.)  Like the context field,
resultinfo is a hook for expansion; fmgr itself doesn't constrain the use
of the field.
【当调用任何预期是一个简单的Datum结果的函数时，resultinfo 为 NULL。如果函数返回的
 不仅仅是一个 Datum，它可能指向 Node 的一些子类型。（例如，resultinfo 在调用返回
 集合的函数时使用，如下所述。）与 context 字段一样，resultinfo 是扩展的钩子；fmgr
 本身并不限制该字段的使用。】

fncollation is the input collation derived by the parser, or InvalidOid
when there are no inputs of collatable types or they don't share a common
collation.  This is effectively a hidden additional argument, which
collation-sensitive functions can use to determine their behavior.
【fncollation 是由解析器派生的输入排序规则，如果没有可排序类型的输入或它们不共享
 公共排序规则，则为 InvalidOid。这实际上是一个隐藏的附加参数，排序敏感函数可以使用
 它来确定它们的行为。】
    
nargs and args[] hold the arguments being passed to the function.
Notice that all the arguments passed to a function (as well as its result
value) will now uniformly be of type Datum.  As discussed below, callers
and callees should apply the standard Datum-to-and-from-whatever macros
to convert to the actual argument types of a particular function.  The
value in args[i].value is unspecified when args[i].isnull is true.
【nargs 和 args[] 保存传递给函数的参数。请注意，传递给函数的所有参数（以及它的结果值）
 现在都将统一为 Datum 类型。如下所述，调用者和被调用者应该应用标准的 Datum-to和Datum-from宏
 来转换为特定函数的实际参数类型。当 args[i].isnull 为真时，args[i].value 中的值未指定。】
笔记：
	NullableDatum args[]; 存储参数的数组类型NullableDatum表示一个可为空的Datum类型，其实
	就是将Datum封装在一个结构体中，额外增加一个boolean类型变量标记是否是null。

It is generally the responsibility of the caller to ensure that the
number of arguments passed matches what the callee is expecting; except
for callees that take a variable number of arguments, the callee will
typically ignore the nargs field and just grab values from args[].
【调用者通常负责确保传递的参数数量与被调用者的期望相匹配；除了采用可变数量参数的被调用者外，
 可变数量参数的被调用者通常会忽略 nargs 字段，而只是从 args[] 中获取值。】
    
The isnull field will be initialized to "false" before the call.  On
return from the function, isnull is the null flag for the function result:
if it is true the function's result is NULL, regardless of the actual
function return value.  Note that simple "strict" functions can ignore
both isnull and args[i].isnull, since they won't even get called when there
are any TRUE values in args[].isnull.
【isnull字段将在调用之前初始化为“false”。从函数返回时，isnull是函数结果的 null 标志：
 如果为真，则函数的结果为 NULL，而不管实际的函数返回值如何。请注意，简单的“strict”函数
 可以忽略 isnull 和 args[i].isnull，因为当 args[].isnull 中有任何 TRUE 值时，它们
 甚至不会被调用。】
    
FunctionCallInfo replaces FmgrValues plus a bunch of ad-hoc parameter
conventions, global variables (fmgr_pl_finfo and CurrentTriggerData at
least), and other uglinesses.
【FunctionCallInfo 替换了 FmgrValues 加上一堆临时参数约定、全局变量（至少 fmgr_pl_finfo
 和 CurrentTriggerData）以及其他丑陋的东西。】
笔记：
	被替换的这些可能是V0调用约定的东西。。。被淘汰了。

Callees, whether they be individual functions or function handlers,
shall always have this signature:
【被调用者，无论是单个函数还是函数处理程序(call handler)，都应始终具有以下签名：】

Datum function (FunctionCallInfo fcinfo);

which is represented by the typedef

typedef Datum (*PGFunction) (FunctionCallInfo fcinfo);
笔记：
	PGFunction类型是一个函数指针，表示版本1的函数调用约定中对函数声明的格式。那么，
	所有符合此格式的函数都可以被FMGR函数管理器直接调用。eg，我们经常在代码中使用的
	DirectFunctionCall和OidFunctionCall这种宏可以直接调用符合V1约定声明的函数，
	其他函数，比如static函数或动态库或第三方库函数等都可以封装一层V1约定格式的函数，
	然后在调用封装的函数，换句话说，V1约定的函数内部调用其他非V1约定格式的函数，然后
	再被FMGR函数管理器去调用即可。

The function is responsible for setting fcinfo->isnull appropriately
as well as returning a result represented as a Datum.  Note that since
all callees will now have exactly the same signature, and will be called
through a function pointer declared with exactly that signature, we
should have no portability or optimization problems.
【该函数负责适当地设置 fcinfo->isnull 以及返回表示为 Datum 的结果。请注意，由于
 所有被调用者现在将具有完全相同的签名，并且将通过使用该签名声明的函数指针调用，因此
 我们应该没有可移植性或优化问题。】


Function Coding Conventions
【函数编码约定】
---------------------------

Here are the proposed macros and coding conventions:
【以下是建议的宏和编码约定：】

The definition of an fmgr-callable function will always look like
【一个可被fmgr(PG内部的函数管理器)调用的函数的定义总是看起来像】
笔记：
	这不就是上面介绍的PGFunction函数指针嘛。额，PGFunction是PG内部函数管理器
	FMGR的函数指针模板。yes it is。

Datum
function_name(PG_FUNCTION_ARGS)
{
	...
}

"PG_FUNCTION_ARGS" just expands to "FunctionCallInfo fcinfo".  The main
reason for using this macro is to make it easy for scripts to spot function
definitions.  However, if we ever decide to change the calling convention
again, it might come in handy to have this macro in place.
【“PG_FUNCTION_ARGS”仅仅是被展开为“FunctionCallInfo fcinfo”。使用这个宏的主要原因是
让脚本更容易识别函数的定义。但是，如果我们决定再次更改调用约定，则使用此宏可能会派上用场。】
笔记：
	fmgr管理器可直接调用的函数的参数必须声明为FunctionCallInfo fcinfo，这个声明为什么
	非要使用PG_FUNCTION_ARGS宏呢？上面解释了，首先，这个宏更直观，我们看见了就知道这是一
	个fmgr函数；另外一个好处就是：现在的函数调用约定是V1,未来改变函数调用约定为V2的时候，
	我们对于函数参数的修改不必费劲的去修改每一个内置fmgr函数的参数，直接修改这个宏就好了，
	这也是宏定义的常见好处。
想法：
	我记得之前工作中遇到过一个问题：好像是PG的函数返回类型不支持精度，但是Oracle可以？
	比如，create table test as select func(...) from dual; 在oracle创建的表
	是带有精度的，精度就是函数声明时返回类型的精度，而pg好像是不可以？
	记不住了，也不去调研了吧，就是看到这块的介绍之后突然想到好像是有这个一个问题。假设
	是不支持精度，这种情况下，我们是否可以通过将函数调用约定修改为V2，V2调用约定的函数
	参数依然是这个宏，返回类型从Datum修改为一个带精度的类型呢？这样使用V2约定声明的函数
	就可以支持精度，同时保持V1调用约定？可行吗？
    
A nonstrict function is responsible for checking whether each individual
argument is null or not, which it can do with PG_ARGISNULL(n) (which is
just "fcinfo->args[n].isnull").  It should avoid trying to fetch the value
of any argument that is null.
【一个非strict属性的函数负责检查每个单独的参数是否为空，它可以使用宏 PG_ARGISNULL(n) （即“fcinfo->args[n].isnull”）来完成。它应该避免尝试获取任何为空的参数的值。】
    
Both strict and nonstrict functions can return NULL, if needed, with
【如果需要，strict和nonstrict的函数都可以返回NULL，使用下面的宏：】
	PG_RETURN_NULL();
which expands to
【这个宏被展开后是：】
	{ fcinfo->isnull = true; return (Datum) 0; }
笔记：
	注意，必须设置fcinfo->isnull = true; 因为这才是表示函数是否返回NULL的标识，后面的
	return (Datum) 0;表示返回一个无用的返回结果。如果不设置fcinfo->isnull为真，那么
	函数的返回值不会认为是NULL。只有return (Datum) 0;而不是设置fcinfo->isnull=true
    表示的是函数返回VOID。见PG_RETURN_VOID宏的定义。所以返回NULL和VOID是不一样的。

Argument values are ordinarily fetched using code like
【参数值通常使用如下代码获取】
	int32	name = PG_GETARG_INT32(number);

For float4, float8, and int8, the PG_GETARG macros will hide whether the
types are pass-by-value or pass-by-reference.  For example, if float8 is
pass-by-reference then PG_GETARG_FLOAT8 expands to
【对于 float4、float8 和 int8，PG_GETARG 宏将隐藏类型是按值传递还是按引用传递。
 例如，如果 float8 是按引用传递，则 PG_GETARG_FLOAT8 被展开为】
	(* (float8 *) DatumGetPointer(fcinfo->args[number].value))
and would typically be called like this:
【并且通常会这样调用：】
	float8  arg = PG_GETARG_FLOAT8(0);
For what are now historical reasons, the float-related typedefs and macros
express the type width in bytes (4 or 8), whereas we prefer to label the
widths of integer types in bits.
【由于现在的历史原因，与浮点相关的 typedef 和宏以字节（4 或 8）来表现类型的宽度，
 而我们更喜欢以位标记整数类型的宽度。】
    
Non-null values are returned with a PG_RETURN_XXX macro of the appropriate
type.  For example, PG_RETURN_INT32 expands to
【使用适当类型的 PG_RETURN_XXX 宏返回非空值。例如，PG_RETURN_INT32 展开为】
	return Int32GetDatum(x)

PG_RETURN_FLOAT4, PG_RETURN_FLOAT8, and PG_RETURN_INT64 hide whether their
data types are pass-by-value or pass-by-reference, by doing a palloc if
needed.
【PG_RETURN_FLOAT4、PG_RETURN_FLOAT8 和 PG_RETURN_INT64 隐藏它们的数据类型是按值传递
 还是按引用传递，如果需要，执行一个 palloc。】
笔记：
	比如，下面介绍的变长数据类型的宏（PG_GETARG_XXX和PG_RETURN_XXX）就需要PALLOC来
	分配空间。

fmgr.h will provide PG_GETARG and PG_RETURN macros for all the basic data
types.  Modules or header files that define specialized SQL datatypes
(eg, timestamp) should define appropriate macros for those types, so that
functions manipulating the types can be coded in the standard style.
【fmgr.h 将为所有基本数据类型提供 PG_GETARG 和 PG_RETURN 宏。定义专门的SQL数据类型
 的模块或头文件应该为这些类型定义相应的宏（比如，tiemstamp），以便可以以标准样式对操作
 这些类型的函数进行编码。】
    
For non-primitive data types (particularly variable-length types) it won't
be very practical to hide the pass-by-reference nature of the data type,
so the PG_GETARG and PG_RETURN macros for those types won't do much more
than DatumGetPointer/PointerGetDatum plus the appropriate typecast (but see
TOAST discussion, below).  Functions returning such types will need to
palloc() their result space explicitly.  I recommend naming the GETARG and
RETURN macros for such types to end in "_P", as a reminder that they
produce or take a pointer.  For example, PG_GETARG_TEXT_P yields "text *".
【对于非原始数据类型（尤其是可变长度类型），隐藏数据类型的传递引用性质不是很实用，
 因此这些类型的 PG_GETARG 和 PG_RETURN 宏不会做比 DatumGetPointer/PointerGetDatum 
 加上适当的类型转换更多的事情（但请参阅下面的 TOAST 讨论）。返回此类类型的函数将需要显式地
 对其结果空间进行 palloc()。我建议将此类类型的 GETARG 和 RETURN 宏命名为以“_P”结尾，
 以提醒它们产生或获取一个指针。例如，PG_GETARG_TEXT_P 提醒我们，这将会获取一个 “text *”
 指针，同时意味着我们执行palloc（）来显示的分配了空间。】
 
 笔记：
 	变长数据类型（varlena）的 PG_GETARG 和 PG_RETURN宏，都2个特点：
 	第一，变长数据类型的宏都以“_P”结尾，起到见名知意的作用，提示我们这种宏会获取一个指针（对应
 	PG_GETARG宏），或者生成一个指针（对应PG_RETURN宏）然后返回生成的指针，即返回palloc分配
 	的空间。
 	第二，变长数据类型的宏会涉及palloc操作，来分配返回结果的空间。
    
When a function needs to access fcinfo->flinfo or one of the other auxiliary
fields of FunctionCallInfo, it should just do it.  I doubt that providing
syntactic-sugar macros for these cases is useful.
【当一个函数需要访问 fcinfo->flinfo 或 FunctionCallInfo 的其他辅助字段之一时，
 直接访问它。我怀疑为这些情况提供语法糖宏是否有用。】
笔记：
	言外之意，暂时没有访问这些字段的宏？


Support for TOAST-Able Data Types
【支持可TOAST的数据类型】
---------------------------------

For TOAST-able data types, the PG_GETARG macro will deliver a de-TOASTed
data value.  There might be a few cases where the still-toasted value is
wanted, but the vast majority of cases want the de-toasted result, so
that will be the default.  To get the argument value without causing
de-toasting, use PG_GETARG_RAW_VARLENA_P(n).
【对于 TOAST-able 数据类型，PG_GETARG 宏将提供一个de-TOASTed 数据值。可能在
 少数情况下需要still-toasted的值，但绝大多数情况都需要de-toasted的结果，所以
 这将是默认值。要想获取参数值而不导致de-toasted操作，
 请使用 PG_GETARG_RAW_VARLENA_P(n)。】

笔记：
	对于toast数据类型，我们使用宏PG_GETARG获取变长数据类型的参数值时，一般都是先将
	toast数据类型先进行detoast操作，即获取行外存储的真实数据值。而不是行内存储的“指
	针”，即varlena* 类型的指针。
	另外也额外提供了一个宏 PG_GETARG_RAW_VARLENA_P(n) 来获取非detoast的变长数据
	类型的值。

Some functions require a modifiable copy of their input values.  In these
cases, it's silly to do an extra copy step if we copied the data anyway
to de-TOAST it.  Therefore, each toastable datatype has an additional
fetch macro, for example PG_GETARG_TEXT_P_COPY(n), which delivers a
guaranteed-fresh copy, combining this with the detoasting step if possible.
【某些函数需要其输入值的可修改副本。在这些情况下，如果我们要复制数据无论如何都要de-TOAST它，
 那么执行额外的复制步骤是很愚蠢的。因此，每个可以toast的数据类型都有一个额外的 fetch 宏，
 例如 PG_GETARG_TEXT_P_COPY(n)，它提供了保证新鲜的副本，如果可能的话，将它与 
 detoasting步骤结合起来。】

笔记：
	为什么说是很愚蠢的呢？因为对于变长数据类型，在调用PG_GETARG_xxx_P宏获取参数值时，
	本身就要执行detoast操作，而正如上面之前所述的，变长数据类型的detoast操作会执行
	palloc来分配结果空间的，所以，没必要再执行一次“额外”的数据拷贝了，在拷贝一次可不
	就是愚蠢的行为嘛。但是，注意，并不是变长数据类型都会执行detoast操作，只有超过toast
	的阈值（一般是页面大小的1/4）的数据才会采用线外存储或压缩。这种才需要detoast，那么
	也只有这种线外存储的、有toast行为的变长数据才会执行detoast，才会执行palloc分配空间，
	而变长数据类型的数据很小的时候，数据不会采用线外存储和压缩，仍然采用行内存储的情景是不
	会出大toast操作的，这种小数据量的数据是不需要detoast操作的，真实数据本身就存储在
	varlena结构中，或者换句话说真实数据本身就存储在当前元组的缓存中，这种的可修改副本就
	需要在执行一次拷贝了，否则就破坏了当前元组的值了，这就不是愚蠢的行为了，所以，
	为了操作简单，PG提供了一个fetch宏，专门处理获取一个可修改（可写）参数副本的宏：
	PG_GETARG_TEXT_P_COPY(n)。

There is also a PG_FREE_IF_COPY(ptr,n) macro, which pfree's the given
pointer if and only if it is different from the original value of the n'th
argument.  This can be used to free the de-toasted value of the n'th
argument, if it was actually de-toasted.  Currently, doing this is not
necessary for the majority of functions because the core backend code
releases temporary space periodically, so that memory leaked in function
execution isn't a big problem.  However, as of 7.1 memory leaks in
functions that are called by index searches will not be cleaned up until
end of transaction.  Therefore, functions that are listed in pg_amop or
pg_amproc should be careful not to leak detoasted copies, and so these
functions do need to use PG_FREE_IF_COPY() for toastable inputs.
【还有一个 PG_FREE_IF_COPY(ptr,n) 宏，当且仅当它与第 n 个参数的原始值不同时，
 它pfree给定的指针。这可以用来释放第 n 个参数的 detoasted 值，如果它实际上是
 de-toasted 的话。目前，对于大多数函数来说，这样做是没有必要的，因为核心后端代码
 会定期释放临时空间，因此函数执行中的内存泄漏不是什么大问题。但是，从 7.1 开始，
 由索引搜索调用的函数中的内存泄漏在事务结束之前不会被清除。因此，在 pg_amop 或 
 pg_amproc 中列出的函数应该注意不要泄漏detoasted的副本，因此这些函数对于可toast
 数据类型的输入确实需要使用 PG_FREE_IF_COPY()。】

笔记：
	当且仅当它与第 n 个参数的原始值不同时，它pfree给定的指针。与原始值不同说明这个
	第n个参数是一个可修改的拷贝，即使用PG_GETARG_TEXT_P_COPY(n)宏获取的参数值，
	在使用完之后要释放掉。

A function should never try to re-TOAST its result value; it should just
deliver an untoasted result that's been palloc'd in the current memory
context.  When and if the value is actually stored into a tuple, the
tuple toaster will decide whether toasting is needed.
【一个函数不应该尝试重新TOAST它的结果值；它应该只是交付一个在当前内存上下文中palloc 
 的untoasted结果值。当值实际存储到元组中时，元组toaster将决定是否需要toast（阈值大于
 页面大小的1/4？）。】
    

Functions Accepting or Returning Sets
【接受或返回集合的函数】
-------------------------------------

If a function is marked in pg_proc as returning a set, then it is called
with fcinfo->resultinfo pointing to a node of type ReturnSetInfo.  A
function that desires to return a set should raise an error "called in
context that does not accept a set result" if resultinfo is NULL or does
not point to a ReturnSetInfo node.
【如果一个函数在 pg_proc 中被标记为返回一个集合，那么调用它时 fcinfo->resultinfo 指向一个  ReturnSetInfo 类型的节点。如果 resultinfo 为 NULL 或不指向 ReturnSetInfo 节点，
 则希望返回集合的函数应发出错误“在不接受集合结果的上下文中调用”。】

There are currently two modes in which a function can return a set result:
value-per-call, or materialize.  In value-per-call mode, the function returns
one value each time it is called, and finally reports "done" when it has no
more values to return.  In materialize mode, the function's output set is
instantiated in a Tuplestore object; all the values are returned in one call.
Additional modes might be added in future.
【目前有两种模式可以让函数返回一个集合结果：value-per-call或materialize。
 在value-per-call 模式下，函数每次调用都返回一个值，最后在没有更多值返回时报告“done”完毕。
 在materialize物化模式下，函数的输出集被实体化到 Tuplestore 对象中；
 所有值都在一次调用中返回。将来可能会添加其他模式。】
笔记：
	返回集合的函数有2中返回结果集的模式：
	（1）value-per-call：调用多次，每次返回一个值。
	（2）materialize：调用一次，一次返回所有值，所有返回值被实体化在Tuplestore中。
	（3）Tuplestore -- 元组存储，是一种临时存储元组的技术，见tuplestore.c中。
    
ReturnSetInfo contains a field "allowedModes" which is set (by the caller)
to a bitmask that's the OR of the modes the caller can support.  The actual
mode used by the function is returned in another field "returnMode".  For
backwards-compatibility reasons, returnMode is initialized to value-per-call
and need only be changed if the function wants to use a different mode.
The function should ereport() if it cannot use any of the modes the caller is
willing to support.
【ReturnSetInfo 包含一个字段“allowedModes”，它（由调用者）设置为一个位掩码，
 该位掩码是调用者可以支持/处理的返回模式的 OR。该函数使用的实际模式在另一个字段
 “returnMode”中返回。出于向后兼容的原因，returnMode 被初始化为value-per-call模式，
 并且只有在函数想要使用不同的模式时才需要更改。如果函数不能使用调用者设置的可以处理的
 任何模式（allowedModes？），则该函数应发出ereport()。】
    
Value-per-call mode works like this: ReturnSetInfo contains a field
"isDone", which should be set to one of these values:
【Value-per-call 模式的工作方式如下：
 ReturnSetInfo 包含一个字段“isDone”，该字段应设置为以下值之一：】

    ExprSingleResult             /* expression does not return a set */
    ExprMultipleResult           /* this result is an element of a set */
    ExprEndResult                /* there are no more elements in the set */

(the caller will initialize it to ExprSingleResult).  If the function simply
returns a Datum without touching ReturnSetInfo, then the call is over and a
single-item set has been returned.  To return a set, the function must set
isDone to ExprMultipleResult for each set element.  After all elements have
been returned, the next call should set isDone to ExprEndResult and return a
null result.  (Note it is possible to return an empty set by doing this on
the first call.)
【（调用者会将其初始化为 ExprSingleResult）。如果函数只返回一个 Datum 而没有触及  ReturnSetInfo，则调用结束并返回一个单项集。要返回一个集合，该函数必须为每个集合元素
 将isDone设置为 ExprMultipleResult。在所有元素被返回后，下一次调用应将isDone设置为  ExprEndResult 并返回空结果。 （请注意，在第一次调用时执行此操作可能会返回一个空集。）】
笔记：
	对于Value-per-call模式，每个集合元组都有对应一个ReturnSetInfo结构吗？
	每个集合元组都对应一个FunctionCallInfo结构吗？因为没看代码所以不确定，
	不过理解上可能是这样，因为前文中说道：这个模式会调用多次，所以必然要有多个
	FunctionCallInfo结构了，那自然也就有多个ReturnSetInfo结构了。

Value-per-call functions MUST NOT assume that they will be run to completion;
the executor might simply stop calling them, for example because of a LIMIT.
Therefore, it's unsafe to attempt to perform any resource cleanup in the
final call.  It's usually not necessary to clean up memory, anyway.  If it's
necessary to clean up other types of resources, such as file descriptors,
one can register a shutdown callback function in the ExprContext pointed to
by the ReturnSetInfo node.  (But note that file descriptors are a limited
resource, so it's generally unwise to hold those open across calls; SRFs
that need file access are better written to do it in a single call using
Materialize mode.)
【Value-per-call 返回模式的函数不能假定它们将运行到完成；执行器可能会简单地停止调用它们，
 例如因为LIMIT。因此，尝试在最终调用中执行任何资源清理是不安全的。无论如何，通常不需要清理内存。
 如果需要清理其他类型的资源，例如文件描述符，可以在 ReturnSetInfo 节点的 ExprContext
 中注册一个shutdown回调函数。（但请注意，文件描述符是一种有限的资源，因此在调用之间保持打开
 通常是不明智的；需要文件访问的 SRF 最好使用 Materialize 模式在单个调用中完成。）】
    
Materialize mode works like this: the function creates a Tuplestore holding
the (possibly empty) result set, and returns it.  There are no multiple calls.
The function must also return a TupleDesc that indicates the tuple structure.
The Tuplestore and TupleDesc should be created in the context
econtext->ecxt_per_query_memory (note this will *not* be the context the
function is called in).  The function stores pointers to the Tuplestore and
TupleDesc into ReturnSetInfo, sets returnMode to indicate materialize mode,
and returns null.  isDone is not used and should be left at ExprSingleResult.
【Materialize 模式的工作方式如下：
 该函数创建一个保存（可能为空）结果集的 Tuplestore，并将其返回。没有多次调用。
 该函数还必须返回一个表示元组结构的 TupleDesc。 Tuplestore 和 TupleDesc 
 应该在上下文 econtext->ecxt_per_query_memory 中创建（注意这将*不是*是函数被调用的上下文）。
 该函数将指向 Tuplestore 和 TupleDesc 的指针存储到 ReturnSetInfo 中，设置
 returnMode 以表示物化模式，并返回 null。isDone没有被使用，其值应保持ExprSingleResult
 （如上所述，ExprSingleResult是初始没有被）】
笔记
	物化模式返回NULL，结果保存在Tuplestore中？

The Tuplestore must be created with randomAccess = true if
SFRM_Materialize_Random is set in allowedModes, but it can (and preferably
should) be created with randomAccess = false if not.  Callers that can support
both ValuePerCall and Materialize mode will set SFRM_Materialize_Preferred,
or not, depending on which mode they prefer.
【如果在 allowedModes 中设置了 SFRM_Materialize_Random，则必须使用 
 randomAccess = true 创建 Tuplestore，但如果没有，则可以（并且最好应该）使用
 randomAccess = false 创建它。可以同时支持 ValuePerCall 和 Materialize 模式
 的调用者将设置 SFRM_Materialize_Preferred 或不设置，这取决于哪种模式是首选的。】
    
If available, the expected tuple descriptor is passed in ReturnSetInfo;
in other contexts the expectedDesc field will be NULL.  The function need
not pay attention to expectedDesc, but it may be useful in special cases.
【如果可用，则在 ReturnSetInfo 中传递预期的元组描述符（expectedDesc）；
 在其他上下文中，expectedDesc字段将为 NULL。该函数不需要关注expectedDesc，
 但在特殊情况下可能有用。】
    
There is no support for functions accepting sets; instead, the function will
be called multiple times, once for each element of the input set.
【不支持接受集合的函数（也就是说，函数的返回值可以是一个集合，但是函数的输入参数不能是一个几个）；
 替代的方式是，该函数将被调用多次，对输入集的每个元素都调用一次。】
笔记：
	函数的返回类型可以是一个集合，输入参数不允许是一个集合。


Notes About Function Handlers
【关于函数处理程序的注意事项】
-----------------------------

Handlers for classes of functions should find life much easier and
cleaner in this design.  The OID of the called function is directly
reachable from the passed parameters; we don't need the global variable
fmgr_pl_finfo anymore.  Also, by modifying fcinfo->flinfo->fn_extra,
the handler can cache lookup info to avoid repeat lookups when the same
function is invoked many times.  (fn_extra can only be used as a hint,
since callers are not required to re-use an FmgrInfo struct.
But in performance-critical paths they normally will do so.)
【在PG这种函数的设计模式中，函数类的处理程序（call handler）应该会发现生活变得更加轻松和简洁。
 被调用函数的OID可以直接从传入的参数中得到（FunctionCallInfo）；
 我们不再需要全局变量 fmgr_pl_finfo。V0函数调用约定有这个全局变量？
 此外，通过修改 fcinfo->flinfo->fn_extra，处理程序可以缓存函数的查找结果信息，
 以避免在多次调用同一函数时重复查找。（fn_extra 只能作为提示使用，因为调用者不需要
 重新使用 FmgrInfo 结构。但在性能关键路径中，它们通常会这样做。）】

笔记：
	FmgrInfo的fn_addr是一个重用字段，其类型是PGFunction，如上所述，我们知道这是一个函数
	指针，这个函数指针表示的就是PG V1调用约定的函数声明格式。也就是可以被PG的函数管理器直接
	调用的函数应该以这个函数模型来声明和定义。
	对于普通的内置函数fn_addr就是其C例程的函数地址，但是对于plpgsql函数时fn_addr则表示的
	是函数类的处理程序，即plpgsql的handler函数的地址，所以说plpgsql等函数类的handle处理
	程序函数也必须是V1调用约定的格式来声明和定义，也就是，handle可以被PG的FMGR直接调用哦。

If the handler wants to allocate memory to hold fn_extra data, it should
NOT do so in CurrentMemoryContext, since the current context may well be
much shorter-lived than the context where the FmgrInfo is.  Instead,
allocate the memory in context flinfo->fn_mcxt, or in a long-lived cache
context.  fn_mcxt normally points at the context that was
CurrentMemoryContext at the time the FmgrInfo structure was created;
in any case it is required to be a context at least as long-lived as the
FmgrInfo itself.
【如果处理程序想要分配内存来保存 fn_extra 数据，它不应该在 CurrentMemoryContext 中这样做，
 因为当前的内存上下文可能比 FmgrInfo 所在的内存上下文的声明周期更短。相反，在上下文
 flinfo->fn_mcxt 或长期声明周期的内存上下文中分配内存。fn_mcxt 通常指向创建 FmgrInfo 
 结构时的 CurrentMemoryContext；在任何情况下，它都必须是至少与 FmgrInfo 本身一样长的上下文。】

笔记：
	主要是注意fn_extra指向内存的声明周期，不要太短。至少要大于等于FmgrInfo的声明周期。
```

总结：

​	下一步分析代码，所以有些理借可能是错误的或不准确的，阅读完代码后更新笔记。


