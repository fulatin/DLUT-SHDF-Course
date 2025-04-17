# DLUT-SHDF-Course
大连理工大学 2024级智能硬件设计基础课程资料 包含部分实验代码


----
- [DLUT-SHDF-Course](#dlut-shdf-course)
  - [背景](#背景)
  - [使用方法](#使用方法)
    - [Windows](#windows)
    - [Linux](#linux)
  - [一些经验](#一些经验)
  - [编译 Error 一些可能的原因](#编译-error-一些可能的原因)

---
## 背景

大连理工大学2024级新添加了课程智能硬件设计基础
该课课程由理论课和实验课组成，实验课具有一定难度。
此仓库是进行代码与课程资料和经验的总结，以便给后来人参考。

目前只上传了实验三和实验四的代码

## 使用方法
- [ppt](./ppt) 中为理论课的代码
- [experiment_report](./experiment_report/) 中为老师发的实验报告模板
- [schemaic](./schematic/) 中为小车的原理图
- [code](./code/) 中为 ws63 的sdk和实验代码

实验的代码在 `code/src/application/course_experiment` 目录下

### Windows

由于我是在linux端编写代码的所以并未使用海思提供的HiSparkIDE，因此无法提供在Windows下的使用方法。欢迎其他人补充。

### Linux

首先克隆本项目

~~~bash
git clone --recursive https://github.com/fulatin/DLUT-SHDF-Course.git
~~~

进入 `code/src` 执行 `python ./build.py -c ws63-liteos-app menuconfig`
在`Application` 中选中 `Enable course experiment` 在子选项中可以选择某个实验(目前只有实验三和四的 欢迎pr)
![](./img/image.png)
保存并退出。之后输入
~~~bash
python ./build.py -c ws63-liteos-app
~~~
即可编译代码，随后的烧录可以使用[xf_burn_tools](https://github.com/geekheart/xf_burn_tools)这个工具。

烧录的示例代码为
~~~bash
burn -p /dev/ttyUSB0 DLUT-SHDF-Code/src/output/ws63/fwpkg/ws63-liteos-app/ws63-liteos-app_all.fwpkg
~~~

其中`/dev/ttyUSB0` 为linux下连接到小车的串口文件`DLUT-SHDF-Code/src/output/ws63/fwpkg/ws63-liteos-app/ws63-liteos-app_all.fwpkg`
为编译后需要烧录的文件的位置(以.fwpkg结尾，一般在`code/src/output/ws63/fwpkg/ws63-liteos-app/` 中可找到)

## 一些经验
*注:  这些经验仅为个人理解，仅供参考，以下谈论的根目录为 `/code/src`*

首先我们想知道我们的代码是如何在小车上运行的。简而言之编译后的 `.fwpkg` 文件中其实包含了一个非常精简的操作系统和你所写的应用程序的代码。把 `.fwpkg` 烧录到小车上的过程就类于给其安装操作系统。在其运行的时候会运行你所注册的任务，具体到代码就是
~~~c++
app_run(a_task_entry_point);
~~~

`app_run` 是一个宏定义，用来告诉操作系统他要运行哪个函数。

在 `a_task_entry_point` 一般会创建一个线程来运行真正的任务。

现在我们已经知道我们写的程序是如何运行的，接下来我们来看看它是如何编译到
 `.fwpkg` 文件里的。

 这主要由到 `Kconfig` 和 `CMakeLists.txt` 这两类文件来完成。
 - `Kconfig` 用来配置 `menuconfig` 的选项界面并生成 `CONFIG_XXXX` 宏定义供应用程序和 `CMakeLists.txt` 使用。
 - `CMakeLists.txt` 用来根据 `Kconfig` 生成的设置选项来生成 Makefile 文件或 ninja 文件（Makefile 和 ninja 文件是给 make 和 ninja 使用的来进行最终的编译）

举一个例子
> application/course_experiment/Kconfig
```Kconfig
config ENABLE_EXPERIMENT_3
    bool
    prompt "Enable experiment 3"
    default n
    depends on COURSE_EXPERIMENT_ENABLE
    help
        This enable the suppert for experiment 3

if ENABLE_EXPERIMENT_3
osource "application/course_experiment/experiment3/Kconfig"
endif
```
这段代码添加了一个编译选项 `ENABLE_EXPERIMENT_3` (是否启用实验三) 默认为 n 。在 menuconfig 中可以看到这个选项如果你启用了它所依赖的 `COURSE_EXPERIMENT_ENABLE` (是否启用实验) 选项。

随后在
> application/course_experiment/CMakeLists.txt

中有这样的几行
```cmake
if(DEFINED CONFIG_ENABLE_EXPERIMENT_3)
    add_subdirectory_if_exist(experiment3)  
endif()
```
这表示如果有 `CONFIG_ENABLE_EXPERIMENT_3` 这个宏定义就会尝试搜索当前文件夹下的 `experiment3` 文件夹下的 `CMakeList.txt` 文件

在 `application/course_experiment/experiment3/CMakeLists.txt` 中
```cmake
set(SOURCES "${SOURCES}"
    "${CMAKE_CURRENT_SOURCE_DIR}/main.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/io_expander.c" 
    "${CMAKE_CURRENT_SOURCE_DIR}/ssd1306.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/ssd1306_fonts.c"
PARENT_SCOPE)
```

最终设置了要编译的文件

了解了这个流程我们就明白如果没有在 `menuconfig` 中勾选 `ENABLE_EXPERIMENT_3` 和其所依赖的选项 `COURSE_EXPERIMENT_ENABLE` 那么就没有 `CONFIG_ENABLE_EXPERIMENT_3` 这个宏定义
，`application/course_experiment/experiment3/CMakeLists.txt` 这个文件就不会被读取，最终就会使 `experiment3` 中的文件不会被加到需要编译的文件中。

## 编译 Error 一些可能的原因
- 在Windows下用户名为中文
- sdk文件夹放的过深
- 未使用的变量未用unused()标明
- main函数无 `return` 语句
- main函数无传参
- .  .  .

更加复杂的错误问题可以查看 `src` 下的 `build.log` ，实在查不出来可以在本仓库提 issue 寻求帮助也许其他遇到过相似错误的人可以帮到你。