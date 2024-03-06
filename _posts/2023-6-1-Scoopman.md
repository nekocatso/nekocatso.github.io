---
layout: tools
title: "你需要掌握的Scoop技巧和知识"
date:   2023-6-1
tags: [tools]
comments: true
author: nekocatso
---
本文列举了Scoop各种使用技巧和相关知识。包含：

* Scoop 的设计与实现理念；
* 自定义Scoop安装路径；
* Scoop潜在错误排查；
* 如何在Scoop中进行断点续传；
* 更新或禁用软件更新；
* Scoop中的别名；
* 如何在同一程序的不同版本之间切换，比如切换不同版本的JDK；
* 等等各种实用技巧

> 本文首发于我的个人博客：

　　Scoop 包管理工具介绍

---

　　Windows 下常用的包管理工具有：

* Scoop
* Chocolatey

　　相比于Chocolatey，Scoop则更专注于开源的命令行工具，使用 Scoop 安装的应用程序通常称为"便携式"应用程序，需要的权限更少，对系统产生的副作用也更少，所以我这里选择了使用 Scoop。

　　⚠️️ 注意：对于像 VirtualBox、Docker for Windows ，输入法等这些需要高权限的软件还是通过在官网下载安装包进行安装。

　　Scoop 安装与配置

---

　　**要求：**

* PowerShell >= 5.0 (如果是 Window10 则默认满足此条件)
* 请确保已允许PowerShell执行本地脚本，可以使用下面的命令开启：

```text
set-executionpolicy remotesigned -scope currentuser

# 或者 （但是它没有上面的命令安全）
set-executionpolicy Unrestricted -scope currentuser
```

　　安装路径：

* 用户级别安装的程序 和 Scoop 本身默认安装于 `C:\Users\<user>\scoop`​
* 全局安装的程序（所有用户可用，使用`--global`​或 `-g`​ 选项）位
  于`C\ProgramData\scoop`​路径中。

　　可以通过更改对应的环境变量更改这些路径 。

　　**将 Scoop 安装到自定义目录** :

　　打开 PowerShell 先配置环境变量 `SCOOP`​，再运行 `iex`​

```text
$env:SCOOP='D:\Scoop'
# 先添加用户级别的环境变量 SCOOP
[environment]::setEnvironmentVariable('SCOOP',$env:SCOOP,'User')

安装 Scoop 和 scoop-cn（推荐）

此方法会把安装 Scoop 过程中的地址都换成中国可快速访问的地址，并设置好 Scoop，添加本仓库。打开 PowerShell，输入以下命令下载安装 Scoop：

irm https://mirror.ghproxy.com/https://raw.githubusercontent.com/duzyn/scoop-cn/master/install.ps1 | iex

或使用 jsDelivr 的地址：

irm https://cdn.jsdelivr.net/gh/duzyn/scoop-cn/install.ps1 | iex

安装成功后，会提示“scoop and scoop-cn was installed successfully!”
```

　　**配置全局安装路径** （可选，建议不改）

```text
$env:SCOOP_GLOBAL='D:\GlobalScoopApps'

[environment]::setEnvironmentVariable('SCOOP_GLOBAL',$env:SCOOP_GLOBAL,'Machine')
```

> 相当于在系统变量中设置： `SCOOP_GLOBAL=D:\GlobalScoopApps`​；默认是在
> `C:\ProgramData\scoop`​。

　　**为什么需要全局安装？**

　　对于那些需要管理员权限的程序需要进行全局安装。我当前遇到的是当使用 Scoop 安装字体时需要使用全局安装，因为字体需要给所有用户使用。

　　初次安装 Scoop 后，建议安装的程序：

```text
# 但 scoop 进行全局安装时需要使用到 sudo 命令
scoop install sudo

# scoop下载程序时支持使用 aria2 来加速下载
scoop install aria2
```

　　我们可以发现，下载的过程中自动下载了依赖 7-zip。 在安装方面，它利用了 7zip 去解
压安装包/压缩包，因此它对绿色软件有天生的友好属性 。不仅如此，下载之后的内容会自
动将加入到（Path）环境变量中，十分方便。

