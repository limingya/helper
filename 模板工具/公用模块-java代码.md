# 接口请求规范

- `模板只是帮助，需要根据实际业务变化使用，切勿强硬套用`

## admin 
```java
	@Autowired
    private LocalProperties localProperties;	
	
	 String url = localProperties.getApiV2Domain() + ".do";
		Map<String, Object> signMap = new HashMap<>();
		JSONObject jsonObject = HttpUtil.getBodyStringForApiV2(url, SignUtil.requestMapBasicSign(signMap),3000);

		if(jsonObject != null){
			String code = jsonObject.getString("code");
			if (HttpStatus.SC_OK == Integer.valueOf(code)) {
				JSONObject data = jsonObject.getJSONObject("data");
				return mv.addObject("msg",data);
			}else{
				String msg = jsonObject.getString("msg");
				mv.addObject("code",Integer.parseInt(code));
				mv.addObject("msg",msg);
			}
		}

		mv.addObject("code",-2);
		return null;
	
	
```


## api-v2		 
	// 签名 不为空，说明是后端请求，验证签名正确性
	if(!SignUtil.checkRequest(request)){
		return ServerResponse.createByError(-101, "签名错误");
	}
	
# 大数据量处理（缓存/定时任务）

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
- 注：通过存oss 的缓存，用新文件去覆盖旧文件的更新，一定要考虑内容为空时，旧文件是否删除，若不删除会出现则前端同步出错 


## 定时统计任务【天、月时间段数据统计】
```java
	public void work(){
        // 1.1 今日日期
        LocalDateTime endDateTime = LocalDateTime.now();
        endDateTime = endDateTime.withHour(0);
        endDateTime = endDateTime.withMinute(0);
        endDateTime = endDateTime.withSecond(0);
        endDateTime = endDateTime.withNano(0);
        endDateTime = endDateTime.withDayOfMonth(1);

        // 1.2 从redis 获取这次开始月份，默认上个月
        ICache codeListCache = cacheDef.getCODELIST_CACHE();
		// TODO 更改RedisKey
        String startDateStr = codeListCache.get(RedisKeyConst.Print.PRINT_ORDER_STATISTICS_START_MONTH);

        LocalDateTime startDateTime = null;
        if(StringUtil.isNotEmpty(startDateStr)){
            startDateTime = LocalDateTime.parse(startDateStr + "-01 00:00:00", DateTimeFormatter.ofPattern(DateUtilEYK.DateFormatEnum.u.getValue()));
        }else{
            // 开始日期 默认设置为上个月的
            startDateTime = endDateTime.plusMonths(-1);
        }

        LocalDateTime loopDateTime = startDateTime;
        // 开始月份 = 结束月份  或者 开始月份 < 结束月份 【循环进行】
        while (loopDateTime.isBefore(endDateTime)){
            // TODO 此处写业务统计代码
            statisticsRepeatBuyRate(loopDateTime);

            // 获取当前循环月份+1
            loopDateTime = loopDateTime.plusMonths(1);
        }
		
		// TODO 更改RedisKey
        codeListCache.set(RedisKeyConst.Print.PRINT_ORDER_STATISTICS_START_MONTH,endDateTime.format(DateTimeFormatter.ofPattern(DateUtilEYK.DateFormatEnum.yyyyMM.getValue())));

    }
```

## 分批量获取数据【统计】（以1000分隔）

```java
     // TODO  更改Dao相应的方法，此方法名称，及业务处理代码
	 private Set<Integer> getPaySuccessUserIds(LocalDateTime startDateTime, LocalDateTime endDateTime) {
        // 1. 获取这个月第一个订单
        TreeSet<Integer> userIdSet = new TreeSet<>();
        PrintOrderStatisticDto orderDto = printOrderDao.getFirstData(startDateTime);
        if(orderDto == null){
            return userIdSet;
        }
        int beginId = orderDto.getId();
        boolean isContinue = true;

        // 正式limit 改成 1000
        int limit = 1000;
	
        List<PrintOrderStatisticDto> orderDtos =  printOrderDao.getPrintOrderStatisticsDto(beginId,0,limit);
        while (orderDtos.size() > 0  && isContinue){
            // 2. 遍历数据，统计出订单支付成功、退款状态=未退款的用户ID
            for (PrintOrderStatisticDto loopOrderDto : orderDtos) {
                 LocalDateTime loopDateTime = loopOrderDto.getCreateTime();
                 if(loopDateTime.isEqual(endDateTime) || loopDateTime.isAfter(endDateTime)){
                    isContinue = false;
                    break;
                 }
                // TODO 此处写业务代码
            }
            orderDtos =  printOrderDao.getPrintOrderStatisticsDto(beginId ,1,limit);
        }
        return userIdSet;

    }
```

## 分批量获取数据【刷新缓存】（以1000分隔）

```java
        //  正式改为1000
        int limit = 1000;
        int startId = 0;
		// TODO 获取数据Dao修改
        List<TemplateSearchInfoTupleDo> tupleDos =  templateSearchInfoTupleDao.findKeyWord(startId,0,limit);
        
        // TODO 封装集合定义
        
        while (tupleDos.size() > 0){
            for (TemplateSearchInfoTupleDo loopTupleDo : tupleDos) {
                startId = loopTupleDo.getId();
            }
			// TODO 获取数据Dao修改
            tupleDos = templateSearchInfoTupleDao.findKeyWord(startId,0,limit);
        }
```


# model类模板


## 枚举类创建模板

```java 
	NO_START(0,"未开始"),

    ING(1,"进行中"),

    OVER(2,"已结束");
    private int value;
    private String description;

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public int getValue() {
        return value;
    }

    public void setValue(int value) {
        this.value = value;
    }

    PrintTimeLimitActivityStateEnum(int value, String description) {
        this.value = value;
        this.description = description;
    }

    /**
     * @desc 根据value获取枚举
     * @author liuph
     * @date  2019/06/11 10:25
     * @return 对应的枚举值optional
     */
    public static Optional<PrintTimeLimitActivityStateEnum> getInstance(int value) {
        return Stream.of(PrintTimeLimitActivityStateEnum.values())
                .filter(e -> e.value == value)
                .findFirst();
    }
```
- 注：替换相应的枚举、名称 （PrintTimeLimitActivityStateEnum）


## 批量更新（一次1000）(循环模板)
```java
	int pageSize = 1000;
	int count = cacheBos.size();
	// 计算总页数
	int totalPage = count % pageSize == 0 ? count / pageSize : count / pageSize + 1;
	
	// pageSize 6个数据 分一页
	for (int i = 0; i < totalPage; i++) {
		int page = i + 1;
		int startIndex = i * pageSize;

		List<OfficialVideoBo> onePageData = cacheBos.stream()
				.skip(startIndex).limit(pageSize)
				.collect(Collectors.toList());
	}
	
```
- 注：此处改为在收集时操作，做法如下

![](doc/media/177c602e.png)


