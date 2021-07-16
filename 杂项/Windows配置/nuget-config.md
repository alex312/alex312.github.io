1. 随.net core或者visual studio 2015及以上版本安装
2. 设置全局包下载位置，打开文件%appdata%\NuGet\NuGet.Config，其中添加

```xml
<configuration>
    ...
    <config>
        ...
        <add key="globalPackagesFolder" value="F:\nuget\packages" />
        ...
    </config>
    ...
</configuration>
```
3. 设置http-cache位置，设置系统环境变量NUGET_HTTP_CACHE_PATH=F:\nuget\http-cche
4. 设置plugins-cache位置, 设置系统环境变量NUGET_PLUGINS_CACHE_PATH=F:\nuget\plugins-cache

5. 如果安装了Visual Studio，打开文件%ProgramFiles(x86)%\NuGet\Config\Microsoft.VisualStudio.Offline.config， 按照一下内容修改

```xml
<configuration>
  ...
  <packageSources>
    ...
    <!-- <add key="Microsoft Visual Studio Offline Packages" value="C:\Program Files (x86)\Microsoft SDKs\NuGetPackages"/> -->
    <add key="Microsoft Visual Studio Offline Packages" value="F:\nuget\msvs"/>
    ...
  </packageSources>
  ...
</configuration>

```