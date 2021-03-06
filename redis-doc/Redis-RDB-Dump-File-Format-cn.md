﻿

**译者： coderbee (wen866595@163.com)   
转载请注明出处**

翻译自：
<https://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-RDB-Dump-File-Format>



# Redis RDB 文件格式
Redis *.rdb 文件是一个内存内存储的二进制表示法。这个二进制文件足以完全恢复Redis的状态。

rdb文件格式为快速读和写优化。LZF压缩可以用来减少文件大小。通常，对象前面有它们的长度，
这样，在读取对象之前，你可以准确地分配内存大小。


为快速读/写优化意味着磁盘上的格式应该尽可能接近于在内存里的表示法。这种方式正是rdb文件采用的。
导致的结果是，在不了解Redis在内存里表示数据的数据结构的情况下，你没法解析rdb文件。


## 解析RDB的高层算法
在高层层面看，RDB文件有下面的格式：
<pre><code>
----------------------------# RDB 是一个二进制文件。文件里没有新行或空格。
52 45 44 49 53              # 魔术字符串 "REDIS"
00 00 00 03                 # RDB 版本号，高位优先。在这种情况下，版本是 0003 = 3
----------------------------
FE 00                       # FE = code 指出数据库选择器. 数据库号 = 00
----------------------------# 键值对开始
FD $unsigned int            # FD 指出 "有效期限时间是秒为单位". 在这之后，读取4字节无符号整数作为有效期限时间。
$value-type                 # 1 字节标记指出值的类型 － set，map，sorted set 等。
$string-encoded-key         # 键，编码为一个redis字符串。
$encoded-value              # 值，编码取决于 $value-type.
----------------------------
FC $unsigned long           # FC 指出 "有效期限时间是豪秒为单位". 在这之后，读取8字节无符号长整数作为有效期限时间。
$value-type                 # 1 字节标记指出值的类型 － set，map，sorted set 等。
$string-encoded-key         # 键，编码为一个redis字符串。
$encoded-value              # 值，编码取决于 $value-type.
----------------------------
$value-type                 # 这个键值对没有有效期限。$value_type 保证 != to FD, FC, FE and FF
$string-encoded-key
$encoded-value
----------------------------
FE $length-encoding         # 前一个数据库结束，下一个数据库开始。数据库号用长度编码读取。
----------------------------
...                         # 这个数据库的键值对，另外的数据库。
FF                          ## RDB 文件结束指示器
8 byte checksum             ## 整个文件的 CRC 32 校验和。
</code></pre>


### 魔术数
文件开始于魔术字符串“REDIS”。这是一个快速明智的检查是否正在处理一个redis rdb文件。
    52 45 44 49 53 # "REDIS"


### RDB 版本号
接下来4个字节存储了rdb格式的版本号。这4个字节解释为ascii字符，然后使用字符串到整数的转换法转换为一个整数。
    00 00 00 03 # Version = 3


### 数据库选择器
一个Redis实例可以有多个数据库。
单一字节0xFE标记数据库选择器的开始。在这个字节之后，一个可变长度的字段指出数据库序号。
见“长度编码”章节来了解如何读取数据库序号。


### 键值对
在数据库选择器之后，文件包含了一序列的键值对。

每个键值对有4部分：
>  1.  键保存期限时间戳。这是可选的。
>  2.  一个字节标记值的类型。
>  3.  键编码为Redis 字符串。见“Redis 字符串编码”。
>  4.  值根据值类型进行编码。见“Redis 值编码”。


#### 键保存期限时间戳
这个区块开始于一字节标记。值FD指出保存期限是以秒为单位指定。值FC指出有效期限是以毫秒为单位指定。

如果时间指定为毫秒，接下来8个字节表示unix时间。这个数字是unix时间戳，精确到秒或毫秒，表示这个键的有效期限。

数字如何编码见“Redis 长度编码”章节。

在导入过程中，已经过期的键将必须丢弃。


#### 值类型
一个字节标记指示用于保存值的编码。

>  1.  0 ＝ “String 编码”
>  2.  1 ＝ “ List 编码”
>  3.  2 ＝ “Set 编码”
>  4.  3 ＝ “Sorted Set 编码”
>  5.  4 ＝ “Hash 编码”
>  6.  9 ＝ “Zipmap 编码”
>  7.  10 ＝ “Ziplist 编码”
>  8.  11 ＝ “IntSet 编码”
>  9.  12 ＝ “以 Ziplist 编码的 Sorted Set”
>  10.  13 ＝ “以 Ziplist 编码的 Hashmap” （在rdb版本4中引入）


#### 键
键简单地编码为Redis字符串。见“字符串编码”章节了解键如何被编码。


