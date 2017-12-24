---
title: pig笔记
date: 2017-03-21 19:59:06
tags: 
- hadoop 
- pig
categories:
- hadoop
- pig
---

`pig`描述的是一个有向无环图(DAG).
节点代表处理数据的操作符，节点间的向量代表数据流。
> 数据流语言，所以无if和for等控制流语言的元素
> (有foreach)
> (有别的方法加入控制流)
> 无关系数据库的事务和索引。

Pig的大小写敏感没有一致的规定。操作和命令是不区分大小写的，比如load，group。而函数是区分大小写的，比如MAX。关系名(类似于变量)也是大小写敏感的。(如`A=load 'table1';a=LOAD 'table2'`)。



pig语句的完全执行是在遇到输出语句的时候(如`dump`和`store`)。
之前即使试图装载一个不存在的文件也不会报错。
在输出语句前，可以重赋值变量，以最后一次赋值为准。

-  run和exec
>这两个命令都可以执行Pig脚本。区别是exec以批处理方式运行，这意味着所有该脚本文件中定义的标示符，当脚本执行完后在脚本调用窗口将不可访问。而以run运行时，相当于将脚本文件中的内容手工在调用窗口输入并执行，这就意味着脚本文件中所有的标示符，比如定义的关系在脚本执行完后仍然可访问，同时在shell的命令历史记录中可以找到脚本中的语句。

- 运行时参数设定
```pig
-- 运行.pig脚本
pig -p date=${DATE} -p store_path=${STORE_PATH} getLiveInfoFromLog.pig
-- -p参数指定键值对
-- -f参数指定运行的脚本，可省略。

-- 指定参数文件:
pig -m daily.params daily.pig

# -p指定的参数设置值 会覆盖 参数文件中的设置。
# -r或-dryrun参数可以查看参数替换后的pig脚本，而不会真正执行这个脚本。
-- 脚本内部定义参数:
%declare paralle_factor 10;
-- 通常的默认值，可通过传入参数覆盖
%default paralle_factor 10;

使用参数:
yesterday =fileter daily by data == '$DATE';
```

- set设置:
`pig`脚本中设置是全局可见的，后面的设置会覆盖前面的值，即使在最后面进行的设置也会对前面的执行语句生效。

```pig
-- 引入其他脚本
set pig.import.search.path 'usr/local/pig,/grid/pig';
import 'acme/macros.pig';
```

- 宏命令
```
-- 简单地内联，不可递归调用
-- 给定每天的输入和指定的年，分析在支持的股息的那些天里股票价格是如何变化的
define dividend_analysis(daily,year,daily_symbol,daily_open,daily_close) 
return analyzed
{
divs=load 'NYSE_dividends' as 
(exchange:chararray,
symbol:chararray,
date:chararray,
dividends:float
);
divsthisyear=filter divs by date matches '$year-.*';
dailythisyear=filter $daily by date matches '$year-.*';
jnd=join divsthisyear by symbol,dailythisyear by $daily_symbol;
$analyzed=foreach jnd generate dailythisyear::$daily_symbol,$daily_close-$daily_open;
};
```



```pig
/*多行注释*/
-- 单行注释
ls / ; --列出目录
fs -l ; --hdfs命令
rmr filename ; --递归删除
pig命令后可以看到版本:
/*Apache Pig version 0.14.0 (r1640057) compiled Nov 16 2014, 18:02:05*/
-- 结束任务:
kill jobid;
```

- 先进行分组后进行连接：
```pig
-- 加载交易表，(select customer,purchase from transactions)
txns = load 'transactions' as (customer,purchase);
-- 分组
grouped= group txns by customer;
-- 聚合函数sum, total表含group和tp两个字段
total = foreach grouped generate group, sum (txns.purchase)as tp;
-- 加载customer_profile表
profile= load 'customer_profile' as (customer,zipcode);
-- 表连接
answer= join total by group, profile by customer;
-- 输出结果
dump answer;
```

`filter`是什么操作?
好像是限制、过滤操作，类似`sql`中的`where`。
```pig
startswithcm= filter divs by symbol matches 'CM.*';
nostartswith= filter divs by no symbol matches 'CM.*';
-- 布尔操作符的优先级是: not、and、or
```

`x==null`返回`null`，filter只允许判断条件返回true的记录通过，所以`2,null,4`中，对于条件`x==2`,只有2通过。(不报错)
`null`既不会匹配也不会失败，因为`null`表示未知。

`explain`语法?

---