{{< alert theme="info" >}}
　　**补充：**   初次安装之后我们可以通过运行 `scoop checkup`​ 来检测当前潜在问题，然后根据提示进行修正。

```text
# 检测本人当前环境存在的问题
$ scoop checkup

WARN  Windows Defender may slow down or disrupt installs with realtime scanning.
  Consider running:
    sudo Add-MpPreference -ExclusionPath 'D:\Scoop\Applications'
  (Requires 'sudo' command. Run 'scoop install sudo' if you don't have it.)
WARN  Windows Defender may slow down or disrupt installs with realtime scanning.
  Consider running:
    sudo Add-MpPreference -ExclusionPath 'C:\ProgramData\scoop'
  (Requires 'sudo' command. Run 'scoop install sudo' if you don't have it.)
WARN  LongPaths support is not enabled.
You can enable it with running:
    Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' -Name 'LongPathsEnabled' -Value 1
ERROR 'dark' is not installed! It's required for unpacking installers created with the WiX Toolset. Please run 'scoop install dark' or 'scoop install wixtoolset'.
WARN  Found 4 potential problems.

```

　　可以看到存在三个警告（WARN），一个错误（ERROR），并给出了解决对应问题的命令：

* 前两个警告（WARN）提示：杀毒软件 Windows Defender 有可能会使得下载变慢或阻止安装
* 第三个警告（WARN）提示：Windows中的 NTFS 中默认不允许大于 260 个字符（byte）的文件全路径存在的限制还未解除。（可能需要添加`sudo`​才能运行给出的命令）
* 最后一个错误提示（ERROR）：需要安装 `dark`​ 才能解压使用 WiX Toolset 创建的安装包。

{{< /alert >}}
　　**Scoop 的设计与实现理念** ：

* 分离用户数据：默认将程序的 **用户数据** 存储到 `persist`​ 目录中，这样当用户日
  后升级该程序后之前的用户配置依然可用。（但是对于部分程序支持的不是很完善）
* ​`shim`​软链接： scoop 会自动在 scoop 应用安装路径下的 `shims`​ 文件夹下为新安装
  的程序添加对应的 `.exe`​ 文件，而 `shims`​ 文件夹提前就已被添加到 `PATH`​ 环境变
  量中，所以程序一旦安装就可以直接在命令行中运行。
* **对于 GUI 程序** ，scoop 还会自动为其在开始菜单中添加快捷方式 ，路径：
  `C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Scoop Apps`​

　　Scoop 常用命令

---

```text
scoop help #查看帮助
scoop help <某个命令> # 具体查看某个命令的帮助

scoop install <app>   # 安装 APP
scoop uinstall <app>  # 卸载 APP

scoop list  # 列出已安装的 APP
scoop search # 搜索 APP
scoop status # 检查哪些软件有更新

scoop update # 更新 Scoop 自身
scoop update appName1 appName2 # 更新某些app
scoop update *  # 更新所有 app （前提是需要在apps目录下操作）

scoop bucket known #通过此命令列出已知所有 bucket（软件源）
scoop bucket add bucketName #添加某个 bucket

scoop cache rm <app> # 移除某个app的缓存
```

　　安装卸载软件

---

```text
# 安装之前，通过 search 搜索 APP, 确定软件名称
scoop search  xxx

# 安装 APP
scoop install AppName

# 安装特定版本的 APP；语法 AppName@[version]，示例
scoop install git@2.23.0.windows.1

# 卸载 APP 
scoop uninstall #卸载 APP
```

　　更新软件

---

　　包含：如何禁用更新

```text
scoop update # 更新 Scoop 自身

scoop update appName1 appName2 # 更新某些app

# 更新所有 app （可能需要在apps目录下操作）
scoop update *

# 禁止某程序更新
scoop hold <app>
# 允许某程序更新
scoop unhold <app>
```

　　清除缓存与旧版本

---

```text
# 查看所有以下载的缓存信息
scoop cache show

# 清除指定程序的下载缓存
scoop cache rm <app>

# 清除所有缓存
scoop cache rm *

# 删除某软件的旧版本
scoop cleanup <app>

# 删除全局安装的某软件的旧版本
scoop cleanup <app> -g

# 删除过期的下载缓存
scoop cleanup <app> -k
```

