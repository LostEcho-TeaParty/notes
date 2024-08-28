# Msys2 配置

当我真正开始熟悉 Linux Shell 时，我就希望能够在 windows 的命令行里使用相同的命令进行操作，而不是使用 Dos 命令或是 PowerShell 命令。与 Msys2 相关的软件还有一些，如 MinGW、Cygwin，它们之前的关系可以参考[这篇博客](https://kegalas.top/p/msys2mingw64cygwin%E7%9A%84%E4%BD%BF%E7%94%A8%E5%8C%BA%E5%88%AB%E6%B5%85%E8%B0%88/)。
在 Msys2 安装后，其下有很多的环境，这些环境的不同点在于一些环境变量、默认的编译器与链接器、架构等等，具体的内容需要参考 [Msys2 Environments](https://www.msys2.org/docs/environments/)，官方会去推荐使用 UCRT64，而我一般会去使用 MinGW64。它们之前的区别是 C Library 的选择，UCRT 使用的是 UCRT (Universal C Runtime)，而 MinGW64 使用的是 MSVCRT (Microsoft Visual C++ Runtime)，UCRT 会更新一点，这两个 C Library 的区别我还没有去更多的了解。

当然，我们是要去能够正常并且舒服的使用 Msys2，需要对 Msys2 进行一些配置，主要有以下方面：
+ Terminal
+ `PATH`设置
+ pacman 包管理器的使用
+ 软件包

## Terminal
如果想以 shell 的方式进行操作，那么这个软件是不可少的，window 本身就有带一些，如 cmd、powershell，Msys2 默认使用的 Mintty，当然可以直接去使用这些。但不用这些的原因是因为它们并不好看。在现代的电脑上可以用 [Windows Terminal](https://github.com/microsoft/terminal)，我一开始也是去使用这个的，但是这个软件越来越不好用，越来越慢，我也就放弃这个软件了。好在我发现了一个好的，[WezTerm](https://wezfurlong.org/wezterm/index.html)，这个小巧又好看。如果是老电脑的话，可以使用 [cmder](https://cmder.app/)。
当然选择这些软件的原因不仅仅是因为好看，还因为这些终端软件是可配置的，可以将 Msys2 集成进去。下面就主要说如何将 Msys2 集成到 WezTerm 当中，重要的是 Msys2 的启动命令。

Msys2的入口是其安装目录的`msys2_shell.cmd`，在运行时，还可以带一些`option`，可以通过`msys2_shell.cmd --help`命令来查看，具体如下：
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

比如`-use-full-path`最后作用就是执行`set MSYS2_PATH_TYPE=inherit`，也即设置全局变量`MSYS2_PATH_TYPE`为`inherit`，其作用是使用当前 windows 设定的`PATH`全局变量。在`msys2_shell.cmd`中只是设定了这个全局变量，其处理过程是在`bash`环境设定档（ `/etc/profile`）中进行的。

所以根据以上`option`的解释，我们可以得到在 terminal 中要运行的指令，
```shell
path/to/msys64/msys2_shell.cmd -defterm -here -no-start -mingw64 -shell bash
```
`PATH`环境变量我们需要再单独做些处理，就不再`option`中设置了。`shell`就用自带的`bash`，用`zsh`感觉挺慢的。

解决了最麻烦的启动命令，集成到 wezterm 中就会比较简单了。这里需要编写 wezterm 的配置文件，需要参考[配置文档](https://wezfurlong.org/wezterm/config/files.html#quick-start)。
根据文档，我们需要在`$HOME/.config/wezterm/wezterm.lua`目录下建立配置文件，配置文件中会描述很多有关外观、字体、快捷键，以及最关键的[默认启动程序](https://wezfurlong.org/wezterm/config/lua/config/default_prog.html?h=default_prog)和[启动菜单](https://wezfurlong.org/wezterm/config/lua/config/launch_menu.html?h=launch_menu)，总之给出`wezterm.lua`[^1]，基本上都是自注释的。
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

[^1]: 参考文章[用 WezTerm 替代 Windows Terminal](https://www.rayalto.pro/2023/10/25/wezterm-replace-windows-terminal/)，不喜欢文章里的讽刺语气，但内容给了我很多帮助。