`group`是关键字吗? 是的。
`for each grouped generate group`是固定用法，因为:
`group`关键字有重载，两个含义(两个场景):
1. `grpd= group daily by stock;`中,作为`group by`语句中关键字元素；
2. `group by`语句的输出中,第一列字段名也为`group`, 所以在紧接着的语句中作为字段名:
```pig
cnt= foreach grpd generate group, COUNT(daily);
-- 可以通过describe语句查看grpd的具体模式（结构）:
describe grpd;
-- 对多个字段进行分组:(多个字段会组合成tuple放入'group'列中)
daily = load 'NYSE_daily' as (exchange,stock,date,dividends);
grpd= group daily by (exchange,stock);
avg= foreach grpd generate group, AVG(daily.dividends);
-- 也可以对所有字段进行分组 'all'为关键字
grpd= group daily all;
cnt = foreach grpd generate COUNT(daily);
-- 此时group的输出字段为all而不是group.
```

- 查找访问次数最多的前5个URL
```pig
Users = load 'users' as (name,age);--as声明具体模式,列名
-- 多的列舍弃不要，缺的列填充为null (null表示未知)
Fltrd = filter Users by age >=18 and age<=25;
Pages = load 'pages' as (user,url);
-- 连接获取每个用户访问过的URL
Jnd = join Fltrd by name, Pages by user;
Grpd = group Jnd by url;
-- 每个url，被年龄[18,25]的用户访问了多少次
Smmd = foreach Grpd generate group , COUNT(Jnd) as clicks;
Srtd = order Smmd by clicks desc;
Top5 = limit Srtd 5;
Store Top5 into 'top5sites';
```

- 加载文件:
```pig
LOG = LOAD '/user/hive/warehouse/temp.db/tmp_solar_user_stat/dt=2015-10-01/*' USING PigStorage('\t') AS (userid:int, timestamp:chararray, log_time:chararray, url:chararray, method:chararray, sc:chararray, fplatform:chararray, vendor:chararray, version:chararray, create_date:chararray, total_query:int, failed_query:int, total_favorate:int, today_favorate:int);
```

