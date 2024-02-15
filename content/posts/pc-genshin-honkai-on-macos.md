+++
title = "在 macOS 上用 Game Porting Toolkit 运行原神崩铁"
date = "2023-07-23"
type = "post"
tags = ["Game"]
+++

参考了在 Linux 上运行《原神》和《崩坏：星穹铁道》的方法，利用 macOS 14 的 Game Porting Toolkit 成功运行了 PC 版的《原神》和《崩坏：星穹铁道》。记录一下过程。

简单来说就是 Wine 没有自带 DirectX 的实现，以前都只能用第三方社区实现的 DXVK。而今苹果官方直接自己实现了一套 DirectX 接口给 Wine，叫做 D3DMetal，这样就能跑起来各种游戏了。所以这套东西的关键是苹果官网下载的 [Game Porting Toolkit](https://developer.apple.com/download/all/?q=Game%20Porting%20Toolkit) 中的 D3DMetal 二进制。

目前网上暂时没找到其它运行 PC 崩铁的方法。PlayCover 用过一段时间，太难用，不适合键鼠——不会有人上班带手柄吧？

出于对一些项目的保护目的，有些工具不方便直接提供，你可以直接从上下文中找到信息，但我不会直接写出来。没有解谜，能一眼看出来，主要是为了防止搜索引擎检索关键字直接搜到这篇文章。

__⚠️绕过反作弊模块存在封号可能，风险自负⚠️__

可以考虑使用小号免费游玩（Free Game BTW），避免财产损失。

虽然原神这么多年了没见过封号案例，崩铁则是从 1.2 开始之后没有听说过封号案例（这之前有群友被封过几天），我至今也还没有被封过号，不过还是谨慎为上。

![运行效果图](https://xr1s.oss-cn-beijing.aliyuncs.com/Snipaste_2023-07-23_13-12-02.webp)

本人的系统是 MBP 18，处理器 Apple M1 Pro，内存 32GB。游戏全特效的话，平时还能 60 帧，打个绽放基本上就只剩 45 帧了。这两个游戏双开的情况会很卡，不要学我，我只是为了截图演示。

## 准备

首先至少需要 M1 苹果芯片的 MacBook，操作系统需要升级到 macOS 14，目前正在 Beta 中。

macOS 13 的系统就算装了 Game Porting Toolkit 也会直接提示无法使用。

关于怎么开启 Beta 更新推送，可以参考[苹果官网资料](https://developer.apple.com/cn/support/install-beta/)。

不建议普通用户升级测试版本。注意备份重要资料、注意提前准备降级回滚。

讲真，如果下面有一些略有些技术性的内容看不懂的话，真不建议升级 Beta 版本。

> 2024-02-15 更新：
>
> macOS 14 Sonoma 现在已经正式了。但是原神 4.4 还是需要 14.4 Beta 版本。原因不明。

以上是基本要求。

特殊网络环境下可能需要代理等工具，请自备。

## Game Porting Toolkit 安装

你需要[下载 Command Line Tools](https://developer.apple.com/download/all/?q=Command%20Line%20Tools)，我亲自测试过 Beta、Beta 2、Beta 7 都可以正常编译 Game Porting Toolkit，而 Beta 3 和 Beta 4 都无法编译，网上也有[数起编译失败的报告](https://developer.apple.com/forums/thread/732940)，没有测试过 Beta 5 和 Beta 6。同时你可以在这个页面提前搜索[下载最新版的 Game Porting Toolkit](https://developer.apple.com/download/all/?q=Game%20Porting%20Toolkit)，之后用得着。

> 2024-02-15 更新：
>
> Command Line Tools 现在可以直接用正式版来编译安装了，15 和 15.1 都可以正常编译安装运行。
>
> Game Porting Toolkit 可以用社区编译版，如果你不愿意自己编译的话，具体见下。

另外记得去官网下载原神和崩铁 PC 版本的安装包，安装 Game Porting Toolkit 非常耗时，可以提前下载一下安装包。你还可以找台 Windows 提前装好游戏本体，这样到需要的时候可以直接把 Windows 上的整个游戏目录复制到 macOS 中的对应位置就好，不需要再用启动器另外下载游戏了。这么做的好处是能并行安装 Game Porting Toolkit 和游戏本体，否则就需要先编译完 Game Porting Toolkit，再通过 Game Porting Toolkit 安装运行启动器后，最后才能通过启动器下载 50 多个 GB 的游戏本体。

然后你需要安装 Rosetta，这是苹果为 M1 系统提供的 x86\_64 翻译器，以便在 arm64 架构的机器上执行 x86\_64 应用程序的指令，你知道原神是 x86\_64 的。

```bash
/usr/sbin/softwareupdate --install-rosetta
```

你还需要安装 x86\_64 的 Homebrew，因为苹果也通过它来分发 Game Porting Toolkit 的组件（CrossOver Wine）。

可能需要注意的是，如果你已经安装 arm64 版的 Homebrew，它默认应该安装在 `/opt/homebrew` 下。x86\_64 的 Homebrew 默认安装在 `/usr/local/Homebrew` 下，Homebrew 自身是通过目录来隔离、区分的，请确保你的 arm64 Homebrew 没有安装到其它目录，以防混淆。接下来我也会使用完整路径，以便于区分。

首先进入 x86\_64 的 zsh 中，后续安装 Game Porting Toolkit 时的所有命令都应当在该环境下执行。

```bash
/usr/bin/arch -x86_64 zsh
```

从这里开始，请尽量不要退出当前 Shell 或打开新的终端、Shell，因为新的 Shell 默认还是 arm64 环境。如果不慎退出了当前 Shell，你可以再次输入上述命令重新进入 x86\_64 环境；如果你忘记了当前的终端是否正处于 x86_64 环境下，可以直接使用 `arch` 或 `uname -m` 命令来检查当前的架构。

接下来安装 x86\_64 的 Homebrew。记住先检查一遍 `/usr/local/Homebrew` 目录不存在或者为空，不要与 arm64 的 Homebrew 安装的软件混到一块儿。

```bash
/bin/bash -c "$(/usr/bin/curl -fsSL \
    https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

我个人不建议把 `/usr/local/bin` 和 `/usr/local/Homebrew/bin` 加入到 PATH 中来，还是因为混淆的问题。但是 macOS 默认 PATH 就包含 `/usr/local/bin`，因为它存在于 `/etc/paths` 中。从 `/etc/paths` 中删除对应行，未来打开新的终端的 PATH 中就不会有这一条，是否删除取决于你自己。

接下来的命令涉及到下载和编译，会花费大量的时间，需要耐心等待。

> 2024-02-15 更新：
>
> 或者你还有一个选择，直接安装 Gcenx 预编译的二进制，这位是 [DXVK-macOS](https://github.com/Gcenx/DXVK-macOS) 的作者：
>
> ```bash
> brew tap gcenx/homebrew-apple
> brew install gcenx/homebrew-apple/game-porting-toolkit
> ```
>
> 就可以使用了，但是官方提供的 Game Porting Toolkit.dmg 中的 D3DMetal 和一些其它库文件你还是需要手动复制到目录中去。


```bash
/usr/local/bin/brew tap apple/apple http://github.com/apple/homebrew-apple
/usr/local/bin/brew install apple/apple/game-porting-toolkit
```

如果愿意看编译日志的话，可以在 `install` 后面加一个 `-v`，免得漫长的等待使人焦虑。不过我一般都是通过 `htop` 看进程树观察编译情况，日志刷屏会让我没法一眼看出来当前进度。

[下载最新版的 Game Porting Toolkit](https://developer.apple.com/download/all/?q=Game%20Porting%20Toolkit) 完后将其挂载到文件树（双击打开就行），默认情况下会挂载到 `/Volumes/Game Porting Toolkit-1.0` 目录下，将 lib 目录复制到我们用 Homebrew 编译的 game-porting-toolkit 目录下。Game Porting Toolkit 1.0.4 版本之后将 lib 目录放到了 `/Volumes/Game Porting Toolkit-1.0/redist/lib` 下，记得自己将下面脚本中的目录改成自己对应的版本。

```bash
/usr/bin/rsync --archive \
  '/Volumes/Game Porting Toolkit-1.0/lib' \
  '/usr/local/opt/game-porting-toolkit'
```

镜像中 bin 目录下还有三个脚本，其实打开看的话，它们只是设置了一下环境变量和过滤了一些日志的简单脚本，而且写得也不太好。比如这些脚本不支持给 exe 传参，然而我们后面启动崩铁必须要给二进制传命令行参数，所以其实我们用不上这三个脚本。不用装这三个脚本直接调用 wine64 也能跑，这里就不装了。

那么到这里 Game Porting Toolkit 已经安装完毕了。如果你进到它在 Homebrew 的目录里观察一下，会发现它其实就是一个 Wine，不过它是苹果修改过后的 CrossOver Wine。

## Wine 配置

后面都已经不需要在 x86\_64 环境下了。

Wine 的中文字体有必要装一下，不影响游戏本体，但是轻的有遇到其它程序无法显示汉字的情况，严重有原神中打开「祈愿历史」、「版本热点」和「云原神」时卡死整个游戏进程导致无法关闭窗口、以及游戏关闭时也会卡死只能强杀进程的问题。

字体建议用 Winetricks 安装就可以了。Winetricks 就是一个纯 Shell 脚本，所以没有环境依赖问题，用 arm64 的 homebrew 安装就行，不需要指定 x86\_64 版本。等装完字体就可以删掉 Winetricks，当然留着它也无妨。

```bash
/opt/homebrew/bin/brew install winetricks
WINE=/usr/local/bin/wine64 \
WINESERVER=/usr/local/bin/wineserver \
    /opt/homebrew/bin/winetricks fakechinese
```

Winetricks 还有很多其它实用的功能，你可以用下面的命令打开图形界面的 Winetricks，这样稍微用户友好一些。图形界面的选项中有一个「选择默认的 Wine 容器」一项，这里稍作解释，如果你跟上面走的话，你的游戏和程序默认都安装在默认容器里，也就是 `~/.wine` 这个目录下。你可以在执行 Wine 之前通过设置环境变量 WINEPREFIX 到其它目录来设定你这次运行 Wine 的容器。

```bash
WINE=/usr/local/bin/wine64 \
WINESERVER=/usr/local/bin/wineserver \
    /opt/homebrew/bin/winetricks --gui
```

不建议「安装所有字体」，它的很多资源都丢失没有继续维护了，会中途失败。比如有些字体 404 了（会报错哈希没匹配上，实际上是下载了 404 HTML）、网站服务器挂了（会报错连接多次重试超时失败）。我对着它们调试修复了半天，最后发现没啥用，只需要安装 fakechinese 就能解决问题。

Wine 默认兼容的操作系统是 Windows 7，最好再执行一下 `winecfg.exe`，或者通过 Winetricks 打开 wine 配置程序，将 Windows 版本设置为 Windows 10。设置界面是图形化的，不难操作。

```bash
/usr/local/bin/wine64 winecfg.exe
```

## 原神

假如已经将原神安装包放到了 `~/Downloads` 目录，直接执行（记得换成自己的目录和文件名）：

```bash
WINEESYNC=1 \
    /usr/local/bin/wine64 ~/Downloads/yuanshen_setup_20230619213504.exe
```

在启动器安装完后，后续都可以通过下面的命令打开启动器。

```bash
WINEESYNC=1 \
    /usr/local/bin/wine64 'C:\Program Files\Genshin Impact\launcher.exe'
```

到这里原神已经可以运行了，你可以先启动游戏体验一把 macOS 上的 PC 原神。

> 2024-02-15 更新：
>
> 原神 4.3 版本出现过因为更新使用了 SSE 4.1 指令集导致整个版本无法启动的问题，在官方公告[《针对部分PC设备无法进入游戏的情况说明
》](https://www.miyoushe.com/ys/article/46994392)中也有提及，4.4 版本已经修复了。
>
> 原神 4.4 版本需要更新 macOS 到 14.4 Beta 版本才能启动，不清楚是什么原因，感谢评论中 [Ysrae1](https://github.com/Ysrae1) 提供的信息。

__⚠️最后提醒一遍，存在封号可能，风险自负。⚠️__

另外如果你需要查看帧率、内存使用率、显卡使用率等数据，可以通过设置环境变量执行，同理后面的崩铁也可以这么操作：

```bash
MTL_HUD_ENABLED=1 \
WINEESYNC=1 \
    /usr/local/bin/wine64 'C:\Program Files\Genshin Impact\launcher.exe'
```

但是我们不可能每次开游戏都打开终端敲一段这么长的命令行，为了方便，我们创建一个 macOS 应用来执行脚本启动游戏。打开「脚本编辑器」，新建文稿，输入：

```applescript
do shell script "
export WINEESYNC=1
/usr/local/bin/wine64 2>&1 \\
        'C:\\Program Files\\Genshin Impact\\launcher.exe' \\
    | /usr/bin/grep --fixed-string D3DM"
```

点击保存，文件格式选择「应用」即可，更具体内容参见官方[脚本编辑器使用手册](https://support.apple.com/zh-cn/guide/script-editor/welcome/mac)。

这里加上 grep 是为了防止大量的调试日志输出被 macOS 系统收集，这会导致在游戏退出后 macOS 系统试图弹一个很长的错误对话框，这往往会导致进程卡死。

出于美化目的，你可以为其修改图标，参见官方 [macOS 使用手册](https://support.apple.com/zh-cn/guide/mac-help/mchlp2313/mac)即可。

![](https://xr1s.oss-cn-beijing.aliyuncs.com/Snipaste_2023-07-23_16-36-55.webp)

图标资源可以从其他地方找一下，比如可以想办法从 PC 原神启动器本身提取，或者直接找网上公开的图标资源等等。

如果你愿意研究怎么制作 macOS 的图标文件的话，这里是一些参考资料：

1. [Extract icons from Windows executable](https://unix.stackexchange.com/questions/510338/extract-icons-from-windows-executable)
2. [How to create an `.icns` macOS app icon](https://gist.github.com/jamieweavis/b4c394607641e1280d447deed5fc85fc)

最后将其拖入 Applications 目录中，安装就算完成了。

## 崩铁

逻辑基本和原神相同。

已经将崩铁安装包放到了 `~/Downloads` 目录，同理执行（记得换成自己的目录和文件名）：

```bash
WINEESYNC=1 /usr/local/bin/wine64 ~/Downloads/StarRail_setup_gw_20230717.exe
```

等待启动器安装完毕、等待游戏本体下载安装完毕。

再后面就无法启动了，因为它有内核级别的反作弊，Wine 执行不了反作弊，于是 Wine 无法启动游戏。

首先屏蔽一些上报信息的域名，执行下述命令在 `/etc/hosts` 中插入域名：

```bash
base64 -d << EOF | sudo tee -a /etc/hosts
CjEyNy4wLjAuMSBsb2ctdXBsb2FkLm1paG95by5jb20KMTI3
LjAuMC4xIHB1YmxpYy1kYXRhLWFwaS5taWhveW8uY29tCg==
EOF
```

避免因为关键字被检索到，这里用 base64 加密了。

出于对项目的保护目的，这里不提供绕过反作弊的工具，你可以想办法从上下文中推断出信息来；或者通过邮件等方法联系我，我提供链接。

> 2024-02-15 更新：
>
> 崩铁在某个版本之后就无法通过上文提到的反·反作弊工具直接启动，这是绕过工具实现的问题。具体讨论可以参考该工具仓库的 Issue #39。
>
> 这里提供一下让它重新跑起来的操作：
>
> - 使用 [unstable-bh-wine-1.2](https://github.com/3Shain/winecx/releases/tag/unstable-bh-wine-1.2) 执行一遍。这样会在 <u>\${WINEPREFIX}/driver\_c/users/crossover/Temp</u> 下留下一个目录。这个目录中的文件是关键。
>
> - 再使用 Game Porting Toolkit 提供的 Wine 直接执行就可以正常启动了。

同样的，通过「脚本编辑器」创建并输入：

```applescript
do shell script "
export I_WANT_A_BAN=1
export WINEESYNC=1
/usr/local/bin/wine64 2>&1 \\
        'C:\\Program Files\\foo\\bar.exe' \\
        'C:\\Program Files\\Star Rail\\Game\\StarRail.exe' \\
        'C:\\Program Files\\Star Rail\\launcher.exe' \\
    | grep --fixed-string D3DM"
```

用同样的方法设置图标，就可以正常启动了。

## 一些问题

原神基本完全可用。原神 3.8 开始似乎删除了反作弊模块校验，所以可以直接用 Game Porting Toolkit 打开，不然也需要其它补丁。

原神之前（Game Porting Toolkit 1.0.2 版本）发现唯一的瑕疵就是草地贴图似乎有点问题，经常不显示或者闪烁。但是不影响游戏本身内容（该在草地上燃烧还是能燃烧）。Game Porting Toolkit 更新到 1.0.4 之后没有再出现过草地问题。

> 2024-02-15 更新：
>
> 原神 4.4 在直接传送到，或者从其它锚点慢慢走到轻策庄左下角锚点（那个一过去野猪就创你的锚点）似乎会遇到问题，导致游戏画面卡死、长时间后自行崩溃退出，并且再也无法进入游戏（卡完整岩然后长时间后崩溃退出）。这里就只能通过其它平台（Windows、手机、云游戏）先进入游戏传送走，才能从 macOS 原神启动。
>
> 挺奇怪的，这玩意儿折腾了我好久才意识到是传送点问题，导致我在春运动车上没事干只能闷头睡觉。好像是不能看到凉亭？可能是某个 Bug 导致渲染挂了。没有细致地做过实验具体是什么物件导致的问题。

崩铁使用了腾讯的 AntiCheatExpert 反作弊模块（相比起来，原神和崩三都是自研的），这是个内核模块，就是它导致了 Wine 无法直接运行崩铁。所以上文采用了特殊的绕过方法。

崩铁启动的时候因为这个绕过反作弊工具的限制，只能直接启动游戏本体，无法经过启动器启动。这大多数情况下都不会有问题，但是遇到版本更新的时候，可能需要手动打开启动器去下载新版内容，只需要像命令行启动原神一样打开崩铁启动器就行，然后还需要更新一下反·反作弊工具。这点还好，一般一个版本只需要一次。

崩铁启动的时候会弹窗提醒有封号风险，这是绕过工具的提醒。这是在代码中写死的，你可以通过修改源码自己重新编译绕过。

崩铁启动时音响会炸两下，不太清楚原因，但是游戏中声音是正常的。Game Porting Toolkit 1.0.4 似乎已经修复了这个问题。

还是提醒一遍，在崩铁 1.2 版本以前有大量因为使用了这个绕过方法导致短期中期封号的报告，<strong>风险自负。</strong>1.2 版本之后都暂时没见到封号，截止目前 1.3 版本，但是我无法做担保，__风险自负！__

还有退出时很偶尔可能才遇到本体已经关闭，但是 wineserver 还没关的情况，这时候可能需要手动去 `killall wineserver` 一遍，暂时没有好的解决方法。这个可能不是游戏的原因。

# 更新日志

## 2023-07-23

* Command Line Tools for Xcode 15 Beta 2
* Game Porting Toolkit Beta 2
* 原神 3.8
* 崩坏：星穹铁道 1.2

完成文章。

## 2023-08-31

* 更新 Command Line Tools 15 Beta 2 到 15 Beta 7
* 更新 Game Porting Toolkit Beta 2 到 Beta 7
* 更新原神到 4.0
* 更新崩坏：星穹铁道到 1.3

原神草地闪烁的问题得到修复。崩铁炸音响问题得到修复。

新增 3.8 版本下半「数据异常」的问题，提及 4.1 中新增的反作弊系统。

## 2024-02-15

* 建议 Command Line Tools for Xcode 使用 15.1 正式版本。
* 更新 Game Porting Toolkit Beta 7 到正式版 1.1。
* 更新原神到 4.4。
* 更新崩坏：星穹铁道到 2.0。

macOS 原神似乎没有再遇到「数据异常」问题了，再观察观察，因为 Linux 那边倒是出现了更多奇怪的数据异常，暂时可以通过 `kill -SIGSTOP` 的手段缓解，这里暂且不提，等 macOS 出现了再尝试解决。

4.1 中提及的反作弊系统更新似乎完全没有影响，删除。
