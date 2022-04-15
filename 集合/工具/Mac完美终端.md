

- [“完美”终端](https://makeoptim.com/tool/terminal#“完美”终端)
- 安装
  - [iTerm2](https://makeoptim.com/tool/terminal#iterm2)
  - [zsh](https://makeoptim.com/tool/terminal#zsh)
  - [Oh My Zsh](https://makeoptim.com/tool/terminal#oh-my-zsh)
  - [Powerlevel10k](https://makeoptim.com/tool/terminal#powerlevel10k)
- 常用插件
  - [autojump](https://makeoptim.com/tool/terminal#autojump)
  - [zsh-syntax-highlighting](https://makeoptim.com/tool/terminal#zsh-syntax-highlighting)
  - [zsh-autosuggestions](https://makeoptim.com/tool/terminal#zsh-autosuggestions)
- [VSCode 配置](https://makeoptim.com/tool/terminal#vscode-配置)
- [参考](https://makeoptim.com/tool/terminal#参考)

## “完美”终端

作为一个程序员，经常需要跟终端（Terminal）打交道。配置一个漂亮、好用的终端，不但心情愉悦，效率也能提升不少。

一个“完美”终端需要

- 漂亮的界面
- 高效的自动补全
- 实用的额外信息
- 自动推荐
- 语法高亮
- 随时唤起
- ……

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/0.gif)

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/13.gif)

本篇文章，带大家利用 [iTerm2](https://makeoptim.com/tool/(https://www.iterm2.com/)), [zsh](https://en.wikipedia.org/wiki/Z_shell), [oh my zsh](https://ohmyz.sh/), [powerlevel10k](https://github.com/romkatv/powerlevel10k) 快速打造一款“完美”终端。

## 安装

### iTerm2

前往 [iTerm2 官网](https://www.iterm2.com/)下载并安装。

> 注：建议为 iTerm2 打开完全磁盘访问权限，避免出现默认 Terminal 执行正确，iTerm2 因为权限问题导致执行有误。

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/1.png)

配置快捷键随时从顶部唤起以及背景图片。

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/14.png)

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/15.png)

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/16.png)

配置完成后，打开 iTerm2，在任何页面按下设置的快捷键，即可从顶部唤起。

### zsh

macOS 下默认已经安装了 zsh。可执行以下命令，更改默认 Shell 为 zsh。

```
1 chsh -s /bin/zsh 
```

### Oh My Zsh

[Oh My Zsh](https://ohmyz.sh/) 是这么介绍自己的。

> Oh My Zsh is a delightful, open source, community-driven framework for managing your Zsh configuration. It comes bundled with thousands of helpful functions, helpers, plugins, themes, and a few things that make you shout…

简单来说，利用 Oh My Zsh 我们可以轻松管理 zsh 的配置，可以做非常多的定制化功能，比如主题，字体，插件等。

Oh My Zsh 支持 curl、wget 安装，命令如下：

- curl：

  `1 sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" `

- wget：

  `1 sh -c "$(wget https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)" `

安装完成后，Oh My Zsh 会加载默认的主题。

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/2.png)

是不是比 zsh 原生的主题好看些了呢？下面，我们进一步完善。

### Powerlevel10k

Oh My Zsh 有上百个自带主题，以及其他的外部主题。而 [Powerlevel10k](https://github.com/romkatv/powerlevel10k) 正是现在最流行的主题之一。

执行以下命令，安装 Powerlevel10k。

```
1 git clone --depth=1 https://gitee.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k 
```

在 zsh 的配置文件 `~/.zshrc` 中设置 `ZSH_THEME=powerlevel10k/powerlevel10k`。

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/3.png)

设置完成后，重启 iTerm2 会提示安装需要的字体，根据提示安装即可。

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/4.png)

完成后，重启 iTerm2 进入配置页，根据提示选择自己喜欢的样式即可。

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/5.gif)

## 常用插件

### autojump

[autojump](https://github.com/wting/autojump) 可以记录下之前 cd 命令访过的所有目录，下次要去那个目录时不需要输入完整的路径，直接 j somedir 即可到达，甚至那个目标目录的名称只输入开头即可。

执行以下命令，安装 autojump。

```
1 brew install autojump 
```

在 zsh 的配置文件 `~/.zshrc` 中的 plugins 中加入 autojump。

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/6.png)

执行以下命令，使插件生效。

```
1 source ~/.zshrc 
```

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/7.gif)

### zsh-syntax-highlighting

[zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting) 终端命令语法高亮插件。

执行以下命令，安装 zsh-syntax-highlighting。

```
1 git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting 
```

在 zsh 的配置文件 `~/.zshrc` 中的 plugins 中加入 zsh-syntax-highlighting。

```
1 2 3 4 5 plugins=(  git  autojump  zsh-syntax-highlighting ) 
```

执行以下命令，使插件生效。

```
1 source ~/.zshrc 
```

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/8.png)

### zsh-autosuggestions

[zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions) 终端命令自动推荐插件，会记录之前使用过的命令，当你输入开头时，会暗色提示之前的历史命令供你选择，可直接按右方向键选中该命令。

执行以下命令，安装 zsh-autosuggestions。

```
1 git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions 
```

在 zsh 的配置文件 `~/.zshrc` 中的 plugins 中加入 zsh-autosuggestions。

```
1 2 3 4 5 6 plugins=(  git  autojump  zsh-syntax-highlighting  zsh-autosuggestions ) 
```

执行以下命令，使插件生效。

```
1 source ~/.zshrc 
```

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/12.gif)

## VSCode 配置

默认情况下，在 VSCode 中选择 zsh 作为默认 Shell 会出现乱码现象。原因是 Oh My Zsh 配置完成后，使用了 `MesloLGS NF` 字体。

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/9.png)

因此，修复乱码只需要在设置中找到 terminal font，设置成 `MesloLGS NF` 即可。

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/10.png)

![terminal](https://cdn.jsdelivr.net/gh/MakeOptim/jsdelivr@main/assets/img/tool/terminal/11.png)

其他终端工具，也类似修改字体即可。

## 参考

- https://support.apple.com/zh-cn/HT208050
- https://github.com/romkatv/powerlevel10k/
- https://ohmyz.sh/#install