#### 值
值的编码取决于值类型标记。

* 当值类型 ＝ 0，值是简单字符串。
* 当值类型是 9， 10， 11 或 12 中的一个，值被包装为字符串。读取字符串后，它必须进一步解析。
* 当值类型是1，2，3 或 4 中的一个，值是一序列字符串。这个序列字符串用于构造list，set，sorted set或hashmap。


## 长度编码
长度编码用于存储流中接下来对象的长度。长度编码是一个按的字节编码，为尽可能少用字节而设计。

这是长度编码如何工作：
> 1.   从流中读取一个字节，最高两bit被读取
> 2.  如果开始bit是 00 ，接下来6bit 表示长度
> 3.  如果开始bit是 01，从流再读取额外一个字节。这组合的的14bit表示长度
> 4.  如果开始bit是 10，那么剩余的6bit丢弃，从流中读取额外的4字节，这4个字节表示长度
> 5.  如果开始bit是 11，那么接下来的对象是以特殊格式编码的。剩余6bit指示格式。这种编码通常用于把数字作为字符串存储
        或存储编码后的字符串。见字符串编码

作为这种编码的结果：
>  1.  数字 [0 - 63] 可以在1个字节里存储
>  2.  数字 [0 - 16383] 可以在2个字节里存储
>  3.  数字 [0 - (2^32 - 1)] 可以在4个字节里存储


## 字符串编码
Redis字符串是二进制安全的－－这意味着你可以在这里存储任何东西。它们没有任何特殊的字符串结束记号。
最好认为Redis字符串是一个字节数组。

Redis里有三种类型的字符串：
> 1.  长度前缀字符串
> 2.  一个8，16或32bit整数
> 3.  LZF压缩的字符串


#### 长度前缀字符串
长度前置字符串是很简单的。字符串字节的长度首先编码为“长度编码”，在这之后存储字符串的原始字节。


#### 整数作为字符串
首先读取“长度编码”块，特别是第一个两bit是 11。在这种情况下，读取剩余的6bit。如果这6bit的值是：
> 1.  0 表示接下来是8bit整数
> 2.  1 表示接下来是16bit整数
> 3.  2 表示接下来是32bit整数


#### 压缩字符串
首先读取“长度编码”，特别是第一个两bit是 11. 在这种情况下，读取剩余6bit。如果这6bit值是4，它表示接下来是一个压缩字符串。

压缩字符串按如下读取：
> 1.  从流中读取压缩后的长度clen，按“长度编码”
> 2.  从流中读取未压缩长度，按“长度编码”
> 3.  接下来从流中读取clen个字节
> 4.  最后，这些字节按LZF算法解压


## List 编码
一个Redis list 表示为一序列字符串。

> 1.  首先，从流中读取list大小size，按“长度编码”
> 2.  然后，size个字符串从流中读取，按“字符串编码”
> 3.  使用这些字符串重新构建list


## Set 编码
Set 编码与list完全类似。


## Sorted Set 编码
> 1.  首先，从流中读取sorted set大小size，按“长度编码”
> 2.  TODO


## Hash 编码
> 1.  首先，从流中读取hash大小size，按“长度编码”
> 2.  下一步，从流中读取 2 * size 个字符串，按“字符串编码”
> 3.  交替的字符串是键和值
> 4.  例如，2 us washington india delhi 表示map {"us" => "washington", "india" => "dlhi"}


## Zipmap 编码
*注意：Zipmap编码从Redis 2.6开始已弃用。小的的hashmap编码为ziplist。*

Zipmap是一个被序列化为一个字符串的hashmap。本质上，键值对按顺序存储。在这种结构里查找一个键的复杂度是O(N)。
当键值对数量很少时，这个结构用于替代dictionary。

为解析zipmap，首先用“字符串编码”从流读取一个字符串。这个字符串包装了zipmap。字符串的内容表示了zipmap。

字符串里的zipmap结构如下：
<pre><code>
  <zmlen><len>"foo"<len><free>"bar"<len>"hello"<len><free>"world"<zmend>

  1.  zmlen : 1字节长，保存zipmap的大小. 如果大于等于254，值不使用。将需要迭代整个zipmap来找出长度.
  2.  len : 后续字符串的长度，可以是键或值的。这个长度存储为1个或5个字节（与上面描述的“长度编码”不同）。
            如果第一个字节位于 0 到252，那么它是zipmap的长度。如果第一个字节是253，读取下4个字节作为无符号整数来表示zipmap的长度。
            254 和 255 对这个字段是非法的. 
  3.  free : 总是1字节，指出值后面的空闲字节数。例如，如果键的值是“America”，更新为“USA”后，将有4个空闲的字节.
  4.  zmend : 总是 255. 指出zipmap结束. 