|别名|
| ------|

　　⚠️️ 注意：请在 Powershell 中运行下面的命令

```text
# 可用操作
scoop alias add|list|rm [<args>]

## 添加别名，格式：
scoop alias add <name> <command> <description>

# 示例：（注意：必须在 Powershell中运行）
scoop alias add st 'scoop status' '检查更新'
# 检查已添加的别名
scoop alias list -v

Name Command      Summary
---- -------      -------
st   scoop status 检查更新

# 测试已添加的别名 st
scoop st


# 另一个示例：
scoop alias add rm 'scoop uninstall $args[0]' '卸载某 app'
```

　　在同一程序的不同版本之间切换

---

　　使用命令：

```text
scoop reset [app]@[version]
```

　　示例：

```text
scoop reset idea-ultimate-eap@201.6668.13

scoop reset idea-ultimate-eap@201.6073.9

# 切换到最新版本
scoop reset idea-ultimate-eap
```

　　对应版本的程序需要已经安装于本地系统中；所以在你清除某个软件的旧版本时考虑一下自己是否还会再次使用到此旧版本。

　　另外 idea-ultimate-eap 切换过程可能需要更长时间。

　　其它命令

---

```text
# 显示某个app的信息
scoop info <app>

# 在浏览器中打开某app的主页
scoop home <app>

# 比如
scoop home git
```

　　添加软件源 Bucket

---

　　Scoop 可安装的软件信息存储在 Bucket（翻译为：桶）中，也可以称其为软件源。Scoop 默认的 Bucket 为 `main`​ ；官方维护的另一个 Bucket 为 `extras`​，我们需要手动添加。

```text
# bucket的用法
scoop bucket add|list|known|rm [<args>]
```

　　添加 extras :

```text
scoop bucket add extras
```

　　我们也可以添加第三方 bucket ，示例：

```text
scoop bucket add dorado https://github.com/h404bi/dorado
```

　　并且明确指定安装此 bucket （软件源）中的的程序：

```text
scoop install dorado/<app_name>
# 下面是dorado中特有的软件，测试其是否添加成功
scoop search trash
```

　　推荐的 Bucket（软件源）：

```text
# 先添加bucket
scoop bucket add spc https://mirror.ghproxy.com/https://github.com/lzwme/scoop-proxy-cn
scoop bucket add scoop-cn https://mirror.ghproxy.com/https://github.com/duzyn/scoop-cn
```

---

　　一个示例：

```text
scoop install mediainfo
```

　　当安装 mediainfo 时由于网络问题，安装包无法下载，从命令行输出信息中可以看到如下
内容

```text
ERROR Download failed! (Error 1) An unknown error occurred
ERROR https://mediaarea.net/download/binary/mediainfo/19.09/MediaInfo_CLI_19.09_Windows_x64.zip
    referer=https://mediaarea.net/download/binary/mediainfo/19.09/
    dir=D:\Scoop\Applications\cache
    out=mediainfo#19.09#https_mediaarea.net_download_binary_mediainfo_19.09_MediaInfo_CLI_19.09_Windows_x64.zip

ERROR & 'D:\Scoop\Applications\apps\aria2\current\aria2c.exe' --input-file='D:\Scoop\Applications\cache\mediainfo.txt'
```

　　我们可以发现文件的下载路径和下载后的文件名称，这里 `out=`​ 后面的压缩包就是下载后
文件的名称，(也可以在 scoop 的 `cache`​ 目录下的 `mediainfo.txt`​ 文件中找到下载路径与文
件名称)

　　然后我们可以尝试在浏览器或其他下载程序中（可以 fq 的程序中）下载该程序，下载完成
后再更改文件名并将其放入 scoop 的 `cache`​​ 目录，最后再次运行
`scoop install mediainfo`​​ 即可安装。

　　‍

　　这里给出几个可以通过配置提升 Scoop 使用体验的建议:

1. 使用 7zip 作为 Scoop 的默认解压工具

```
scoop config 7ZIPEXTRACT_USE_EXTERNAL 7zip
```

