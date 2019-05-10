1、insert ignore 插入时间判断主键或者索引是否存在，存在跳过不执行，避免插入重复的数据

```sql
    INSERT IGNORE INTO org_user_active_record ( user_id, active_day, active_time, ip_address, browser_user_agent )
VALUES
	( 702393, '2019-05-10', '2019-05-10 20:08:10', 2886731694, 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.10 Safari/537.36' );
```

2、批量修改

    格式：
         update 表名 set  name = case id 
             when 1 then 张三
             when 2 then 李四
             when 3 then 王五
         end  where id in(1,2,3)
    说明：当id为1是将name改为张三，当id为2时将name改为李四........
 
 ```sql
    <update id="batchUpdateShowOrder">
        UPDATE template_theme_client_relation SET show_order = CASE id
        <foreach collection="list" item="item" separator=" ">
            WHEN ${item.id} THEN ${item.showOrder}
        </foreach>
        END WHERE id IN
        <foreach collection="list" separator="," item="item" open="(" close=")">
            ${item.id}
        </foreach>
     </update>
  ```   
     
3、插入或者更新：插入时判断唯一键或者索引是否存在,如果存在则修改update_time值，如果不存在则插入 

```sql
 INSERT INTO wx_mini_program_message_push_record (user_id, source) VALUES (#{userId},#{source}) ON DUPLICATE KEY UPDATE update_time = NOW(),user_id = #{userId},source=#{source}
 ```
 
 - 注意：小心入坑：自增主键不连续
 
 4、REPLACE INTO replace into 首先尝试插入数据到表中， 1. 如果发现表中已经有此行数据（根据主键或者唯一索引判断）则先删除此行数据，然后插入新的数据。 2. 否则，直接插入新数据。

```sql 
    REPLACE INTO org_user_active_record ( user_id, active_day, active_time, ip_address, browser_user_agent )
    VALUES
      ( 702393, '2019-05-10', '2019-05-11 20:08:10', 2886731694, 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.10 Safari/537.36' );
 ```     
 