*有效的例子*
  18 02 06 4d 4b 44 31 47 36 01 00 32 05 59 4e 4e 58 4b 04 00 46 37 54 49 ff ..

  1.  从使用“字符串编码”开始解码。你会注意到18是字符串的长度。因此，我们将读取下24个字节，直到ff。
  2.  现在，我们开始解析从  @02 06… @ 开始的字符串，使用 “Zipmap 编码”
  3.  02是hashmap里条目的数量.
  4.  06是下一个字符串的长度. 因为长度小于254, 我们不需要读取任何额外的字节
  5.  我们读取下6个字节  4d 4b 44 31 47 36 来得到键 “MKD1G6”
  6.  01是下一个字符串的长度，这个字符串应当是值
  7.  00是空闲字节的数量
  8.  读取下一个字节 0x32，得到值“2”
  9.   在这种情况下，空闲字节是0，所以不需要跳过任何东西
  10.  05是下一个字符串的长度，在这种情况下是键。
  11.  读取下5个字节 59 4e 4e 58 4b, 得到键 “YNNXK”
  12.  04是下一个字符串的长度，这是一个值
  13.  00是值后面的空闲字节数
  14.  读取下4个字节 46 37 54 49 来得到值 “F7TI”
  15.  最终，遇到 FF, 这表示这个zipmap的结束
  16. 因此，这个zipmap表示hash {"MKD1G6" => "2", "YNNXK" => "F7TI"}
</code></pre>



## Ziplist 编码
一个Ziplist是一个序列化为一个字符串的list。本质上，list的元素按顺序地存储，借助于标记（flag）和偏移（offset）来达到高校地双向遍历list。

为解析一个ziplist，首先从流中读取一个字符串，按”字符串编码“。这个字符串是ziplist的封装。这个字符串的内容表示了ziplist。

字符串里的ziplist的结构如下：
  <zlbytes><zltail><zllen><entry><entry><zlend>

> 1.  zlbytes ：这是一个4字节无符号整数，表示ziplist的总字节数。这4字节是little endian格式－－最先出现的是最低有效位组
> 2.  zltail：这是一个4字节无符号整数，little endian格式。它表示到ziplist的尾条目（tail entry）的偏移。
> 3.  zllen：这是一个2字节无符号整数，little endian格式。它表示ziplist的条目的数量
> 4.  entry：一个条目表示ziplist的元素。细节在下面
> 5.  zlend：总是等于255.它表示ziplist的结束

ziplist的每个条目有下面的格式：
  <length-prev-entry><special-flag><raw-bytes-of-entry>

>  length-prev-enty： 这个字段存储上一个条目的长度，如果是第一个条目则是0。这允许容易地进行反向遍历list。这个长度存储为1或5个字节。
     如果第一个字节小于等于253，它被认为是长度，如果第一个字节是254，接下来4个字节用于存储长度。4字节按无符号整数读取。
>  
>  special-flag：这个标记指出条目是字符串还是整数。它也指示字符串长度或整数的大小。这个标记的可变编码如下：
>  >  1.  |00pppppp|  － 1字节：字符串值长度小于等于63字节（6bit）
>  >  2.  |01pppppp|qqqqqqqq|  － 2字节：字符串值长度小于等于16383字节（14bit）
>  >  3.  |10______|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt|  － 5字节：字符串值长度大于等于16384字节
>  >  4.  |1100____|  － 读取后面2个字节作为16bit有符号整数
>  >  5.  |1101____|  － 读取后面4个字节作为32bit有符号整数
>  >  6.  |1110____|  － 读取后面8个字节作为64bit有符号整数
>  >  7.  |11110000|  － 读取后面3个字节作为24bit有符号整数
>  >  8.  |11111110|  － 读取后面1个字节作为8bit有符号整数
>  >  9.  |1111xxxx|  － （当xxxx位于 0000 到 1101）直接4bit整数。0 到 12 的无符号整数。被编码的实际值是从 1 到
               13，因为 0000 和 1111 不能使用，所以应当从编码的4bit值里减去 1 来获得正确的值。
>  
>  Raw Bytes：在special flag后，是原始字节。字节的数字由前面的special flag部分决定。
>  
>  *举例*
23 23 00 00 00 1e 00 00 00 04 00 00 e0 ff ff ff ff ff ff ff 7f 0a d0 ff ff 00 00 06 c0 fc 3f 04 c0 3f 00 ff ... 
  |           |           |     |                             |                 |           |           |       

