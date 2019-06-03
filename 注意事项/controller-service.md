# 命名
- 命名原则：见名知意

## 接口URL命名设置

接口Action 中已经有了父路径，方法URL中可以去除父路径内容

示例： 

```java
@Controller
@RequestMapping("/template/user/favorite/")
public class TemplateUserFavoriteAction{

    @RequestMapping("collect.do")
    @ResponseBody
    public Object collectTemplate(){}
```

## service 接口参数

- controller 方法中进行参数基本校验 
- service 参数尽量采用基本数据类型，避免其他地方调用出现`空指针`

# 注释

- 方法保证每一个参数都有必要的解释

# 权限

- 涉及用户内容的操作 
例如：用户的模板、文件夹、素材 的删除、修改、移动
 
 一定要校验用户是否有此权限，杜绝出现一个用户可以操作其他用户资源的情况
