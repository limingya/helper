# mybatis 公用模板抽离


## 1.7 批量更新	(1.0版本)

```sql
    UPDATE
		table_name
    SET view_times = 
        CASE design_template_ad_id
            <foreach collection="list" item="item" > 
                WHEN #{item.id} THEN #{}					
            </foreach>
        END
    WHERE
    design_template_ad_id IN
    <foreach collection="list" item="item" separator="," close=")" open="(" > 
        #{item.id} 
    </foreach>

   （2.0版本）
    UPDATE 
		print_center_page_item
    SET show_order =
    <foreach collection="list" item="item" index="index"
             separator=" " open="case ID" close="end">
        when #{item.id} then #{}
    </foreach>
    where id in
    <foreach collection="list" index="index" item="item"
             separator="," open="(" close=")">
        #{item.id}
    </foreach>
```
	
## 1.8 mapper 文件公用Do,sql抽离

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.eyuanku.web.chuangkit.api.framework.M.dao.VideoKindDao">

    <resultMap id="videoKindDoResult" type="com.eyuanku.web.chuangkit.api.framework.domain.dox.VideoKindDo" autoMapping="true"/>

    <sql id="videoKindDoSql">
        id,create_time,main_title,sub_title,sort,for_test
    </sql>
    
    <select id="getVideoKind" resultMap="videoKindDoResult">
        SELECT 
        <include refid="videoKindDoSql"/>
        FROM video_kind
        where for_test = #{forTest}
        ORDER BY sort ASC
    </select>

</mapper>
```