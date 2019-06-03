# 接口请求规范

## admin 
```java
	@Autowired
    private LocalProperties localProperties;
	

	String url = localProperties.getApiV2Domain() + "/keyword/refreshMapCache.do";
	Map<String, Object> signMap = new HashMap<>();
	com.alibaba.fastjson.JSONObject jsonObject = HttpUtil.getBodyStringForApiV2(url,SignUtil.requestMapBasicSign(signMap),3000);

	if(jsonObject != null){
	 String code = jsonObject.getString("code");
	 if (HttpStatus.SC_OK == Integer.valueOf(code)) {
		 return Boolean.TRUE;
	 }else {
		 return Boolean.FALSE;
	 }
	}else{
	return Boolean.FALSE;
	}
```


## api-v2		 
	// 签名 不为空，说明是后端请求，验证签名正确性
	if(!SignUtil.checkRequest(request)){
		return ServerResponse.createByError(-101, "签名错误");
	}
	
# 缓存建立规范

## 需要oss 和 redis 双重缓存

```java 
public PageResult<TemplateSearchInfoTupleDo> getKeyWordMap(String mapKind,Integer pageNo) {
        // 获取需要获取的分页数据的 文件名称
        String redisKey = null;
        String ossFileKey = null;

        // 获取OSS 中文件的最新更新时间
        if("-1".equals(mapKind)){
            ossFileKey =  pageNo + ".json";
        }else{
            ossFileKey =  mapKind +  "_" + pageNo + ".json";
        }

        ObjectMetadata objMetaData = publicFileStorage.getObjectMetadata(StorageBtype.TEMPLATE_KEYWORD_MAP.getAttachFinalKey(ossFileKey));
        PageResult<TemplateSearchInfoTupleDo> pageResult = new PageResult<>();
        pageResult.setList(new ArrayList<>());
        if(objMetaData == null){
            return pageResult;
        }

        Date lastModified = objMetaData.getLastModified();
        if("-1".equals(mapKind)){
            redisKey = RedisKeyConst.System.TEMPLATE_KEYWORD_MAP + "_"+ pageNo + "_" + lastModified.getTime();
        }else{
            redisKey = RedisKeyConst.System.TEMPLATE_KEYWORD_MAP + "_"+ mapKind + "_" + pageNo + "_" + lastModified.getTime();
        }

        //  先从redis 获取数据
        ICache templateCache = cacheDef.getSYS_CACHE();
        String pageResultStr  = templateCache.get(redisKey);


        if(StringUtils.isNotEmpty(pageResultStr)){
             pageResult = JSONObject.parseObject(pageResultStr,PageResult.class);
        }else{
            // redis 无数据，从oss中获取
            OSSObject file = publicFileStorage.getFile(StorageBtype.TEMPLATE_KEYWORD_MAP.getAttachFinalKey(ossFileKey));
            try(InputStream pageResultIs = file.getObjectContent();){
                if(pageResultIs == null){
                    pageResult = new PageResult<>();
                }else {
                    pageResultStr = StreamUtil.inputStreamToString(pageResultIs);
                    pageResult = JSONObject.parseObject(pageResultStr,PageResult.class);
                }

            }catch (Exception e){
                logger.error("获取模板关键词分页信息失败,类型：{},页码：{}",mapKind,pageNo);
            }
        }

        // 存入redis 中，有的话，就延时 30分钟
        templateCache.set(redisKey,pageResultStr, Expire.m30);
        return pageResult;
    }
```

## 分批量获取数据（以1000分隔）

```java
        //  正式开启
        LocalDate nowLocalDate = LocalDate.now();
        //  模拟数据 测试专用  正式删除
        LocalDate startLocalDate = nowLocalDate.plusDays(-7);
        LocalDate endLocalDate = nowLocalDate.plusDays(-1);
        LocalDate loopDate = startLocalDate;

        // TODO 收集集合 
        HashMap<Integer, MutableInt> oneTemplateIdAndUsageCount = new HashMap<>();

        // 近7天处理数据
        while (loopDate.isBefore(endLocalDate) || loopDate.isEqual(endLocalDate)) {
            // TODO 替换dao
            String suffix = DbTableShardingHelper.calcNameSuffix(loopDate.getYear(), loopDate.getMonth());
            TemplateUsageDataDo firstData = templateUsageDataShardingDao.getFirstData(suffix, loopDate);
            if (Objects.nonNull(firstData)) {
                int beginId = firstData.getId();
                boolean singleDayContinue = true;

                // TODO 替换dao 从DB 中查询 >= 此ID 的 1000 条数据
                List<TemplateUsageDataDo> list = templateUsageDataShardingDao.getUserPointBillDos(suffix, beginId, 0, 1000);
                for (; CollectionUtils.isNotEmpty(list) && singleDayContinue; ) {
                    if (CollectionUtils.isNotEmpty(list)) {
                        for (TemplateUsageDataDo loopDo : list) {
                            // 比较当前数据循环的日期，看是否存在大于循环日期的
                            Date currentDate = loopDo.getCreateTime();
                            Instant instant = currentDate.toInstant();
                            ZoneId zoneId = ZoneId.systemDefault();
                            LocalDate currentLocalDate = instant.atZone(zoneId).toLocalDate();
                            if (currentLocalDate.isAfter(loopDate)) {
                                singleDayContinue = false;
                                break;
                            }
                            
                            // TODO 统计 逻辑写在此处
                            
                            
                            // 更新开始时间
                            if (loopDo.getId() > beginId) {
                                beginId = loopDo.getId() + 1;
                            }
                        }
                        // 替换Dao
                        list = templateUsageDataShardingDao.getUserPointBillDos(suffix, beginId, 0, 1000);
                    }
                }
            }
            loopDate = loopDate.plusDays(1);
        }

```


# api service 编写

	ServerResponse getVideoKind(Integer forTest); 
	尽量使用对象形式，少用ServerResponse
	示例：List<VideoKindDo> getVideoKind(Integer forTest); 



