---
layout: post
title:  "批量升级 NugetPackage 版本"
date:   2024-01-11
author: D.Q
modifyTime: 2024-04-01 12:00:00
isDisplay: true
excerpt: 有时候我们需要升级整个解决方案中的某些Nuget版本，如果每个手动使用[NuGet Package Manager]会很麻烦。经过一个周末的踩坑，我找到一个解决方案。

---




# 批量升级项目中的 Nuget

有时候我们需要升级整个解决方案中的某些Nuget版本，如果每个手动使用[NuGet Package Manager](https://learn.microsoft.com/zh-cn/nuget/consume-packages/install-use-packages-visual-studio)
会很麻烦。经过一个周末的踩坑，我找到一个解决方案。

| Name            | OldVersion | NewVersion |
|-----------------|------------|------------|
| Newtonsoft.Json | 13.0.1     | 13.0.3     |
| Polly           | 8.0.0      | 8.2.1      |


> 以下所有命令都需要在 **Visual Studio Package Manager Console (程序包管理控制台)** 中执行 
> 具体路径为：**Tools(工具) > NuGet Package Manager(NuGet包管理) > Package Manager Console(程序包管理控制台)**

## Get-Package

先使用[Get-Package](https://learn.microsoft.com/zh-cn/nuget/reference/ps-reference/ps-ref-get-package) 命令看一下现有安装包的版本
```shell
PM> Get-Package -ProjectName ClassLibrary1

Id                                  Versions                                 ProjectName                                                                                                                                                                                                       
--                                  --------                                 -----------                                                                                                                                                                                                       
Newtonsoft.Json                     {13.0.1}                                 ClassLibrary1                                                                                                                                                                                                     
Polly                               {8.0.0}                                  ClassLibrary1                                                                                                                                                                                                    
PM> 
```

当前项目安装的是版本 `13.0.1` 和 `8.0.0`

## Update-Package

[Update-Package](https://learn.microsoft.com/zh-cn/nuget/reference/ps-reference/ps-ref-update-package)命令可以升级指定包

```shell
PM> Update-Package -ProjectName ClassLibrary1 -Id Newtonsoft.Json -Version 13.0.3
正在还原 D:\Source\Repos\ClassLibrary1\ClassLibrary1.csproj 的包...
正在安装 NuGet 程序包 Newtonsoft.Json 13.0.3。
将资产文件写入磁盘。路径: D:\Source\Repos\ClassLibrary1\obj\project.assets.json
已还原 D:\Source\Repos\ClassLibrary1\ClassLibrary1.csproj (用时 5 毫秒)。
已从 ClassLibrary1 成功卸载“Newtonsoft.Json 13.0.1”
已将“Newtonsoft.Json 13.0.3”成功安装到 ClassLibrary1
执行 nuget 操作花费时间 101 毫秒
已用时间: 00:00:00.1531212
PM> 
```

## 批量安装

为此需要写一段 [PowerShell](https://learn.microsoft.com/zh-cn/nuget/reference/powershell-reference) 脚本，先获取整个项目的指定包信息，每个进行判断。如果符合条件则更新。

```shell
# 定义要升级的包 key:packageName value:targetVersion
# 定义要升级的包 key:packageName value:targetVersion
$packages = @{
    "Newtonsoft.Json" = "13.0.3";
    "Polly" = "8.2.1";
 };
  
 foreach ($packageName in $packages.Keys) { # 遍历要升级的包
    Write-Host "--------------------$($packageName) BEGIN------------------------";

     $targetVersion = $packages[$packageName]; # 获取要升级的版本
     # 获取项目包中已经安装的包信息 see https://learn.microsoft.com/en-us/nuget/reference/ps-reference/ps-ref-get-package
     $projectPackages = Get-Package -Filter $packageName; 
     
     foreach ($projectItem in $projectPackages) { # 处理每一个项目
        Write-Host "--------------------$($projectItem.ProjectName) BEGIN------------------------";

        if ($projectItem.Version -lt $targetVersion) { # 如果项目安装版本小于目标版本
            Write-Host "Project: $($projectItem.ProjectName) ↑ $($packageName)";
            Write-Host "Version: $($projectItem.Version) < 目标版本：$targetVersion";
            # 执行升级 see https://learn.microsoft.com/en-us/nuget/reference/ps-reference/ps-ref-update-package
            Update-Package -ProjectName $projectItem.ProjectName $packageName -Version $targetVersion;
        }

        Write-Host "--------------------$($projectItem.ProjectName) END------------------------";
    }

    Write-Host "--------------------$($packageName) END------------------------";
 }
```

### 输出日志

两个包都是先卸载，然后安装了指定版本。

>执行的时候脚本代码没有换行而是一整行，在 `PowerShell` 管道中不支持 `Win` 换行。所以需要把代码压缩成一行执行。

```shell
PM> $packages = @{"Newtonsoft.Json" = "13.0.3"; "Polly" = "8.2.1"}; foreach ($packageName in $packages.Keys) { Write-Host "--------------------$($packageName) BEGIN------------------------"; $targetVersion = $packages[$packageName]; $projectPackages = Get-Package -Filter $packageName; foreach ($projectItem in $projectPackages) { Write-Host "--------------------$($projectItem.ProjectName) BEGIN------------------------"; if ($projectItem.Version -lt $targetVersion) { Write-Host "Project: $($projectItem.ProjectName) ↑ $($packageName)"; Write-Host "Version: $($projectItem.Version) < 目标版本：$targetVersion"; Update-Package -ProjectName $projectItem.ProjectName $packageName -Version $targetVersion } Write-Host "--------------------$($projectItem.ProjectName) END------------------------" } Write-Host "--------------------$($packageName) END------------------------" }
--------------------Polly BEGIN------------------------
--------------------ClassLibrary1 BEGIN------------------------
Project: ClassLibrary1 ↑ Polly
Version: 8.0.0 < 目标版本：8.2.1
正在还原 D:\Source\Repos\ClassLibrary1\ClassLibrary1.csproj 的包...
正在安装 NuGet 程序包 Polly 8.2.1。
将资产文件写入磁盘。路径: D:\Source\Repos\ClassLibrary1\obj\project.assets.json
已还原 D:\Source\Repos\ClassLibrary1\ClassLibrary1.csproj (用时 5 毫秒)。
已从 ClassLibrary1 成功卸载“Polly 8.0.0”
已从 ClassLibrary1 成功卸载“Polly.Core 8.0.0”
已将“Polly 8.2.1”成功安装到 ClassLibrary1
已将“Polly.Core 8.2.1”成功安装到 ClassLibrary1
执行 nuget 操作花费时间 117 毫秒
已用时间: 00:00:00.3168864
--------------------ClassLibrary1 END------------------------
--------------------Polly END------------------------
--------------------Newtonsoft.Json BEGIN------------------------
--------------------ClassLibrary1 BEGIN------------------------
Project: ClassLibrary1 ↑ Newtonsoft.Json
Version: 13.0.1 < 目标版本：13.0.3
正在还原 D:\Source\Repos\ClassLibrary1\ClassLibrary1.csproj 的包...
正在安装 NuGet 程序包 Newtonsoft.Json 13.0.3。
将资产文件写入磁盘。路径: D:\Source\Repos\ClassLibrary1\obj\project.assets.json
已还原 D:\Source\Repos\ClassLibrary1\ClassLibrary1.csproj (用时 6 毫秒)。
已从 ClassLibrary1 成功卸载“Newtonsoft.Json 13.0.1”
已将“Newtonsoft.Json 13.0.3”成功安装到 ClassLibrary1
执行 nuget 操作花费时间 81 毫秒
已用时间: 00:00:00.1507802
--------------------ClassLibrary1 END------------------------
--------------------Newtonsoft.Json END------------------------
PM> 
```

再次看一下现有安装包的版本，包都被安装为指定版本了。

```shell
PM> Get-Package -ProjectName ClassLibrary1

Id                                  Versions                                 ProjectName                                                                                                                                                                                                       
--                                  --------                                 -----------                                                                                                                                                                                                       
Newtonsoft.Json                     {13.0.3}                                 ClassLibrary1                                                                                                                                                                                                     
Polly                               {8.2.1}                                  ClassLibrary1                                                                                                                                                           

PM> 
```

## 参考

* [what-is-nuget](https://learn.microsoft.com/zh-cn/nuget/what-is-nuget)
* [powershell-reference](https://learn.microsoft.com/zh-cn/nuget/reference/powershell-reference)
* [Install and use a NuGet package in Visual Studio](https://learn.microsoft.com/zh-cn/nuget/quickstart/install-and-use-a-package-in-visual-studio)