　　7zip 比 Scoop 自带的内置解压工具效率更高。

2. 开启全局命令自动完成

```
scoop config AUTOCOMPLETE true
```

　　这样可以通过 Tab 键自动补全 Scoop 命令。

6. 切换 git 下载工具

```
scoop config GIT_CLIENT git
```

　　使用 git 而不是内置的下载工具可以提速克隆仓库。

　　适当配置这些选项可以显著提升 Scoop 的性能和使用体验。

　　如何利用 aria2 进行断点续传？

---

　　先看具体示例：

```text
# 更新 vscode
scoop update vscode-portable
```

　　scoop 更新 vscode 时下载到 80%的时候 失败了（安装时处理方法也一样）。我们需要在提示中找到如下内容：

```text
'D:\Scoop\Applications\apps\aria2\current\aria2c.exe' --input-file='D:\Scoop\Applications\cache\vscode-portable.txt' 
--user-agent='Scoop/1.0 (+http://scoop.sh/) PowerShell/5.1 (Windows NT 10.0; Win64; x64; Desktop)' 
--allow-overwrite=true --auto-file-renaming=false 
--retry-wait=2 --split=5 --max-connection-per-server=5 
--min-split-size=5M --console-log-level=warn --enable-color=false 
--no-conf=true --follow-metalink=true --metalink-preferred-protocol=https 
--min-tls-version=TLSv1.2 --stop-with-process=15584 --continue
```

　　我们从上面的信息中提取出下面的命令；若要进行断点续传，只需再次执行下面的命令即可：

```text
aria2c.exe --input-file='D:\Scoop\Applications\cache\vscode-portable.txt'

```

　　当提示下载完成后，我们需要再次运行 scoop 对应的 app 更新命令（或安装命令），即可完成 app 更新（或安装）：

```text
scoop update vscode-portable
```

　　安装和切换JDK、Python的版本

---

　　这里需要使用 `scoop reset`​ 它的作用是：重置一个应用程序来解决冲突。

　　命令格式为：

```text
scoop reset <java>[@<version>]
```

　　**安装和切换不同的 JDK 版本：**

```text
PS C:> scoop bucket add java

PS C:> scoop install oraclejdk
Installing 'oraclejdk' (12.0.2-10) [64bit]

PS C:> scoop install zulu6
Installing 'zulu6' (6.18.1.5) [64bit]

PS C:> scoop install openjdk10
Installing 'openjdk10' (10.0.1) [64bit]

PS C:> java -version
openjdk version "10.0.1" 2018-04-17
OpenJDK Runtime Environment (build 10.0.1+10)
OpenJDK 64-Bit Server VM (build 10.0.1+10, mixed mode)

PS C:> scoop reset zulu6
Resetting zulu6 (6.18.1.5).
Linking ~\scoop\apps\zulu6\current => ~\scoop\apps\zulu6\6.18.1.5

PS C:> java -version
openjdk version "1.6.0-99"
OpenJDK Runtime Environment (Zulu 6.18.1.5-win64) (build 1.6.0-99-b99)
OpenJDK 64-Bit Server VM (Zulu 6.18.1.5-win64) (build 23.77-b99, mixed mode)

PS C:> scoop reset oraclejdk

PS C:> java -version
java version "12.0.2" 2019-07-16
Java(TM) SE Runtime Environment (build 12.0.2+10)
Java HotSpot(TM) 64-Bit Server VM (build 12.0.2+10, mixed mode, sharing)
```

　　安装和切换不同的 Python 版本：

```text
$ scoop bucket add versions # add the 'versions' bucket if you haven't already

$ scoop install python27 python
python --version # -> Python 3.6.2

# switch to python 2.7.x
$ scoop reset python27
python --version # -> Python 2.7.13

# switch back (to 3.x)
$ scoop reset python
python --version # -> Python 3.6.2
```

```text
$ scoop bucket add versions # add the 'versions' bucket if you haven't already

$ scoop install python27 python
python --version # -> Python 3.6.2

# switch to python 2.7.x
$ scoop reset python27
python --version # -> Python 2.7.13

# switch back (to 3.x)
$ scoop reset python
python --version # -> Python 3.6.2
