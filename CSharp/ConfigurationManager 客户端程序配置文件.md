# ConfigurationManager 客户端程序配置文件

1. 可以自己添加配置文件

 ![img](https://img-blog.csdnimg.cn/20190529161758924.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zODIxMTE5OA==,size_16,color_FFFFFF,t_70) 

2. 添加key和value

``````c#
<configuration>
    <appSettings>
      <add key = "connectionString" value = "server =.;database = School;uid = sa;pwd = 123"/>
    </appSettings>
</configuration>
``````

3. 添加引用

 ![img](https://img-blog.csdnimg.cn/20190529162116301.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zODIxMTE5OA==,size_16,color_FFFFFF,t_70) 

4.  添加命名空间 

`using System.Configuration`

5. 使用

`string connectionString = ConfigurationManager.AppSettings["connectionString"];`

