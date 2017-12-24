---
title: kylin笔记
date: 2017-03-15 19:54:15
tags: kylin
categories: 
- hadoop
---

星型模型:
>有冗余,快 所有维度表直接连接到事实表;(间接,eg. 地域id->(国家,省份))
维度表之间没有关联.

雪花模型:
>无冗余,慢 不是所有维度表直接连接到事实表.(间接,eg. 地域id->(国家id,省份id)->国家维度表,省份维度表)
维度表之间存在关联.

kylin只支持星型模型,所以不能太复杂.

聚合组:
>通过聚合组分隔维度,使cube数量降低;

Mandatory:(必选列)
>将总是出现在where或group by 里的列设置为Mandatory,减少预计算开销;
A为必选:
(AB,C,AC)=>(AB,AC)

Hierarchy:(WITH ROLLUP)
>将国家\省\市,设置为层级关系Hierarchy,减少预计算开销.

Derived:(推断):
>LOOKUP表(维度表),若DimA的值唯一确定DimB的值,则DimB能从DimA获得(Derived),则标记DimB为Derived可以将group by 简化.
(ABC,AB,AC => AC,A)


job状态:
>PENDING
RUNNING
ERROR
DISCARD
FINISH

api:
>get xxx/kylin/api/jobs/{job_uuid}/steps/{step_id}/output

>PUT xxx/kylin/api/jobs/{job_uuid}/resume
(ERROR可以resume,DISCARD无法resume)

>PUT xxx/kylin/api/jobs/Cubes/{cube_name}/rebuild
cubename
startTime
endTime
buildType: BUILD,MERGE,REFRESH

>POST xxx/kylin/api/query
sql:"select 1"
offset:0
limit:2
acceptPartial:false
project:"learn_kylin"


可以用视图解决字段数据类型的约束.