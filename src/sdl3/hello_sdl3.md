# SDL3 入门（1）：Hello, SDL3!

在本系列中我们使用 Windows Terminal + Powershell 组合作为我们在 Windows 系统下的终端工具，Windows 11 自带该环境。你也可以使用任意自己喜欢的终端环境代替，或使用鼠标执行等价的操作。
## 源码准备
我们使用 git 管理我们的项目，所以首先我们创建一个名为 "hello_sdl3" 的目录并且使用 git 进行初始化，这组命令实际上是各平台通用的：
```
mkdir hello_sdl3
cd ./hello_sdl3
git init
```
使用 git submodule 机制引入 SDL3 的源代码，源码地址： https://github.com/libsdl-org/SDL.git
```
git submodule add https://github.com/libsdl-org/SDL.git ./third_party/SDL3
```
如果不使用 git 可以手工下载源码到对应的目录，不影响后续使用。

接下来在项目文件夹根目录下创建源码文件和 CMakeLists.txt 文件：

hello_window.cpp
```
#include <SDL3/SDL.h>
#include <SDL3/SDL_main.h>

int main(int argc, char* argv[])
{
    SDL_Log("Hello, SDL3!");
    return 0;
}
```

CMakeLists.txt
```
cmake_minimum_required(VERSION 3.10)

project(hello_sdl3)
set(CMAKE_CXX_STANDARD 20)

add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/third_party/SDL3" EXCLUDE_FROM_ALL)
link_libraries(SDL3::SDL3)

add_executable(hello_window hello_window.cpp)
```
## 编译运行
创建并切换当前目录到 build 文件夹
```
mkdir build
cd ./build
```
执行 cmake 命令初始化项目配置，cmake 会自动扫描系统安装的开发环境，在 Windows 系统下一般是 VisualStudio（简称 VS）开发环境：
```
cmake ..
```
一切顺利的话已经可以看到 build 文件夹下生成的 VisualStudio 工程文件，接下来可以使用 VS 打开 sln 文件执行编译调试等工作，也可以继续使用 cmake 命令编译。

<img src=../img/sdl3_01_ls.png width="760"/>

cmake 命令编译项目：
```
cmake --build .
```
如果编译成功可以在 build/Debug 文件夹下看到 hello_window.exe 可执行文件，如果使用 clang 编译则直接在 build 文件夹下。命令行执行：

<img src=../img/sdl3_01_exe.gif width="642"/>

呃，没输出任何字符，有点不合预期 😓

这里是因为我们以动态链接库（DLL）的方式使用 SDL3，而 SDL3.dll 并没有生成到 build/Debug 文件夹下，hello_window.exe 可执行程序找不到需要的 DLL 所以无法启动。
解决方法是使用 CMake 的 `add_custom_command` 方法自定义文件拷贝指令，在 CMakeLists.txt 中增加如下内容：
``` 
add_custom_command(TARGET hello_window POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
    $<TARGET_FILE:SDL3::SDL3>
    $<TARGET_FILE_DIR:hello_window>
)
```
重新编译执行，输入如下：

<img src=../img/sdl3_01_result.png width="300"/>

搞定！

至此，说明我们已经解决基本的工程设置、动态库依赖等一系列问题，接下来可以继续编写更复杂的程序了。

## 注意事项
注意观察 hello_window.cpp 文件的源码，如果 IDE 比较智能的话可以看到 "main" 这个单词不是正常的表示函数名的颜色，比如在我这里它显示的是蓝色，这表示 "main" 是一个宏。
```
#include <SDL3/SDL.h>
#include <SDL3/SDL_main.h>

int main(int argc, char* argv[])
...
```
玄机在 SDL_main.h 中：
```
#if defined(SDL_MAIN_NEEDED) || defined(SDL_MAIN_AVAILABLE) || defined(SDL_MAIN_USE_CALLBACKS)
#define main SDL_main
#endif
```
"main" 被宏定义修改成了 "SDL_main"，因此 main 函数的声明必须写成带参数的完整形式，不能使用如下的无参写法：
```
int main()
{
    ...
}
```
否则会在链接时报 "无法解析的外部符号" 错误，因为无法解析的符号是 "SDL_main" 所以如果不熟悉 SDL 的话即使是有经验的 C++ 程序员（比如我）也颇为挠头（万恶的宏）。去掉 `#include <SDL3/SDL_main.h>` 这一行或者使用完整的 main 函数声明都可以解决问题。
不过 SDL_main.h 头文件还有其他用处，在 Windows 下如果希望将程序编译成窗口程序（启动后不带控制台）则必须包含这个头文件，同时在 CMakeLists.txt 中增加如下内容：
```
set_property(TARGET hello_window PROPERTY WIN32_EXECUTABLE TRUE)
```
不过这样在调试过程就看不到日志了，需要做一些额外的处理接管日志输出，所以在调试阶段建议继续使用控制台类型编译可执行程序。