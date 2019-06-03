# 页面大块

```html 
<#include "/common/header.ftl"> <#include "/common/space.ftl">
<link rel="stylesheet" href="/libs/layui-v2.4.5/css/layui.css" media="all">
<script src="/libs/tagsinput/jquery.min.js"></script>






<script src="/libs/layui-v2.4.5/layui.js" charset="utf-8"></script>
<script src="/libs/layui-v2.3.0/layer/layer.js" charset="utf-8"></script>
<script src="/libs/jquery-ui/jquery-ui.js"></script>
<script src="/mod/seo/page_admin.js"></script>


<#include "/common/footer.ftl">

```

# 表格

## 页面 
```html 

<!-- div  表格 -->
<div>
    <table class="layui-hide" id="" lay-filter=""></table>
</div>



<#------------------------------------------- 表格titleBar------------------------------------------>
<script type="text/html" id="tb_title_bar">
    <div class="layui-btn-container">
        <button class="layui-btn layui-btn-sm" id="add" >
            添加
        </button>
        <button class="layui-btn layui-btn-sm" id="clear_cache" >
            清空缓存
        </button>
        <button class="layui-btn layui-btn-sm" id="search" >搜索</button>
    </div>
</script>
<#------------------------------------------   表格 操作 栏   --------------------------------------->
<script type="text/html" id="tb_operate_bar">
    <a class="layui-btn layui-btn-xs"  lay-event="image_manager">image标签管理</a>
    <a class="layui-btn layui-btn-danger layui-btn-xs" lay-event="edit"> 修改</a>
</script>

```
## js
```javascript
<!-- ---------------------- 表格加载--------------- -->
$(function () {

    /**
     * 数据表单赋值
     */
    layui.use('table', function () {
        var table = layui.table;

        // 表单加载
        table.render({
            elem: '#tb_page',
            url: '/seo/pagesettingnew/pageAdminList.do',
            toolbar: '#tb_title_bar',
            title: '用户数据表',
            where:{
                search_page_path:''
            },
            loading: true,
            parseData: function (res) { //res 即为原始返回的数据
                return {
                    success: true,
                    data: res.data.list, //解析数据列表
                    count: res.data.count
                };
            },
            request: {
                pageName: 'pageNo' //页码的参数名称，默认：page
                , limitName: 'pageSize' //每页数据量的参数名，默认：limit
            },
            response: {
                statusName: 'success' //规定数据状态的字段名称，默认：code
                , statusCode: true //规定成功的状态码，默认：0
            },
            cols: [[
                {type: 'checkbox', fixed: 'left'},
                {field: 'pagePath', title: '页面路径', width: 160, fixed: 'left',  sort: true},
                {field: 'title', title: '标题'},
                {field: 'keywords', title: '关键词' },
                {
                    field: 'description',
                    title: '描述',
                    width: 120,
                },
                { title: '创建时间',templet: '<div>{{ format(d.createTime)}}</div>'},
                {  fixed: 'right',
                    width:300,
                    title: '操作',
                    templet: '#tb_operate_bar'
                },
            ]],
            page: true
        });



        // 头工具栏事件
        table.on('toolbar(tb_page)', function(obj){
            //var checkStatus = table.checkStatus(obj.config.id);
            switch(obj.event){
                case 'add':
                    layer.msg('add');
                    break;
                case 'clearCache':
                    layer.msg('clearCache');
                    break;
                case 'search':
                    layer.msg('search');
                    break;
            };
        });

        // 监听行工具事件
        table.on('tool(tb_page)', function (obj) {
            var data = obj.data;
            switch (obj.event) {
                case 'imageManager':
                    console.log("imageManager");
                    break;
                case 'edit':
                    console.log("edit");
                    break;
            }
        });

    });

});
```


# 弹窗

## 页面
```html 

<!----------------------------------------------------------- 弹窗 --------------------------------- -->
<div id="pop_edit_or_video" hidden="hidden" style="display: none;">
    <div class="layui-form layui-row">
        <div class="layui-form-item">
            <a hidden="hidden" id="pop_edit_unique_id"></a>
            <div class="layui-form-item">
                <img src="/images/default_avatar.png" alt=""
                     style="height: 100px ;margin-left: 111px;margin-bottom: 10px;" id="pop_edit_show_thumb" >
                <div>
                    <label class="layui-form-label"/>缩略图</label>
                    <input type="file" name="thumbFile"  id="pop_edit_thumb_file" onchange="thumbFileChange();"
                           autocomplete="off" class="layui-input" style="width: 200px;height: 38px;" >
                </div>
            </div>
            <div class="layui-form-item">
                <label class="layui-form-label"/>视频文件</label>
                <div>
                    <input type="file" name="metaFile" placeholder="请输入密码" id="pop_edit_meta_file" autocomplete="off"
                           class="layui-input" style="width: 200px ;height: 38px;">
                </div>
            </div>
			
			  <div class="layui-form-item">
                <div>
                    <label class="layui-form-label"/>标题</label>
                    <input type="text" name="title" placeholder="请输入标题" id="pop_edit_title" autocomplete="off"
                           class="layui-input" style="width: 200px;height: 38px;">
                </div>
            </div>	
			
        </div>
    </div>

    <div class="layui-input-block">
        <button class="layui-btn" id="pop_edit_btn_add_video">立即提交</button>
    </div>

</div>



```

## js
```javascript
<!--  ------------------  弹窗 ------------------------- -->

 popEditOrVideo = layer.open({
                title: '修改视频',
                type: 1,
                area: ['400px', '500px'],
                content: $("#pop_edit_or_video"),
                scrollbar: false,
            });

 //弹窗中单选框、复选框不显示问题解决
 
// 重新渲染layui控件
layui.use('form', function () {
	var form = layui.form;
	form.render();
});		
```



# 其他常用js
## js 日期格式化

```javascript
<!-- --------------------  日期格式化 ------------------- -->
function format(datetime) {
    var date = "";
    layui.use('util', function () {
        var util = layui.util;
        date = util.toDateString(datetime.time);
    });
    return date;
}
```


## a标签点击事件

```javascript
<!--  ------------------  a 标签点击事件 ------------------------- -->	
 <a hidden id="download_csv_template"></a>
  $("#download_csv_template").attr("href",url);
  document.getElementById("download_csv_template").click();
  
```
  
# freemark 常用标签

## 状态值

```html
<td>
	<%
		var state = row.state;
		var stateDesc;
		if (state == -1){
			stateDesc = "被关闭";
		}else if(state == 0) {
			stateDesc = (new Date().getTime() - row.create_time.time)/1000/60 < 1440 ? "待支付" : "超时未支付";
		} 
	%>
	<%=stateDesc%>
</td>
<td>${(record.processingState == 0) ? string('等待处理' , (record.processingState == 1) ? string("处理成功" , "处理失败"))}</td>
  
```

  

