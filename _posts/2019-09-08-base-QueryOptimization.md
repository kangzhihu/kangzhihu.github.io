---
layout: post
title: "技巧-代码查询优化"
subtitle: '批量查询小技巧'
author: "Kang"
date: 2019-09-08 18:40:47
header-img: "img/post-head-img/post-bg-first.jpg"
catalog: false
tags:
  - 技巧
  - 数据库
---

## 技巧
### 业务需要单条获取
&emsp;&emsp;一般来说，在没有前台页面的情况下，会将数据全部查询出来，这样就可能导致单次查询的数据量过大，查询耗时，所以需要对查询进行优化，可以使用引入分页和本地缓存的方式来优化整个查询，具体示例代码：    
[ExampleRecordDBTableReader](https://github.com/kangzhihu/docs/blob/master/exampleCode/DB/ExampleRecordDBTableReader.java)   
&emsp;&emsp;特别需要注意的是，在使用的时候，ExampleRecordDBTableReader需要为有状态的，所以必须为<b>非单利模式<b/>


### 大数据捞取

思路：将最大值与最小值之间的数据量划分为固定的块，每次将该块中符合条件数据捞取出来。
1. loadContext全局变量，在for循环中保存查询全局变化参数。外部根据DataLoadResult中的 hasMore去控制是否需要捞数据
2. 捞取sql示例
```xml
select from TableName
     where transaction_date=#transactionDate#
     and id >= #idStart# and id < #idEnd#
     order by id desc
     limit #pageStart#, #pageSize#
```
``` java
    protected DataLoadResult<T> queryDataById(DataLoadContext loadContext) {
        LoggerUtil.info(logger, "按minId, maxId方式捞取数据, taskKey={0},taskBizId={1}",
            loadContext.getTaskKey(), loadContext.getTask().getBizId());

        initContextForId(loadContext);
        // 分页查询逻辑
        int pageSize = loadContext.getPageSize();
        long minId = Long.valueOf(loadContext.getBizData(TaskParamEnum.MIN_ID.getName()));
        long pageCount = Long
            .valueOf(loadContext.getBizData(TaskParamEnum.PAGE_COUNT.getName()));
        long count = Long.valueOf(loadContext.getBizData(TaskParamEnum.ID_COUNTER.getName()));
        if (count < pageCount) {
            // 总页数进行分页，由于通过id进行主键索引进行500行捞去。
            List<R> exportRecordList = loadExportRecordsById(loadContext.tableNo(),
                loadContext.getBizDate(), 0, pageSize, count * pageSize + minId, count * pageSize
                                                                                 + minId + pageSize);
            updateContextForId(loadContext);
            List<T> dataMapList = assembleResult(exportRecordList);
            return new DataLoadResult(true,dataMapList);
        }
        return new DataLoadResult(false,null);
    }

    /**
     * 初始化根据id 捞取上下文
     * @param loadContext
     */
    protected void initContextForId(DataLoadContext loadContext) {
        //ID_COUNTER标识当前第几页
        if (loadContext.getBizData(TaskParamEnum.ID_COUNTER.getName()) == null) {
            loadContext.putBizData(TaskParamEnum.ID_COUNTER.getName(), "0");
            // 分页查询逻辑
            long maxId = getMaxId(loadContext.tableNo(), loadContext.getBizDate());
            long minId = getMinId(loadContext.tableNo(), loadContext.getBizDate());
            long totalCount = maxId - minId + 1;
            int pageSize = loadContext.getPageSize();
            // 只有一页的场景
            long pageCount = (totalCount + pageSize - 1) / pageSize;
            loadContext.putBizData(TaskParamEnum.MIN_ID.getName(), String.valueOf(minId));
            loadContext.putBizData(TaskParamEnum.PAGE_COUNT.getName(),
                String.valueOf(pageCount));
        }
    }
    
    protected void updateContextForId(DataLoadContext loadContext) {
        loadContext.putBizData(TaskParamEnum.ID_COUNTER.getName(), String.valueOf(Long
            .valueOf(loadContext.getBizData(TaskParamEnum.ID_COUNTER.getName()) + 1L)));
    }
```
