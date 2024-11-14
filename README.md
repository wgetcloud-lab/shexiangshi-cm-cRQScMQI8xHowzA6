
# 问题描述


在创建App Service服务的时候，根据定价层不同，内存使用的最大值也有不同。但在实际测试中，发现内存最大只能占用2GB左右，


而定价层中内存分配明明是大于2GB(比如B3定价层的内存为7GB), 这是一种什么情况呢？


![](https://img2024.cnblogs.com/blog/2127802/202411/2127802-20241113202845624-1544123643.png)


在App Service中Kudu工具上，查看进程分配的内存大小：


![](https://img2024.cnblogs.com/blog/2127802/202411/2127802-20241113203005789-851616001.png)


 


# 问题解答


使用一段简短C\#代码来进行验证，代码是一个循环，根据URL中输入的数字，循环创建多个100MB大小的对象.




```
var builder = WebApplication.CreateBuilder(args);
// Add services to the container.
var app = builder.Build();

app.MapGet("/oom/{objs}", (string objs) =>
{
    int objunmbers = int.Parse(objs);
    try
    {
        byte[][] largeArray = new byte[objunmbers][];
        for (int i = 0; i < objunmbers; i++)
        {
            largeArray[i] = new byte[1024 * 1024 * 100]; // 每个对象占用100MB
        }
        return "成功创建大数组对象: " + objs;
    }
    catch (OutOfMemoryException)
    {
        return "内存不足，无法创建大数组对象";
    }
});

app.Run();
```


代码本地调试，可以看见应用占用内存直线上涨：


 ![](https://img2024.cnblogs.com/blog/2127802/202411/2127802-20241113210454021-209298404.gif)


部署到Azure App Service后，同样通过Kudu站点Process信息，发现内存的占用的确只有2GB左右，但请求需要更多内存资源时，页面返回：***内存不足，无法创建大数组对象***


这是因为App Service for Windows默认使用32位操作系统，最大内存只能分配2GB。


当主动修改位64位后，App Service 内存就可以占用到定价层所允许的最大值！


![](https://img2024.cnblogs.com/blog/2127802/202411/2127802-20241113211245412-84199397.png)


如B3的定价层在Kudu中查看到w3wp.exe进程分配的内存达到了6GB


![](https://img2024.cnblogs.com/blog/2127802/202411/2127802-20241113211517526-430202653.png)


 


**根据以上测验，当使用App Service内存没有达到预期的值，且应用异常日志出现OutOfMemory时，就需要检查Platform的设置是否位64bit。**



# 参考资料


**I see the message "Worker Process requested recycle due to 'Percent Memory' limit." How do I address this issue?**


[**https://learn.microsoft.com/en\-us/troubleshoot/azure/app\-service/web\-apps\-performance\-faqs\#i\-see\-the\-message\-worker\-process\-requested\-recycle\-due\-to\-percent\-memory\-limit\-how\-do\-i\-address\-this\-issue**](https://github.com)



> The maximum available amount of memory for a 32\-bit process (even on a 64\-bit operating system) is 2 GB. By default, the worker process is set to 32\-bit in App Service (for compatibility with legacy web applications).
> 
> 
> Consider switching to 64\-bit processes so you can take advantage of the additional memory available in your Web Worker role. This triggers a web app restart, so schedule accordingly.
> 
> 
> Also note that a 64\-bit environment requires a Basic or Standard service plan. Free and Shared plans always run in a 32\-bit environment.


 


 本博客参考[westworld加速](https://tianchuang88.com)。转载请注明出处！
