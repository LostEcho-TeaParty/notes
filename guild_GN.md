# GN

唉，为什么要折腾 GN 呢？还不是因为 cmake 实在是太难用了。在 cmake 里折腾了好久，最后还是不想用，不过学到那些知识可能还是需要记录一下，不然又全部忘记了。

GN 是 Google 出品的元构建系统，生成 Ninja 构建文件。Chromium 就是使用 GN 进行构建的，而且又因为其生成的 Ninja 构建文件，所以这套系统是可以跨平台的，这正是我想要的，学习能力实在是有限，使用的平台又多，折腾不动了。

不过还是需要吐槽一下 Google 编写的文档，感觉和我学习的英语不是同一种一样，语法也看不太懂，使用的词又很偏僻，还有些词完全就是借用来的，其真实意思根本就不清楚。总之非常的难懂。可能还是自己的英语不行，想起看 TI 与 ST 写的芯片手册，同样看得也是非常的困难，心累。

## 编译链编写

对于我现在最大的难点就是去编写这个编译链了。难点有二
+ 需要搞清楚 GN 是如何控制编译链的
+ 编译链本身的那些 options

聚焦一下，第一个问题，需要参考 `tool` 的用法，可以使用 `gn help tool` 查看更加详细的内容，也可以从[官方的帮助文档](https://gn.googlesource.com/gn/+/main/docs/reference.md#func_tool)查看。首先贴一下在教程里面的编译链，位置为 `//build/toolchain/BUILD.gn`，编译链的标识符为 `//build/toolchain:gcc`。

```gn
# Copyright 2014 The Chromium Authors. All rights reserved.
# Use of this source code is governed by a BSD-style license that can be
# found in the LICENSE file.

toolchain("gcc") {
  tool("cc") {
    depfile = "{{output}}.d"
    command = "gcc -MMD -MF $depfile {{defines}} {{include_dirs}} {{cflags}} {{cflags_c}} -c {{source}} -o {{output}}"
    depsformat = "gcc"
    description = "CC {{output}}"
    outputs =
        [ "{{source_out_dir}}/{{target_output_name}}.{{source_name_part}}.o" ]
  }

  tool("cxx") {
    depfile = "{{output}}.d"
    command = "g++ -MMD -MF $depfile {{defines}} {{include_dirs}} {{cflags}} {{cflags_cc}} -c {{source}} -o {{output}}"
    depsformat = "gcc"
    description = "CXX {{output}}"
    outputs =
        [ "{{source_out_dir}}/{{target_output_name}}.{{source_name_part}}.o" ]
  }

  tool("alink") {
    command = "ar rcs {{output}} {{inputs}}"
    description = "AR {{target_output_name}}{{output_extension}}"

    outputs =
        [ "{{target_out_dir}}/{{target_output_name}}{{output_extension}}" ]
    default_output_extension = ".a"
    output_prefix = "lib"
  }

  tool("solink") {
    soname = "{{target_output_name}}{{output_extension}}"  # e.g. "libfoo.so".
    sofile = "{{output_dir}}/$soname"
    rspfile = soname + ".rsp"
    if (is_mac) {
      os_specific_option = "-install_name @executable_path/$sofile"
      rspfile_content = "{{inputs}} {{solibs}} {{libs}}"
    } else {
      os_specific_option = "-Wl,-soname=$soname"
      rspfile_content = "-Wl,--whole-archive {{inputs}} {{solibs}} -Wl,--no-whole-archive {{libs}}"
    }

    command = "g++ -shared {{ldflags}} -o $sofile $os_specific_option @$rspfile"

    description = "SOLINK $soname"

    # Use this for {{output_extension}} expansions unless a target manually
    # overrides it (in which case {{output_extension}} will be what the target
    # specifies).
    default_output_extension = ".so"

    # Use this for {{output_dir}} expansions unless a target manually overrides
    # it (in which case {{output_dir}} will be what the target specifies).
    default_output_dir = "{{root_out_dir}}"

    outputs = [ sofile ]
    link_output = sofile
    depend_output = sofile
    output_prefix = "lib"
  }

  tool("link") {
    outfile = "{{target_output_name}}{{output_extension}}"
    rspfile = "$outfile.rsp"
    if (is_mac) {
      command = "g++ {{ldflags}} -o $outfile @$rspfile {{solibs}} {{libs}}"
    } else {
      command = "g++ {{ldflags}} -o $outfile -Wl,--start-group @$rspfile {{solibs}} -Wl,--end-group {{libs}}"
    }
    description = "LINK $outfile"
    default_output_dir = "{{root_out_dir}}"
    rspfile_content = "{{inputs}}"
    outputs = [ outfile ]
  }

  tool("stamp") {
    command = "touch {{output}}"
    description = "STAMP {{output}}"
  }

  tool("copy") {
    command = "cp -af {{source}} {{output}}"
    description = "COPY {{source}} {{output}}"
  }
}
```

在最外层，`toolchain()` 函数中包含了一系列的 `command` 和 `build flags` 来对编译链的行为进行定义。对于简单的构建程序，该函数下只需要定义 `tool()` 函数就可以了，这也是解决开始问题的重点。
`tool()` 函数的用法如下：
```gn
tool(<tool type>) {
  <tool variables...>
}
```

其中 `tool type` 就是编译时常用的工具，如编译器、链接器等等。以下只给出目前关心的工具。
+ `cc`：C 编译器
+ `cxx`：C++ 编译器
+ `asm`：汇编器
+ `alink`：静态库链接器，链接成为静态库，Linker for static libraries (archives)
+ `solink`：动态库链接器，链接成为动态库，Linker for shared libraries
+ `link`：可执行程序链接器，链接成为可执行程序，Linker for executables
+ `stamp`：创建文件工具？Tool for creating stamp files
+ `copy`：复制文件工具

接下来就是 `tool variables`，这里面的内容也有很多，自然也只记录些我目前关心的内容。
+ `command  [string with substitutions]`：这个无疑是最重要的命令，当需要使用上面定义的工具的时候，实际上就是执行 `command`，其内容是 `string with substitutions`，表示为包含有替代部分的字符串（下同），因为很多的指令都是会有 `options` 去进行控制，不能用一条固定的字符串，可替代的部分也是为了方便修改那些 `option` 与输入输出的文件名称。
+ `command_launcher [string]`：在执行 `command` 的前缀，可以是工具所在位置。
+ `default_output_dir  [string with substitutions]`：适用于链接器工具，给工具设定默认的输出地址。实现的方式是给 `{{output_dir}}` 设置上默认值，当手动设置 `{{output_dir}}` 的值时，默认值会被覆盖。
+ `default_output_extension  [string]`：适用于链接器工具，给工具设定输出文件的默认格式。通过给 `{{output_extension}}` 设置默认值。
+ `depfile  [string with substitutions]`：设定依赖描述文件的名称，这些文件用于列出在构建时发现的头文件依赖项（或其他隐式输入依赖项）
+ `depsformat  [string]`：设定依赖描述文件的类型。This is either "gcc" or "msvc". See the ninja documentation for "deps" for more information.
+ `description  [string with substitutions, optional]`：运行命令时将会打印的信息。
+ `outputs  [list of strings with substitutions]`：用来指定输出文件名称的列表
+ `output_prefix  [string]`：适用于链接器工具，可选，默认为空。输出文件名的前缀

接下来描述的是在定义上方的 `tool variables` 时可以被替换的部分，也就是 `[string with substitutions]` 中的 `substitutions`。
+ `{{label}}`：The label of the current target.
+ `{{label_name}}`：The short name of the label of the target. 不受“output_prefix”影响。
+ `{{target_output_name}}`：The short name of the current target with no path information, or the value of the "output_name" variable if one is specified in the target. 受“output_prefix”影响。
+ `{{output}}`：The relative path and name of the output(s) of the current build step.
+ 编译相关的 `options`
    - `{{asmflags}}`
    - `{{cflags}}`：c/c++ 编译选项
    - `{{cflags_c}}`：c 编译选项
    - `{{cflags_cc}}`：c++ 编译选项
    - `{{cflags_objc}}`
    - `{{cflags_objcc}}`
    - `{{defines}}`：宏定义
    - `{{include_dirs}}`：头文件目录
    Defines will be prefixed by "-D" and include directories will be prefixed by "-I" (these work with Posix tools as well as Microsoft ones).
+ `{{source}}`：The relative path and name of the current input file.
+ `{{inputs}}`：Expands to the inputs to the link step. This will be a list of object files and static libraries.