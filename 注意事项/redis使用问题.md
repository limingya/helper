## 刷新缓存
  redis 缓存一次刷新过多，导致CPU利用率高，导致宕机 
  
- 分次提交，每次提交数量做限制
- 使用临时变量保存,保存成功rename,rename介绍，见参考文档
   
闭坑操作举例： 

```java
    
     /**
     * 每次插入redis 缓存的个数大小
     */
    private final int EVERY_REDIS_CACHE_SIZE = 50;

    /**
     * 每次刷新缓存睡眠时间
     */
    private final long EVERY_REFRESH_CACHE_SLEEP_TIME = 100;
    
    private void refreshCache() {
        // 1. 移除原有缓存
        ICache codeListCache = cacheDef.getCODELIST_CACHE();
        Map<String,String> toRefreshCacheMap = new HashMap<>();

        //  正式改为1000
        int limit = 1000;
        int startId = 0;
		
        List<XxxDo> xxxDoList =  xxxDao.findXxx(startId,limit);
        // 2. 获取模板关键词
        while (xxxDoList.size() > 0){
            for (TemplateKeywordDo loopKeywordDo : xxxDoList) {
                startId = loopKeywordDo.getId();
				// TODO业务处理,封装toRefreshCacheMap
				
                if(toRefreshCacheMap.size() == EVERY_REDIS_CACHE_SIZE){
                    // 设置模板关键词 短名称 对应关系
                    codeListCache.hmset(RedisKeyConst.SEO.XXX_TEMP,toRefreshCacheMap);
                    try {
                        TimeUnit.MILLISECONDS.sleep(EVERY_REFRESH_CACHE_SLEEP_TIME);
                        toRefreshCacheMap.clear();
                    } catch (InterruptedException e) {
                        log.error("异常：{}",e);
                    }
                }
            }
           
            xxxDoList = xxxDao.findXxx(startId,limit);
        }
        if(toRefreshCacheMap.size() > 0){
            codeListCache.hmset(RedisKeyConst.SEO.XXX_TEMP,toRefreshCacheMap);
        }    
        codeListCache.rename(RedisKeyConst.SEO.XXX_TEMP,RedisKeyConst.SEO.XXX);
    }
```  

- 40万数据刷入redis用时记录

| EVERY_REDIS_CACHE_SIZE| 开始时间| 结束时间| 总用时 | rename时间 | 
|---|---|---|---|---|
|10  | 2019-08-07 10:38:01 | 2019-08-07 11:50:11 | 72分钟 | 0秒   |
|50  | 2019-08-07 12:49:30 | 2019-08-07 13:04:13 | 15分钟 | 0秒   |
|100 | 2019-08-07 13:45:30 | 2019-08-07 13:53:49 | 8分钟  | 1秒   |


# 参考文档

[redis-rename介绍](https://redis.io/commands/rename)