- 特殊数据类型(pig为强类型语言)
1. `chararray`: `java.lang.String`.单引号使用：'fred'.
2. `map`: ['name'#'bob','age'#55] 键值间使用'#'号,键值对间使用逗号','。  `map`的`key`一定为`chararray`类型,`value`可以为任意类型而且可以不同。(也可以设定为都一样。)
3. `tuple`: ('bob',55)
4. `bag`: {('bob',55),('john',25)} 无序的`tuple`集合。

- 声明模式:
```pig
-- 限定类型或不限定统一类型:
as (aaa:map[],bbb:map[int])
as (aaa:tuple(),bbb:tuple(x:int,y:int))
as (aaa:bag{},bbb:bag{t:(x:int,y:int)})
```

- 未声明时根据下标(从0开始)引用,类型推测:
```pig
daily= load 'NYSE_daily'
calcs =foreach daily generate $7/1000;
```
猜测不出时，会推断为`chararray`.
```pig
-- 使用范围表示位置(闭区间)
prices = load 'NYSE_daily' as (exchange,symbol,date,open,high,low,close,volume,adj_close);
beginning= foreach prices generate ..open;
-- [exchange,open]
middle= foreach prices generate open..close;
end = foreach prices generate volume..;
-- 也可以使用'*'表示所有字段。
```

- 三元条件操作符
判断条件为`null`时整个表达式返回`null`.
如`null==2 ? 1:4`返回`null`.
此外冒号两边应该是同一数据类型。

- 访问`map`、`tuple`和`bag`:
```pig
-- map:
avg = foreach bball generate bat#'batting_average';
-- tuple:
B = foreach A generate t.x, t.$1;
-- bag:
A = load 'input' as (b:bag {t:(x:int,y:int)});
B = foreach A generate b.x;
```

`P39`为什么`A.y`是`bag`?

- `order by`
```pig
bydate= order daily by date desc,symbol;
-- desc 仅对紧靠着的date生效.
```
`null`一般是最小的.（升序时排最前.）
复合类型不可排序。
`pig`的`order`语句在`shuffle`阶段分发任务时,同一个键对应的任务可能分发到不同的`reducer`上.

- `distinct`
```pig
uniq= distinct daily;
```

<table>
<tr>
<td>参与作用阶段
</td>
<td>map
</td>
<td>shuffle
</td>
<td>reduce
</td>
</tr>
<tr>
<td>
</td>
<td>?
</td>
<td>order by
  <br>group by
  <br>limit
  <br>
</td>
<td>
 order by
 <br>group* by 
 <br>聚合函数
 <br>limit
 <br>distinct
 <br>join*
 <br>cogroup*
 <br>cross
</td>
</table>
影响`reduce`的操作符可以使用`parallel`以进行并行化优化:
```pig
bysymbol= group daily by symbol parallel 10;
-- 使任务具有10个reducer.
set default_parallel 10;
-- 使整个脚本范围内的reducer数量为10.
```



- `Join`连接后通过`::`访问同名列：
```pig
daily::exchange : bytearray
```
- 外连接
左外连接要求右边表模式已知。以此类推。
```pig
-- 左外连接 `left outer`
jnd = join daily by(simbol,date) left,
    divs by (simbol,date);
-- 右外连接 `right outer`
-- 全外连接 `full outer`.
-- 设定连接具体算法:
jnd = join daily by (exchange,symbol),
    divs by (exchange,symbol)
    using 'replicated'
;
--内连接和左外连接可以使用分片-复制算法,从小到大排列表，最左边的表读入内存。
-- 数据倾斜时，可对两个表使用skewed join(抽样):
jnd = join cinfo by city, users by city using 'skewed';

-- 按连接键排好序的表: 
using 'merge';
```




- 抽样(`sample`)
```pig
divs = load 'NYSE_dividends';
some= sample divs 0.1;
```

- 注册`python`的`UDF`
```pig
register 'production.py' using jython as bballudfs;
-- bballudfs表示命名空间
-- jython表示编译器(脚本是python便携)
-- 调用时使用bballudfs.production(参数)
```

- 宏定义`define`
```pig
define convert com.acme.financial.Convert('dollar','euro');
-- 可调用java函数:（有性能代价）
define hex InvokeForString('java.lang.Integer.toHexString','int');
-- 参数列表以空格作为分隔符 如'int int[] long'
```

- `flatten`
`foreach`中的`flatten`修饰符，`bag`中每条记录会和`generate`中其他所有表达式进行交叉乘积。
```pig
players= load 'baseball' as  
(
name : chararray,
position: bag {t: (p:chararray)},
bat: map[]
);
pos=foreach players generate name,flatten(position)as position;
bypos= group pos by position;
```
如记录为:
`Jorge,{(Catcher),(Designated_hitter)}`
则生成两条:
`Jorge,Catcher`
`Jorge,Designated_hitter`
`flatten`对于`tuple`不会进行交叉乘积，只会把`tuple`展开为顶层字段。

- 内嵌`foreach`语句
(支持`distinct`,`filter`,`limit`,`order`)
```pig
-- 每笔交易对应的不同股票交易号的个数。
daily= load 'NYSE_daily' as (exchange,symbol);
grpd = group daily by exchange;
uniqcnt = foreach grpd {
    sym = daily.symbol;
    uniq_sym= distinct sym;
    generate group, COUNT(uniq_sym);
};
--查找每只股票支付的前三个股息值:
divs= load 'NYSE_dividends' as (exchange,symbol,date,dividends);
grpd = group divs by symbol;
tops= foreach grpd {
    sorted = order divs by dividends desc;
    top =limit sorted 3;
    generate group, flatten(top);
};
```

---

复习一下semi-join?

- `union`
```pig
A= load 'input1' as (x:int,y:float);
B= load 'input2' as (x:int,y:double);
C= union A , B;
describe C;
/* * 类型自动转换
C:{x:int , y: double}
*/
-- union onschema A,B;
-- 强制新增列:
A = load 'input1' as (w:chararray,x:int,y:float);
B = load 'input2' as (x:int,y:double,z:chararray);
C = union onschema A,B;
describe C;
/*
C: {w:chararray,x:int,y:double,z:chararray}
*/

```

- `cross`?
先`cross`后`filter`以实现非等值`join`。
(`pig`不直接支持非等值`join`)
```pig
crossed = cross table1,table2;
output= filter crossed by table1::date< table2::date;
```

- `cogroup`:
```pig
A= load 'input1' as (id:int,val1:float);
B= load 'input2' as (id:int,val2:int);
C= cogroup A by id, B by id;
describe C;
/**注意到第一个字段名依然是group。
C:{
group:int,
 A:{id:int,val1:float}},
 B:{id:int,val2:int}
} 
*/
```


- `stream`:
```pig
-- 把py脚本加载到集群,
-- getLiveInfoFromLog表示加载后的别名，
-- `getLiveInfoFromLog.py`表示加载的脚本，
-- SHIP ('getLiveInfoFromLog.py');表示依赖的文件。
-- 可以依赖多个文件,
-- 例如可以SHIP ('getLiveInfoFromLog.py','rule.conf')。

define getLiveInfoFromLog `getLiveInfoFromLog.py` SHIP ('getLiveInfoFromLog.py');

-- 使用stream xxx through语句，调用脚本生成新表。
new_table = STREAM table1 THROUGH getLiveInfoFromLog AS (date:chararray,roomId:chararray,userId:chararray);
# 加载目录必须是工作目录的相对路径，不能是绝对路径。
```

- 分布式缓存
```pig
crawl = load 'webcrawl as (url,pageid)';
normalized =foreach crawl generate normalized (url);
define blc 'blacklistchecker.py' cache('/data/share/badurls#badurls');
goodurls=stream normalized through blc as (url,pageid);
-- '#'号前是hdfs上的路径，后面是读取的文件名。
```

- 非标准输入输出的脚本
```pig
define blc 'blacklistchecker.py -i urls -o good' input('urls') output('good');
-- input和output中指定的都是工作目录的相对路径下文件。
```

- mapreduce任务
```pig
goodurls=mapreaduce 'blacklistchecker.jar'
store table1 into 'input'
load 'output' as (url,pageid)
'com.acmeweb.security.BlackLiskChecker -i input -o output';
```

- `split`
```pig
split wlogs into 
apr03 if timestamp < '20110404',
apr02 if timestamp < '20110403' and timestamp>'20110401',
apr01 if timestamp < '20110402' and timestamp>'20110331'
;

store apr03 into '20110403';
store apr02 into '20110402';
store apr01 into '20110401';
-- 可由fliter by等效改写
```






















- 数据仓库:
> ODS（Operational Data Store，操作型存储）、EDW（Enterprise Data Warehouse，企业数据仓库）、DM（DataMart，数据集市）区别开来，共分三层。


如何查看环境变量如`$date`?

- 正则:
`{ab,cd}`匹配字符串集合中任一字符串。



- 完整示例脚本
```pig
set mapred.output.compress false;
set pig.exec.reducers.bytes.per.reducer 500000000;
set mapreduce.reduce.memory.mb 10240;
DEFINE dwSolarUserStat `dwSolarUserStat.py` SHIP('dwSolarUserStat.py');
DEFINE solarFrog `solarFrog.py` SHIP('solarFrog.py','rule.conf');
DEFINE solarSpam `solarSpam.py` SHIP('solarSpam.py');
LOG = LOAD '/user/hive/warehouse/temp.db/tmp_solar_user_stat/dt=$date/*' USING PigStorage('\t') AS (userid:int, timestamp:chararray, log_time:chararray, url:chararray, method:chararray, sc:chararray, fplatform:chararray, vendor:chararray, version:chararray, create_date:chararray, total_query:int, failed_query:int, total_favorate:int, today_favorate:int);

LOG_FROG = LOAD '/user/hive/warehouse/temp.db/tmp_solar_frog/dt=$date/*' USING PigStorage('\t') AS (userid:chararray, url:chararray, duration:chararray, imagesize:chararray, net:chararray, timestamp:chararray, log_dt:chararray, competitors:chararray);

LOG_FROG_ORDER = ORDER LOG_FROG BY userid,timestamp;


SESSION_TASKINFO_GRP = GROUP LOG BY userid;
SESSION_TASKINFO_GRP_SORTED = FOREACH SESSION_TASKINFO_GRP {
ORDERED = ORDER LOG BY log_time;
GENERATE FLATTEN(ORDERED);
}


SESSION_TASKINFO_GRP_FROG = GROUP LOG_FROG_ORDER BY userid;
SESSION_TASKINFO_GRP_SORTED_FROG = FOREACH SESSION_TASKINFO_GRP_FROG {
ORDERED_FROG = ORDER LOG_FROG_ORDER BY timestamp;
GENERATE FLATTEN(ORDERED_FROG);
}


DATA = STREAM SESSION_TASKINFO_GRP_SORTED THROUGH dwSolarUserStat AS (userid:int,create_date:chararray,total_query:int,failed_query:int,total_favorate:int,today_favorate:int,session_num:int,session_time:int,vendor:chararray,platform:chararray,version:chararray);

SPAM_FROG = STREAM SESSION_TASKINFO_GRP_SORTED_FROG THROUGH solarSpam AS (userid:int);

DATA_FROG = STREAM SESSION_TASKINFO_GRP_SORTED_FROG THROUGH solarFrog AS (userid:int,favorite_action:int,load_image_action:int,load_image_lasttime:int,load_image_failed_action:int,search_lasttime:int,load_image_imagesize:int,net_action:chararray,net_lasttime:chararray,mark_compeitorst:chararray);


STORE DATA INTO '$store_path/$date/stat';
STORE SPAM_FROG INTO '$store_path/$date/spam';
STORE DATA_FROG INTO '$store_path/$date/frog';
```

究竟谁是zhengguanyu?


---

##调试：

- describe
```pig
describe grpd;
/*
输出:
grpd:{group:chararray,trimmed:{(symbol:chararray,dividends:float)}}
*/
```

- explain
```pig
pig -x local -e 'explain -script explain.pig
-- -x local 指定运行模式为本地模式
-- -e excute后的命令
```

-- pigUnit测试脚本