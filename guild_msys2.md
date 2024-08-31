# Msys2 配置

当我真正开始熟悉 Linux Shell 时，我就希望能够在 windows 的命令行里使用相同的命令进行操作，而不是使用 Dos 命令或是 PowerShell 命令。与 Msys2 相关的软件还有一些，如 MinGW、Cygwin，它们之前的关系可以参考[这篇博客](https://kegalas.top/p/msys2mingw64cygwin%E7%9A%84%E4%BD%BF%E7%94%A8%E5%8C%BA%E5%88%AB%E6%B5%85%E8%B0%88/)。
在 Msys2 安装后，其下有很多的环境，这些环境的不同点在于一些环境变量、默认的编译器与链接器、架构等等，具体的内容需要参考 [Msys2 Environments](https://www.msys2.org/docs/environments/)，官方会去推荐使用 UCRT64，而我一般会去使用 MinGW64。它们之前的区别是 C Library 的选择，UCRT 使用的是 UCRT (Universal C Runtime)，而 MinGW64 使用的是 MSVCRT (Microsoft Visual C++ Runtime)，UCRT 会更新一点，这两个 C Library 的区别我还没有去更多的了解。

当然，我们是要去能够正常并且舒服的使用 Msys2，需要对 Msys2 进行一些配置，主要有以下方面：
+ Msys2 启动方式
+ `PATH`设置
+ Terminal
+ pacman 包管理器的使用
+ 软件包

## Msys2 启动方式
想要使用 Msys2，我们先需要对这个软件的启动方式有些了解。Msys2的入口是其安装目录的`msys2_shell.cmd`，在运行时，还可以带一些`option`，可以通过`msys2_shell.cmd --help`命令来查看，具体如下：
```
$ msys2_shell.cmd --help
Usage:
    msys2_shell.cmd [options] [login shell parameters]

Options:
    -mingw32 | -mingw64 | -ucrt64 | -clang32 | -clang64 |
    -msys[2] | -clangarm64           Set shell type
    -defterm | -mintty | -conemu     Set terminal type
    -here                            Use current directory as working
                                     directory
    -where DIRECTORY                 Use specified DIRECTORY as working
                                     directory
    -[use-]full-path                 Use full current PATH variable
                                     instead of trimming to minimal
    -no-start                        Do not use "start" command and
                                     return login shell resulting
                                     errorcode as this batch file
                                     resulting errorcode
    -shell SHELL                     Set login shell
    -help | --help | -? | /?         Display this help and exit

Any parameter that cannot be treated as valid option and all
following parameters are passed as login shell command parameters.
```
除了在启动文件这里加上`option`，还可以通过修改`msys2_shell.cmd`来控制一些功能，其实上面的那些`option`在被`msys2_shell.cmd`处理时，最后也是修改一些环境变量，所以直接修改`msys2_shell.cmd`来设置一些全局变量也完全是可行的。

比如`-use-full-path`最后作用就是执行`set MSYS2_PATH_TYPE=inherit`，也即设置全局变量`MSYS2_PATH_TYPE`为`inherit`，其作用是使用当前 windows 设定的`PATH`全局变量。在`msys2_shell.cmd`中只是设定了这个全局变量，其处理过程是在`bash`环境设定档（`/etc/profile`）中进行的。

如果需要更深入的了解`msys2_shell.cmd`，了解 cmd 的语法，可以参考[msys2_shell.cmd源码分析](https://ffmpeg.xianwaizhiyin.net/msys2/msys2_shell.html)

所以根据以上`option`的解释，我们可以得到在 terminal 中要运行的指令，
```shell
path/to/msys64/msys2_shell.cmd -defterm -here -no-start -mingw64 -shell bash
```
`PATH`环境变量我们需要再单独做些处理，就不再`option`中设置了。`shell`就用自带的`bash`，用`zsh`感觉挺慢的。如果要使用`zsh`，可以参考 Windows Terminal 与 MSYS2 MinGW64 集成记[^1]。

## 环境变量`PATH`设置
在使用 msys2 的时候当然想加载 window 的`PATH`变量，如果只是单单这个需求，那么就很方便的，在启动命令中加入`-use-full-path`选项，或者是在`msys2_shell.cmd`文件中加入`set MSYS2_PATH_TYPE=inherit`命令，这样就可以了。
但是这样做了之后，在 Msys2 中打印`PATH`变量就会发现，windows 的`PATH`变量数据会出现在末尾，而 Msys2 新加入的`PATH`数据则出现在前面。我记得最开始用 Msys2 并不是这样的，不知道从什么版本开始就变成现在这样了。这样会有一个麻烦，当 windows 与 Msys2 安装了相同的软件时，直接输入软件名称会去优先使用 Msys2 安装的，如 ssh。但是，在多数情况下，我都希望调用的是 windows 系统上安装的软件，所以基于这个需求，我需要将`PATH`数据的位置修改一下。
就像前面所说，在`msys2_shell.cmd`中只是设置了下全局变量`MSYS2_PATH_TYPE`，真正的处理是在`/etc/profile`中进行的，需要去到`/etc/profile`中进行修改。
在`/etc/profile`文件中，最后修改`PATH`变量的脚本如下：
```shell
case "${MSYSTEM}" in
MINGW*|CLANG*|UCRT*)
  MINGW_MOUNT_POINT="${MINGW_PREFIX}"
  PATH="${MINGW_MOUNT_POINT}/bin:${MSYS2_PATH}${ORIGINAL_PATH:+:${ORIGINAL_PATH}}"
  PKG_CONFIG_PATH="${MINGW_MOUNT_POINT}/lib/pkgconfig:${MINGW_MOUNT_POINT}/share/pkgconfig"
  PKG_CONFIG_SYSTEM_INCLUDE_PATH="${MINGW_MOUNT_POINT}/include"
  PKG_CONFIG_SYSTEM_LIBRARY_PATH="${MINGW_MOUNT_POINT}/lib"
  ACLOCAL_PATH="${MINGW_MOUNT_POINT}/share/aclocal:/usr/share/aclocal"
  MANPATH="${MINGW_MOUNT_POINT}/local/man:${MINGW_MOUNT_POINT}/share/man:${MANPATH}"
  INFOPATH="${MINGW_MOUNT_POINT}/local/info:${MINGW_MOUNT_POINT}/share/info:${INFOPATH}"
  ;;
*)
  PATH="${MSYS2_PATH}:/opt/bin${ORIGINAL_PATH:+:${ORIGINAL_PATH}}"
  PKG_CONFIG_PATH="/usr/lib/pkgconfig:/usr/share/pkgconfig:/lib/pkgconfig"
esac
```
在修改`PATH`变量前，先做了下选择，如果正确指定了环境`MSYSTEM`，就会将`PATH`设置为`PATH="${MINGW_MOUNT_POINT}/bin:${MSYS2_PATH}${ORIGINAL_PATH:+:${ORIGINAL_PATH}}"`，否则设置为`PATH="${MSYS2_PATH}:/opt/bin${ORIGINAL_PATH:+:${ORIGINAL_PATH}}"`，这两个我们都要改下，其中`ORIGINAL_PATH`变量中存放的就是 windows 下设置的`PATH`环境变量数据，当前被放在了最后，我们只要把这个变量放在最前面就好了。修改后的两条命令如下：
```shell
PATH="${ORIGINAL_PATH:+${ORIGINAL_PATH}}:${MINGW_MOUNT_POINT}/bin:${MSYS2_PATH}"

PATH="${ORIGINAL_PATH:+${ORIGINAL_PATH}}:${MSYS2_PATH}:/opt/bin"
```

## Terminal
如果想以 shell 的方式进行操作，那么这个软件是不可少的，window 本身就有带一些，如 cmd、powershell，Msys2 默认使用的 Mintty，当然可以直接去使用这些。但不用这些的原因是因为它们并不好看。在现代的电脑上可以用 [Windows Terminal](https://github.com/microsoft/terminal)，我一开始也是去使用这个的，但是这个软件越来越不好用，越来越慢，我也就放弃这个软件了。好在我发现了一个好的，[WezTerm](https://wezfurlong.org/wezterm/index.html)，这个小巧又好看。如果是老电脑的话，可以使用 [cmder](https://cmder.app/)。
当然选择这些软件的原因不仅仅是因为好看，还因为这些终端软件是可配置的，可以将 Msys2 集成进去。

根据之前说的启动命令，将 Msys2 作为 wezterm 的一种配置。这里需要编写 wezterm 的配置文件，需要参考[配置文档](https://wezfurlong.org/wezterm/config/files.html#quick-start)。
根据文档，我们需要在`$HOME/.config/wezterm/wezterm.lua`目录下建立配置文件，配置文件中会描述很多有关外观、字体、快捷键，以及最关键的[默认启动程序](https://wezfurlong.org/wezterm/config/lua/config/default_prog.html?h=default_prog)和[启动菜单](https://wezfurlong.org/wezterm/config/lua/config/launch_menu.html?h=launch_menu)，总之给出`wezterm.lua`[^2]，基本上都是自注释的。
```lua
local wezterm = require 'wezterm'
local c = {}
if wezterm.config_builder then
    c = wezterm.config_builder()
end

-- 初始大小
c.initial_cols = 96
c.initial_rows = 24

-- 关闭时不进行确认
c.window_close_confirmation = 'NeverPrompt'

-- 字体
c.font = wezterm.font 'Sarasa Mono SC'

-- 配色
local materia =
    wezterm.color.get_builtin_schemes()['Material Darker (base16)']
materia.scrollbar_thumb = '#cccccc'
c.colors = materia

-- 透明背景
c.window_background_opacity = 0.9
-- 取消 Windows 原生标题栏
c.window_decorations = "INTEGRATED_BUTTONS|RESIZE"
-- 滚动条尺寸为15，其下方不需要 pad
c.window_padding = {left = 0, right = 15, top = 0, bottom = 0}
-- 启用滚动条
c.enable_scroll_bar = true

-- 默认启动 MinGW64 / MSYS2
c.default_prog = {
    'D:/Software/msys64/msys2_shell.cmd',
    '-defterm',
    '-here',
    '-no-start',
    '-shell', 'bash',
    '-mingw64'
}

-- 启动菜单的一些启动项
c.launch_menu = {
    {label = 'MINGW64 / MSYS2', args = {
        'D:/Software/msys64/msys2_shell.cmd',
        '-defterm',
        '-here',
        '-no-start',
        '-shell', 'bash',
        '-mingw64'
    }},
    {label = 'CMD', args = {'cmd.exe'}}
}

-- 取消所有默认的热键
c.disable_default_key_bindings = true
local act = wezterm.action
c.keys = {
    -- Ctrl+Shift+Tab 遍历 tab
    {
        key = 'Tab',
        mods = 'SHIFT|CTRL',
        action = act.ActivateTabRelative(1)
    },
    -- F11 切换全屏
    { key = 'F11', mods = 'NONE', action = act.ToggleFullScreen },
    -- Ctrl+Shift++ 字体增大
    { key = '+', mods = 'SHIFT|CTRL', action = act.IncreaseFontSize },
    -- Ctrl+Shift+- 字体减小
    { key = '_', mods = 'SHIFT|CTRL', action = act.DecreaseFontSize },
    -- Ctrl+Shift+C 复制选中区域
    { key = 'C', mods = 'SHIFT|CTRL', action = act.CopyTo 'Clipboard' },
    -- Ctrl+Shift+N 新窗口
    { key = 'N', mods = 'SHIFT|CTRL', action = act.SpawnWindow },
    -- Ctrl+Shift+T 新 tab
    { key = 'T', mods = 'SHIFT|CTRL', action = act.ShowLauncher },
    -- Ctrl+Shift+Enter 显示启动菜单
    {
        key = 'Enter',
        mods = 'SHIFT|CTRL',
        action = act.ShowLauncherArgs {
            flags = 'FUZZY|TABS|LAUNCH_MENU_ITEMS'
        }
    },
    -- Ctrl+Shift+V 粘贴剪切板的内容
    { key = 'V', mods = 'SHIFT|CTRL', action = act.PasteFrom 'Clipboard' },
    -- Ctrl+Shift+W 关闭 tab 且不进行确认
    {
        key = 'W',
        mods = 'SHIFT|CTRL',
        action = act.CloseCurrentTab{ confirm = false }
    },
    -- Ctrl+Shift+PageUp 向上滚动一页
    { key = 'PageUp', mods = 'SHIFT|CTRL', action = act.ScrollByPage(-1) },
    -- Ctrl+Shift+PageDown 向下滚动一页
    { key = 'PageDown', mods = 'SHIFT|CTRL', action = act.ScrollByPage(1) },
    -- Ctrl+Shift+UpArrow 向上滚动一行
    { key = 'UpArrow', mods = 'SHIFT|CTRL', action = act.ScrollByLine(-1) },
    -- Ctrl+Shift+DownArrow 向下滚动一行
    { key = 'DownArrow', mods = 'SHIFT|CTRL', action = act.ScrollByLine(1) },
}

return c
```

当这些配置好之后，就可以非常好的使用了。这里还有一点需要注意一下，就是最好使用安装包的方式来安装 wezterm，因为这样可以在右键菜单中加入“Open WezTerm here”，这个是非常方便的，如果使用 portable 安装方式的话，就需要自己手动设置注册表了，比较麻烦。我最开始使用的就是 portbale 安装方式，之后想在右键菜单中加入相关设置，参照着[右键菜单加入用VSCode打开文件和文件夹](https://zhuanlan.zhihu.com/p/428704111)的方法进行设置。对照着相关的 wezterm 的 option 进行相关配置，但是最后有一些小问题，wezterm 无法正常关闭，最后还是又使用安装包的方式重新进行了安装。这里给出安装包在注册表中加入的设置，以供之后使用。
在`HKEY_CLASSES_ROOT\Directory\shell`地址下建立项`Open WezTerm here`，并在该项下面再建立项`command`。`Open WezTerm here`下新建`REG_SZ`类型数据，名称为`icon`，值为`wezterm-gui.exe`的地址。`command`下面将默认的`REG_SZ`数据的值改为`"D:\Software\WezTerm\wezterm-gui.exe" start --no-auto-connect --cwd "%V\\"`。这里是在右键文件夹时的弹出菜单中加入“Open WezTerm here”，如果要在空白处右键菜单中加入，需要在`HKEY_CLASSES_ROOT\Directory\Background\shell\`地址下做相同操作，`command`下面的值变更为`"D:\Software\WezTerm\wezterm-gui.exe" start --no-auto-connect --cwd "%V`（这个命令真奇怪，最后都没有冒号，不知道是漏了还是故意的）。

## pacman 包管理器的使用
Msys2 使用的是 Arch Linux 同款的包管理器，但我并没有使用过 Arch Linux，更多的是 Debian 系列。网上对 pacman 有很多介绍，这里记录一些我常用的命令，更多的可以去参考[wiki](https://wiki.archlinuxcn.org/wiki/Pacman)。

+ `pacman -Syu`：升级整个系统
+ `pacman -S 包名1 包名2 ...`：安装软件包
+ `pacman -Ss string1 string2 ...`：在包数据库中查询软件包
+ `pacman -Qs string1 string2 ...`：查询已安装的软件包
+ `pacman -R package_name`：删除单个软件包
+ `pacman -Rs package_name`：删除指定软件包，及其所有没有被其他已安装软件包使用的依赖关系

## 软件包
### git
我当时想着，git 的 windows 版也是基于 Msys2 做的，那我直接在 Msys2 安装 git 不就好了吗？这样想是不错，我也这样做了，直到我在用 vsCode 的时候发现 git 插件没有办法正常使用。上网找了一下发现是因为 Msys2 中的 git 对 windows 的地址信息处理不一样，网上有些人写了一段脚本叫`git-wrapper`，用来将 windows 的地址信息进行转换，但我试了下并不好使，最终还是安装了 windows 版本的 git，也就是说我的电脑里有了两个 Msys2。唉，没有办法，如果你再安装 octave 的话，我的电脑里就会有三个 Msys2 了。

### arm-none-eabi-gcc
之前在 Msys2 直接安装这个的时候好像 gdb 用起来有点问题，这个需要再次测试。

[^1]: [Win10终端神器——Windows Terminal 与 MSYS2 MinGW64 集成记](https://ttys3.dev/blog/windows-terminal-msys2-mingw64-setup)，这篇文章我真的很喜欢，我也按这些配置过，在我找到它的英文原版之前...

[^2]: 参考文章[用 WezTerm 替代 Windows Terminal](https://www.rayalto.pro/2023/10/25/wezterm-replace-windows-terminal/)，不喜欢文章里的讽刺语气，但内容给了我很多帮助。