---
layout: post
title: 编写PostgreSQL的extenison教程1-基础知识
categories: extension
description: how to write an extension
keywords: extension-dev
---

# 编写PostgreSQL的extenison教程1-基础知识

## 背景

Postgres具有大量的功能，并提供各种数据类型，函数，运算符和聚合运算，但是有的时候这仍然无法满足你的用例。幸运的是，通过extension可以很容易的扩展Postgres的功能，那为什么不写一个自己的extension呢？

这是有关通过extension来扩展Postgres系列文章中的第一篇，这是一个extenison代码的[demo](https://github.com/adjust/postgresql_extension_demo/tree/part_i)示例。

## base36

你可能已经知道了短网址URL使用的技巧，使用一些独特的随机字符（例如[http://goo.gl/EAZSKW](http://goo.gl/EAZSKW)）指向其他内容。当然，你必须要记住指向何处的信息，因此需要将其存储在数据库中。但是为什么不使用4个字节的整数并将其表示为[base36](https://en.wikipedia.org/wiki/Base36)来代替使用varchar（6）保存6个字符（因此会浪费7个字节）呢？

## extension的框架

为了能够在数据库中运行[CREATE EXTENSION](http://www.postgresql.org/docs/9.4/static/sql-createextension.html)命令，你的extension至少需要两个文件：一个格式为`extension.control`的控制文件，该文件告诉Postgres有关该extension名的一些基础内容，以及一个格式为`extension--version.sql`的SQL脚本文件。因此，让我们将它们添加到我们的项目目录中。

对于我们的控制文件，一个很好的起点可能是：

<center>base36.control</center>
```shell
#base36 extension
comment = 'base36 datatype'
default_version = '0.0.1'
relocatable = true
```

截至目前，我们的extension还没有任何功能，让我们在SQL脚本文件中添加一些内容：

<center>base36–0.0.1.sql</center>
```sql
-- complain if script is sourced in psql, rather than via CREATE EXTENSION
\echo Use "CREATE EXTENSION base36" to load this file. \quit
CREATE FUNCTION base36_encode(digits int)
RETURNS text
LANGUAGE plpgsql IMMUTABLE STRICT
  AS $$
    DECLARE
      chars char[];
      ret varchar;
      val int;
    BEGIN
      chars := ARRAY[
                '0','1','2','3','4','5','6','7','8','9','a','b','c','d','e','f','g','h',
                'i','j','k','l','m','n','o','p','q','r','s','t', 'u','v','w','x','y','z'
              ];

      val := digits;
      ret := '';

    WHILE val != 0 LOOP
      ret := chars[(val % 36)+1] || ret;
      val := val / 36;
    END LOOP;

    RETURN(ret);
    END;
  $$;
```

第二行确保了SQL文件不会被直接加载到数据库中，而只能通过CREATE EXTENSION加载。

这个简单的plpgsql函数允许我们将任何整数编码为其base36表示形式。如果我们将这两个文件复制到postgres的`SHAREDIR/extension`目录中，那么我们就可以通过`CREATE EXTENSION`来使用这个extension。但是，我们不会去麻烦用户来弄清楚这些文件的放置位置以及如何手动复制它们-这是Makefile干的事。因此，我们将添加一个Makefile到我们的项目中。

## Makefile

从9.1版开始的每个PostgreSQL的安装都会为extenison提供一个编译的基础设施叫做PGXS，针对一个已安装的Postgres服务可以很轻松的编译extension。编译extension所需的大多数环境变量都设置在`pg_config`中，可以简单地重用。

对于我们的示例，此Makefile符合我们的需求：

<center>Makefile</center>
```makefile
EXTENSION = base36        # the extensions name
DATA = base36--0.0.1.sql  # script files to install

# postgres build stuff
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

现在，我们可以开始使用这个extension了，在extension项目目录中运行：

```shell
`make install`
```

然后，在数据库执行：

```sql
test=# CREATE EXTENSION base36;
CREATE EXTENSION
Time: 3,329 ms
test=# SELECT base36_encode(123456789);
 base36_encode
---------------
 21i3v9
(1 row)

Time: 0,558 ms
```

## 编写回归测试

如今，每个严谨的开发人员都会编写测试，作为处理数据（可能是公司中最有价值的东西）的数据库开发人员，你也应该这样做。

你可以很容易地向项目中添加一些回归测试，这些测试用例可以在执行`make install`之后由`make installcheck`调用。为此，你可以将测试脚本文件放在名为`sql/`的子目录中。对于每个测试文件，还应该有一个对应的同名文件，该文件在名为`expected/`的子目录中包含期望的输出，文件的扩展名为`.out`。`make installcheck`命令使用psql执行每个测试脚本，并将输出结果与匹配的预期文件进行比较。任何差异都将写入文件`regression.diffs`，让我们这样做：

<center>sql/base36_test.sql</center>
```sql
CREATE EXTENSION base36;
SELECT base36_encode(0);
SELECT base36_encode(1);
SELECT base36_encode(10);
SELECT base36_encode(35);
SELECT base36_encode(36);
SELECT base36_encode(123456789);
```

我们还需要告诉我们的Makefile有关测试（第3行）的信息：

<center>Makefile</center>
```makefile
EXTENSION = base36        # the extensions name
DATA = base36--0.0.1.sql  # script files to install
REGRESS = base36_test     # our test script file (without extension)

# postgres build stuff
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

如果现在运行`make install && make installcheck`，那么我们的测试将失败，这是因为我们没有指定预期的输出。但是，我们会找到新的目录`results`，其中包含`base36_test.out`和`base36_test.out.diff`，前者包含我们测试脚本文件的实际输出，让我们将其移至所需目录（`expected/`）。

```shell
`mkdir expected mv results/base36_test.out expected`
```

如果现在重新运行测试，则会看到类似以下的内容：

```shell
============== running regression test queries        ==============
test base36_test              ... ok

=====================
 All 1 tests passed.
=====================
```

Nice！ 但是，我们有点作弊了，如果我们看看我们的期望，就会发现这不是我们应该期望的。

```shell
cat expected/base36_test.out
CREATE EXTENSION base36;
SELECT base36_encode(0);
 base36_encode
---------------

(1 row)

SELECT base36_encode(1);
 base36_encode
---------------
 1
(1 row)

SELECT base36_encode(10);
 base36_encode
---------------
 a
(1 row)

SELECT base36_encode(35);
 base36_encode
---------------
 z
(1 row)

SELECT base36_encode(36);
 base36_encode
---------------
 10
(1 row)

SELECT base36_encode(123456789);
 base36_encode
---------------
 21i3v9
(1 row)
```

你会注意到，在第6行中，`base36_encode（0）`在期望返回为0的地方返回了一个空字符串，如果我们修正了期望输出，测试将再次失败。

```shell
============== running regression test queries        ==============
test base36_test              ... FAILED

======================
 1 of 1 tests failed.
======================

The differences that caused some tests to fail can be viewed in the
file "regression.diffs".  A copy of the test summary that you see
above is saved in the file "regression.out".

make: *** [installcheck] Error 1
```

通过查看提到的`regression.diffs`，我们可以轻松检查失败的测试。

```sql
*** 2,8 ****
  SELECT base36_encode(0);
   base36_encode
  ---------------
!  0
  (1 row)

  SELECT base36_encode(1);
--- 2,8 ----
  SELECT base36_encode(0);
   base36_encode
  ---------------
!
  (1 row)

  SELECT base36_encode(1);
```

现在，让我们修复encoding函数，以使测试再次通过（第12-14行）：

<center>base36–0.0.1.sql</center>
```sql
-- complain if script is sourced in psql, rather than via CREATE EXTENSION
\echo Use "CREATE EXTENSION base36" to load this file. \quit
CREATE FUNCTION base36_encode(digits int)
RETURNS character varying
LANGUAGE plpgsql IMMUTABLE STRICT
  AS $$
    DECLARE
      chars char[];
      ret varchar;
      val int;
    BEGIN
      IF digits = 0
        THEN RETURN('0');
      END IF;
      chars := ARRAY[
                '0','1','2','3','4','5','6','7','8','9','a','b','c','d','e','f','g','h',
                'i','j','k','l','m','n','o','p','q','r','s','t', 'u','v','w','x','y','z'
              ];

      val := digits;
      ret := '';

    WHILE val != 0 LOOP
      ret := chars[(val % 36)+1] || ret;
      val := val / 36;
    END LOOP;

    RETURN(ret);
    END;
  $$;
```

## 速度优化，编写C代码

尽管对于代码共享来说在一个extension中提供相关功能是一个便捷的方式，但真正的乐趣始于用C实现东西，让我们来得到第一个1000000（1M）个base36数值。

```SQL
test=# SELECT i, base36_encode(i) FROM generate_series(1,1e6::int) i; 
Time: 11289,610 ms
```

11s？ …嗯，不是那么的快。

让我们看看C语言是否可以做的更好，编写C语言函数（ [C-Language Functions](http://www.postgresql.org/docs/9.4/static/xfunc-c.html)）并不难。

<center>base36.c</center>
```c
#include "postgres.h"
#include "fmgr.h"
#include "utils/builtins.h"

PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(base36_encode);
Datum
base36_encode(PG_FUNCTION_ARGS)
{
    int32 arg = PG_GETARG_INT32(0);
    char base36[36] = "0123456789abcdefghijklmnopqrstuvwxyz";

    /* max 6 char + '\0' */
    char *buffer = palloc(7 * sizeof(char));
    unsigned int offset = sizeof(buffer);
    buffer[--offset] = '\0';

    do {
        buffer[--offset] = base36[arg % 36];
    } while (arg /= 36);


    PG_RETURN_TEXT_P(cstring_to_text(&buffer[offset]));
}
```

你可能已经注意到，实际的算法是[Wikipedia](https://en.wikipedia.org/wiki/Base36)提供的算法，让我们看看我们添加了哪些东西以使其可以与Postgres一起工作。

`#include "postgres.h"`包括Postgres接口所需的大多数基本内容。此行需要在每个声明了Postgres函数的C文件中被包含。

`#include "fmgr.h"`需要包含`#include "fmgr.h"`才能使用`PG_GETARG_XXX`和`PG_RETURN_XXX`宏。

`#include "utils/builtins.h"`定义了Postgres内置数据类型的某些操作（例如：`cstring_to_text`）

`PG_MODULE_MAGIC`是PostgreSQL 8.2以来所需的一个“*magic block*”。“*magic block*”的作用主要用来验证已加载extension的后端兼容性，这样我们就可以检查是否存在明显的不兼容，例如编译和加载（使用）extenison的PostgreSQL版本的主版本号不一致的问题。请注意，在多源文件extension中，宏调用仅应出现一次，也就是说在extension的多个C源文件中，只需要在其中任何一个C文件中调用此宏即可，记得要包含头文件`fmgr.h`。

`PG_FUNCTION_INFO_V1(base36_encode);` 将该函数作为[版本1调用约定](https://www.postgresql.org/docs/9.4/xfunc-c.html#AEN55804)引入到Postgres中，只有在你希望该函数成为Postgres接口时才需要调用此宏。

`Datum`是每个C语言Postgres函数的返回类型，可以是任何数据类型，你可以将其视为类似于`void *`指针。

`base36_encode（PG_FUNCTION_ARGS）`我们的函数被命名为`base36_encode`，参数宏`PG_FUNCTION_ARGS`，可以接受任何数量和任何类型的参数。

`int32 arg = PG_GETARG_INT32(0);` 获取第一个参数，参数的编号从0开始，你必须使用[fmgr.h](https://github.com/postgres/postgres/blob/b7ca57ac0e80b8b511780ef1f19fa2124c901efb/src/include/fmgr.h#L224)中定义的`PG_GETARG_XXX`宏来获取实际的参数值。

`char *buffer = palloc(7 * sizeof(char));` 为了防止在分配内存时发生内存泄漏，请始终使用PostgreSQL函数palloc和pfree而不是相应的C库函数malloc和free。由palloc分配的内存将在每个事务结束时自动释放，你也可以使用palloc0来确保字节被初始化为0。

`PG_RETURN_TEXT_P(cstring_to_text(&buffer[offset]));`要想将值返回给Postgres，你必须总是使用PG_RETURN_XXX宏，`cstring_to_text`将cstring转换为Postgres的文本类型text。

一旦我们完成了C的代码，我们需要修改SQL函数：

<center>base36–0.0.1.sql</center>

```SQL
-- complain if script is sourced in psql, rather than via CREATE EXTENSION
\echo Use "CREATE EXTENSION base36" to load this file. \quit
CREATE FUNCTION base36_encode(integer) RETURNS text
AS '$libdir/base36'
LANGUAGE C IMMUTABLE STRICT;
```

为了能够使用该函数，我们还需要修改Makefile（第4行）

<center>Makefile</center>

```makefile
EXTENSION = base36        # the extensions name
DATA = base36--0.0.1.sql  # script files to install
REGRESS = base36_test     # our test script file (without extension)
MODULES = base36          # our c module file to build

# postgres build stuff
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

我们之前已经进行过测试了，可以通过`make install && make installcheck`再次进行测试，打开一个数据库控制台也能证明它快很多（30倍）：

```SQL
`test=# SELECT i, base36_encode(i) FROM generate_series(1,1e6::int) i; Time: 361,054 ms`
```

## 返回错误

你可能已经注意到了，我们的简单实现不适用于负数，就像之前使用0一样，它将返回一个空字符串。我们可能想为负值添加一个负号“-”或者只简单的抛出一个错误，我们选择了后者（12-20行）。

<center>base36.c</center>

```c
#include "postgres.h"
#include "fmgr.h"
#include "utils/builtins.h"

PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(base36_encode);
Datum
base36_encode(PG_FUNCTION_ARGS)
{
    int32 arg = PG_GETARG_INT32(0);
    if (arg < 0)
        ereport(ERROR,
            (
             errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),
             errmsg("negative values are not allowed"),
             errdetail("value %d is negative", arg),
             errhint("make it positive")
            )
        );
    char base36[36] = "0123456789abcdefghijklmnopqrstuvwxyz";

    /* max 6 char + '\0' */
    char *buffer = palloc(7 * sizeof(char));
    unsigned int offset = sizeof(buffer);
    buffer[--offset] = '\0';

    do {
        buffer[--offset] = base36[arg % 36];
    } while (arg /= 36);


    PG_RETURN_TEXT_P(cstring_to_text(&buffer[offset]));
}
```

这将导致:

```sql
test=# SELECT base36_encode(-10);
ERROR:  negative values are not allowed
DETAIL:  value -10 is negative
HINT:  make it positive
```

Postgres内置了一些不错的[错误报告功能](https://www.postgresql.org/docs/9.4/error-message-reporting.html)，虽然在此示例中使用一个简单的`errmsg`就足够了，但是你可以（但并不需要）添加错误的详细信息（detail）、提示（hint）等。

要进行简单的调试，可以使用

`elog(INFO, "value here is %d", value);`

`INFO`的错误级别只会导致插入一条log日志消息，而不会立即停止该函数调用，错误消息的严重级别从`DEBUG`到`PANIC`。

## 更多...

现在，我们已经知道编写extensions和C语言函数的基础框架，[在下一篇文章中]()，我们将继续下一步操作并实现一个完整的新数据类型。