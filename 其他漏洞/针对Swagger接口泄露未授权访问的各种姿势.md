## Swagger是什么
Swagger 是一个用于生成、描述和调用 RESTful 接口的 Web 服务。通俗的来讲，Swagger 就是将项目中所有（想要暴露的）接口展现在页面上，并且可以进行接口调用和测试的服务
Swagger优点：

- 将项目中所有的接口展现在页面上，这样后端程序员就不需要专门为前端使用者编写专门的接口文档；
- 当接口更新之后，只需要修改代码中的 Swagger 描述就可以实时生成新的接口文档了，从而规避了接口文档老旧不能使用的问题；
- 通过 Swagger 页面，我们可以直接进行接口调用，降低了项目开发阶段的调试成本。
Swagger注解：
swagger通过注解表明该接口会生成文档，包括接口名、请求方法、参数、返回信息的等等。
```
@Api：修饰整个类，描述Controller的作用
@ApiOperation：描述一个类的一个方法，或者说一个接口
@ApiParam：单个参数描述
@ApiModel：用对象来接收参数
@ApiProperty：用对象接收参数时，描述对象的一个字段
@ApiResponse：HTTP响应其中1个描述
@ApiResponses：HTTP响应整体描述
@ApiIgnore：使用该注解忽略这个API
@ApiError ：发生错误返回的信息
@ApiImplicitParam：一个请求参数
@ApiImplicitParams：多个请求参数
```
## 漏洞描述
`Swagger`是一个规范和完整的框架，用于生成、描述、调用和可视化 RESTful 风格的 Web 服务。总体目标是使客户端和文件系统作为服务器以同样的速度来更新。

相关的方法，参数和模型紧密集成到服务器端的代码，允许API来始终保持同步。`Swagger-UI`会根据开发人员在代码中的设置来自动生成`API说明文档`，若存在相关的配置缺陷，攻击者可以`未授权`翻查`Swagger接口文档`，得到系统功能`API接口的详细参数`，再构造参数发包，通过回显获取系统大量的敏感信息。
**Swagger 未授权访问地址存在以下默认路径：**
下面的路径就是常见的Swagger 未授权访问泄露路径，师傅们可以通过bp抓包，然后再通过bp对该接口路径进行爆破，但是我一般是先使用曾哥的一款spring-boot扫描工具去做一个自动化扫描，但是有部分网站可能对那个工具会拒绝请求，所以还是可以尝试使用bp爆破
```
/api
/api-docs
/api-docs/swagger.json
/api.html
/api/api-docs
/api/apidocs
/api/doc
/api/swagger
/api/swagger-ui
/api/swagger-ui.html
/api/swagger-ui.html/
/api/swagger-ui.json
/api/swagger.json
/api/swagger/
/api/swagger/ui
/api/swagger/ui/
/api/swaggerui
/api/swaggerui/
/api/v1/
/api/v1/api-docs
/api/v1/apidocs
/api/v1/swagger
/api/v1/swagger-ui
/api/v1/swagger-ui.html
/api/v1/swagger-ui.json
/api/v1/swagger.json
/api/v1/swagger/
/api/v2
/api/v2/api-docs
/api/v2/apidocs
/api/v2/swagger
/api/v2/swagger-ui
/api/v2/swagger-ui.html
/api/v2/swagger-ui.json
/api/v2/swagger.json
/api/v2/swagger/
/api/v3
/apidocs
/apidocs/swagger.json
/doc.html
/docs/
/druid/index.html
/graphql
/libs/swaggerui
/libs/swaggerui/
/spring-security-oauth-resource/swagger-ui.html
/spring-security-rest/api/swagger-ui.html
/sw/swagger-ui.html
/swagger
/swagger-resources
/swagger-resources/configuration/security
/swagger-resources/configuration/security/
/swagger-resources/configuration/ui
/swagger-resources/configuration/ui/
/swagger-ui
/swagger-ui.html
/swagger-ui.html#/api-memory-controller
/swagger-ui.html/
/swagger-ui.json
/swagger-ui/swagger.json
/swagger.json
/swagger.yml
/swagger/
/swagger/index.html
/swagger/static/index.html
/swagger/swagger-ui.html
/swagger/ui/
/Swagger/ui/index
/swagger/ui/index
/swagger/v1/swagger.json
/swagger/v2/swagger.json
/template/swagger-ui.html
/user/swagger-ui.html
/user/swagger-ui.html/
/v1.x/swagger-ui.html
/v1/api-docs
/v1/swagger.json
/v2/api-docs
/v3/api-docs
```
当然也可以用工具
![](image/366.png)

## Swagger漏洞猎杀
首先一般大家分享Swagger泄露接口敏感信息，一般都是在Swagger-UI这个插件里面分析
我这里以Google商店的插件为例，然后火狐和eg浏览器的话也差不多都是这个绿色的小图标
`https://chromewebstore.google.com/detail/liacakmdhalagfjlfdofigfoiocghoej?hl=zh`
![](image/367.png)
找到的一个Swagger接口泄露的一个站点
![](image/368.png)
但是这个插件可以看到`Authorize`关键字，这个你可以点击下，这个标识就是表示这个泄露的 接口需要我们输入加密的信息，要是按照正常的直接访问这个泄露的api接口，然后看敏感信息就不可行了，下面我来带大家使用一个Swagger脚本工具来给师傅们演示下
![](image/369.png)
首先我们先访问下这个泄露的public/openapi3.json文件目录
![](image/380.png)
然后可以在json文件看到里面有非常多的api接口泄露，但是太多了很多都是没有权限访问的，要是挨个拼接不太现实，那么下面我就给师傅们介绍下面下面的这款swagger工具
![](image/381.png)
#### swagger-hack工具
简介：自动化爬取并自动测试所有swagger接口
`https://github.com/jayus0821/swagger-hack`
用命令
```
python -W ignore swagger-hack2.0.py -u "http://156.239.26.40:13333/public/api-merged.json"
```
或者`-f 文件`
![](image/382.png)
扫描完成后会生成一个`.csv`文件。
![](image/383.png)
然后可以在里面找泄露的接口信息












## 参考文章
https://blog.csdn.net/xiangqian0721/article/details/127463042
https://www.freebuf.com/articles/web/423253.html