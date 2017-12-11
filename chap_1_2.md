# 环境搭建

对于开发者来讲，一个友好的、强大的开发环境可以起到事半功倍的作用。这个章节中我们会把本书需要安装的开发环境和推荐的 IDE 配置。

## Spring Boot 的环境安装

### Ubuntu Linux

在 `Linux` 环境下，推荐使用 `SDKMAN` 来安装 `Spring`、`Java` 以及 `Grails` 等依赖类库。安装 `SDKMAN` 非常简单，打开一个 `terminal` \(命令行终端窗口\)，然后输入以下命令，请一定注意需要科学上网才能确保安装成功：

```
curl -s "https://get.sdkman.io" | bash
```

注意在此过程中，安装脚本可能会提示你缺少某些依赖的软件包，这时我们需要按屏幕提示进行软件依赖的安装，比如下面的这个提示，说的是我们的系统没有安装 zip 和 unzip

```
Looking for a previous installation of SDKMAN...
Looking for unzip...
Looking for zip...
Not found.
======================================================================================================
 Please install zip on your system using your favourite package manager.

 Restart after installing zip.
======================================================================================================
```

那么我们需要使用下面命令进行安装

```
apg-get install zip unzip
```

安装好依赖后，请再次执行上面的 `SDKMAN` 安装脚本并按屏幕提示完成安装。如果你可以看到类似下面的输出，那么安装过程就结束了

```
All done!


Please open a new terminal, or run the following in the existing one:

    source "/home/ubuntu/.sdkman/bin/sdkman-init.sh"

Then issue the following command:

    sdk help

Enjoy!!!
```

安装好之后，在 `terminal` 中再输入以下命令

```
source "$HOME/.sdkman/bin/sdkman-init.sh"
```

这样就安装好了，你可以通过以下命令来验证安装是否成功：

```
sdk version
```

使用 `sdk` 这个命令就可以很方便安装依赖类库

### macOS

`macOS` 对于开发来讲是非常友好的，首先我们需要安装 `brew` ，这是 `macOS` 上的一个包管理工具，类似于 `Ubuntu` 中的 `apt-get` 。在 `terminal` 中键入下面的命令即可完成安装。

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

有了 \`brew\` 之后，

