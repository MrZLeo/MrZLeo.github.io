


>  gn 是 Google 的大型项目采用的编译系统，广泛应用在 chromium、Fuchsia 等项目中，主要的功能是生成 ninja 文件，再通过 ninja 编译真实的代码，与 CMake 的逻辑类似
>
> Gn 的特点是为**大项目**而生，因此并不一定能保证构建配置的简单，而是清晰、结构化、易于维护

# 根目录结构

根目录需要有一个 `.gn` 文件，最少要包含构建的配置文件路径：

```gn
# The location of the build configuration file.
buildconfig = "//build/BUILDCONFIG.gn"
```

# BuildConfig 规则

对应的配置文件需要描述当前的编译规则：

```gn
# Copyright 2014 The Chromium Authors. All rights reserved. # Use of this source code is governed by a BSD-style license that can be # found in the LICENSE file.
if (target_os == "") {
  target_os = host_os
}
if (target_cpu == "") {
  target_cpu = host_cpu
}
if (current_cpu == "") {
  current_cpu = target_cpu
}
if (current_os == "") {
  current_os = target_os
}

is_linux = host_os == "linux" && current_os == "linux" && target_os == "linux"
is_mac = host_os == "mac" && current_os == "mac" && target_os == "mac"

# All binary targets will get this list of configs by default.
_shared_binary_target_configs = [ "//build:compiler_defaults" ]

# Apply that default list to the binary target types.
set_defaults("executable") {
  configs = _shared_binary_target_configs

  # Executables get this additional configuration.
  configs += [ "//build:executable_ldconfig" ]
}
set_defaults("static_library") {
  configs = _shared_binary_target_configs
}
set_defaults("shared_library") {
  configs = _shared_binary_target_configs
}
set_defaults("source_set") {
  configs = _shared_binary_target_configs
}

set_default_toolchain("//build/toolchain:gcc")

```

目录下有简单的跨平台编译的规则，同时也有不同构建成果的默认设置，其中 `//build:compiler_defaults` 这种形式表达的是下 `$ROOTDIR/build` 目录下的（`BUILD.gn`）的 `compiler_defaults` 变量。

# 可执行文件

可执行文件的编译规则以 `executable` 作为前缀，在 `tutorial` 目录下的 `BUILD.gn` 描述：

```gn
executable("tutorial") {
  sources = [
    "tutorial.cc",
  ]
}
```

接着可以在根目录的 `BUILD.gn` 中添加 `tutorial` 的信息，这里因为是可执行文件，无法添加成某些其他程序的依赖，因此构建一个 `group` 来实现：

```gn
group("tools") {
    deps = [
        "//tutorial",
    ]
}
```

`group` 在 gn 中表达的是一系列不可以被编译 or 链接的目标

# 编译执行

编译执行分两个步骤：
1. 使用 gn 生成 ninja 的编译脚本
2. 使用 ninja 编译

为了编译执行我们的 `tutorial`：

```bash
gn gen out
ninja -C out tutorial
./out/tutorial
```

# 链接库

动态和静态链接库的构建规则也非常简单：

```gn
executable("hello") {
  sources = [ "hello.cc" ]

  deps = [
    ":hello_shared",
    ":hello_static",
  ]
}

shared_library("hello_shared") {
  sources = [
    "hello_shared.cc",
    "hello_shared.h",
  ]

  defines = [ "HELLO_SHARED_IMPLEMENTATION" ]
}

static_library("hello_static") {
  sources = [
    "hello_static.cc",
    "hello_static.h",
  ]
}
```

> 可以注意下路径和作用域

# Config 配置

因为链接库总是需要考虑编译的各种配置，例如 Flags、defines 以及 include 的文件夹路径，所以可以把这些都写入到 config 中，gn 提供了这样的功能：

```gn
config("my_lib_config") {
  defines = [ "ENABLE_DOOM_MELON" ]
  include_dirs = [ "//third_party/something" ]
}

static_library("hello_shared") {
  ...
  # Note "+=" here is usually required, see "default configs" below.
  configs += [
    ":my_lib_config",
  ]
}
```

- 可以用 `+` 、`-` 来增加和删除配置

```gn
config(“icu_dirs”) {
	include_dirs = [ "include" ]
}

shared_library(“icu”) {
	public_configs = [ ":icu_dirs" ]
}

executable(“doom_melon”) {
	deps = [
		# Apply ICU’s public_configs.
		":icu",
	]
}
```

- `public_configs` 可以用来定义所有依赖你的目标的配置
- 这个 config 是可以传播的
-

# More about labels

### Full label

```
//chrome/browser:version
	-> Looks for “version” in chrome/browser/BUILD.gn
```
​
### Implicit name

```
//base
	-> Shorthand for //base:base
```

### In current file

```
:baz
	-> Shorthand for “baz” in current file.
```

# Built-in target types

- **executable**, **shared_library**, **static_library**
- **loadable_module:** like a shared library but loaded at runtime
- **source_set:** compiles source files with no intermediate library
- **group:** a named group of targets (deps but no sources)
- **copy**
- **action,** **action _foreach**: run a script
- **bundle_data**, **create_bundle**: Mac & iOS

# 动态加载

- 有些测试需要依赖某些数据文件，可以动态加载：

```gn
test(“doom_melon_tests”) {
	# This file is loaded @ runtime.
    data = [
        "melon_cache.txt",
    ]
}
```

# BUILD. Gn 和 xxx. Gni

- `BUILD.gn` 一般作为模块的工程入口文件，可以配合 `.gni` 文件来划分源码或模块。
- 当模块比较大时，可以用 `.gni ` 来划分内部子模块，减轻工程文件的负担
- 当多个模块之间差异很小时，可以利用 BUILD.gn 来统一管理这些模块的设置，利用.gni 来专属负责各个模块源文件管理。