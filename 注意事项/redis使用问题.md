## 刷新缓存
  redis 缓存一次刷新过多，导致CPU利用率高，导致宕机 
  
- 分次提交，每次提交数量做限制
- 使用临时变量保存,保存成功rename,rename介绍，见参考文档
   
闭坑操作举例： 

```java
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



# 参考文档

[redis-rename介绍](https://redis.io/commands/rename)