>  >  1.  从使用“字符串编码”开始解码。23 是字符串的长度，然后读取35个字节直到 ff
>  >  2.  使用“Ziplist 编码”解析开始于 23 00 00  ... 的字符串
>  >  3.  前4个字节 23 00 00 00 表示Ziplis长度的字节总数。注意，这是little endian 格式
>  >  4.  接下来4个字节 1e 00 00 00 表示到尾条目的偏移。 0x1e = 30，这是一个基于0的偏移。
>  >       0th position = 23, 1st position = 00 and so on. It follows that the last entry starts at 04 c0 3f 00 .. 。
>  >  5.  接下来2个字节 04 00 表示list里条目的数量。
>  >  6.  从现在开始，读取条目。
>  >  7.  00 表示前一个条目的长度。0表示这是第一个条目。
>  >  8.  e0 是特殊标记，因为它开始于位模式 1110____，读取下8个字节作为整数。这是list的第一个条目。
>  >  9.  现在开始读取第二个条目。
>  >  10.  0a 是前一个条目的长度。10 字节 ＝ 1 字节prev长度 ＋ 1 字节特殊标记长度 ＋ 8 字节整数
>  >  11.  d0 是特殊标记，因为它开始于位模式 1101____，读取下4个字节作为整数。这是list的第二个条目。
>  >  12.  现在开始第二个条目。
>  >  13.  06 是前一个条目的长度。 6 字节 ＝ 1 字节prev长度 ＋ 1 字节特殊标记 ＋ 4 字节整数。
>  >  14.  c0 是特殊标记，因为它开始于位模式 1100____，读取下2个字节作为整数。这是list的第三个条目。
>  >  15.  现在开始读取第四个条目。
>  >  16.  04 是前一个题目的长度。
>  >  17.  c0 指出是2字节整数。
>  >  18.  读取下2个字节，作为第四个条目。
>  >  19.  最终遇到 ff，这表明已经读取完list里的所有元素。
>  >  20.  因此，ziplist存储了值  [0×7fffffffffffffff, 65535, 16380, 63]。


##  Intset 编码
一个Inset是一个整数的二叉搜索树。这个二叉树在一个整数数组里实现。inset用于当set的所有元素都是整数时。Inset支持达64位的整数。
作为一个优化，如果整数能用更少的字节表示，整数数组将由16位或32位整数构建。当一个新元素插入时，intset实现在需要时将进行一次升级。

因为Intset是二叉搜索树，set里的数字总是有序的。

一个Intset有一个Set的外部接口。

为了解析Inset，首先使用“字符串编码”从流中读取一个字符串。这个字符串包含了Intset。这个字符串的内容表示了Intset。

在字符串里，Intset有一个非常简单的布局：
>  <encoding><length-of-contents><contents>
>  1.  encoding：是一个32位无符号整数。它有3个可能的值 － 2, 4 或 8.它指出内容里存储的每个整数的字节大小。
>       嗯，是的，这是浪费的－可以在2bit里存储这些信息。
>  2.  length-of-contet：是一个32位无符号整数，指出内容数组的长度。
>  3.  contents：是一个 $length-of-content 个字节的数组。它包含了二叉搜索树。

*举例*
14 04 00 00 00 03 00 00 00 fc ff 00 00 fd ff 00 00 fe ff 00 00 ...

1.  使用“字符串编码”来开始。14 是字符串的长度，读取下20个字节直到 00.
2.  现在，开始解析开始于 04 00 00 .... 的字符串。
3.  前4个字节 04 00 00 00 是编码，因为它的值是4，我们知道我们正在处理32位整数。
4.  下4个字节 03 00 00 00 是内容的长度。这样，我们知道我们正在处理3个整数，每个4字节长。
5.  从现在开始，我们以4个字节为一组读取，再把它转换为一个无符号整数。
6.  这样，我们的intset看起来是这样的 － 0x0000FFFC, 0x0000FFFD, 0x0000FFFE。注意，这些整数是little endian格式的。首先出现的是最低有效位。


## 以Ziplist 编码的 Sorted Set 
以ziplist编码存储的sorted list跟上面描述的Ziplist很像。在ziplist里，sorted set的每个元素后跟它的score。

*举例*
 [‘Manchester City’, 1, ‘Manchester United’, 2, ‘Totenham’, 3] 

如你所见score跟在每个元素后面。


## Ziplist编码的Hashmap
在这里，hashmap的键值对是作为连续的条目存储在ziplist里。

注意：这是在rdb版本4引入，它废弃了在先前版本里使用的zipmap。

*举例*
 {"us" => “washington”, “india” => "delhi"} 
存储在ziplist里是：  [“us”, “washington”, “india”, “delhi”]


### CRC32 校验和
从RDB版本5开始，一个8字节的CRC32校验和被加到文件结尾。可以通过redis.conf 文件的一个参数来作废这个校验和。

当校验和被作废时，这个字段将是0。

