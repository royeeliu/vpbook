# SDL3 入门（2）：第一个窗口

在上一篇文章中我们已经利用 SDL 的日志接口实现了简单的字符串输出，实际上是解决了开发环境搭建问题，接下来我们将在已有代码的基础上继续开发，实现第一个窗口的创建和背景色绘制。

## 初始化
首先设置日志输出级别：
```
SDL_SetLogPriorities(SDL_LOG_PRIORITY_VERBOSE);
```
因为还是开发阶段所以我们将输出日志级别设置为最低的 VERBOSE，这样所有的日志都会输出，有助于我们观察 SDL 的运行情况，出现错误时可以得到尽量详细的出错信息，有助于我们快速定位问题。

接下来初始化 SDL 库，参数 `SDL_INIT_VIDEO` 指定初始化的子系统为视频系统：
```
if (SDL_Init(SDL_INIT_VIDEO) < 0) {
    SDL_Log("SDL_Init failed: %s", SDL_GetError());
    return -1;
}

```
实际上这一步可以省略，因为在调用 SDL API 时其内部会自行检查和初始化所需使用的子系统。比如接下来要使用的 `SDL_CreateWindow` 函数，内部有这样的代码：
```
if (!_this) {
    /* Initialize the video system if needed */
    if (SDL_Init(SDL_INIT_VIDEO) < 0) {
        return NULL;
    }

    ...
}
```
其中 `_this` 是视频子系统初始化完成后设置的全局变量，声明如下：
```
static SDL_VideoDevice *_this = NULL;
```
虽然不是必须的，但是我们仍然建议调用 `SDL_Init` 对主要用到的子系统进行显式初始化，目的有两个：
1. 清晰完整的展示出执行过程，有助于理解代码；
2. 如果执行失败，可以在第一时间定位出错位置，提高排障效率。

## 创建窗口和渲染器
创建一个 800x600 大小的窗口，然后移动到屏幕居中的位置：
```
SDL_Window* window = SDL_CreateWindow("Hello, SDL3!", 800, 600, SDL_WINDOW_RESIZABLE);
if (!window) {
    SDL_Log("Could not create a window: %s", SDL_GetError());
    return -1;
}

SDL_SetWindowPosition(window, SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED);
```
创建和窗口关联的渲染器：
```
SDL_Renderer* renderer = SDL_CreateRenderer(window, nullptr);
if (!renderer) {
    SDL_Log("Create renderer failed: %s", SDL_GetError());
    return -1;
}
```
所有图形图像都是通过渲染器绘制到窗口，以背景色绘制为例，代码如下：
```
SDL_SetRenderDrawColor(renderer, 16, 0, 16, 255);
SDL_RenderClear(renderer);
SDL_RenderPresent(renderer);
```
这里使用的背景色是 RGB(16, 0, 16)，像我这样曾经用过 DirectDraw 渲染视频的老家伙们对这个颜色值应该很熟悉😔，不过这里用这个颜色只是为了和 SDL 窗口默认的黑色区分以便更好的观察渲染结果，没有其他特殊效果。

## 事件循环
SDL 有两种方式获取事件队列中的事件：
1. `SDL_PollEvent` 类似 Win32 API 中的 PeekMessage，无论队列中有无事件都会立即返回，区别只是返回值不同
2. `SDL_WaitEvent` 类似 Win32 API 中的 WaitMessage，如果队列中没有事件会阻塞等待，直到收到第一个事件才返回

我们使用第二种方式实现事件循环：
```
SDL_Event event{};
bool keep_going = true;

while (keep_going) {
    SDL_WaitEvent(&event);

    switch (event.type) {
        case SDL_EVENT_QUIT: {
            keep_going = false;
            break;
        }

        case SDL_EVENT_KEY_DOWN: {
            keep_going = keep_going && (event.key.keysym.sym != SDLK_ESCAPE);
            break;
        }

        case SDL_EVENT_WINDOW_EXPOSED: {
            SDL_SetRenderDrawColor(renderer, 16, 0, 16, 255);
            SDL_RenderClear(renderer);
            SDL_RenderPresent(renderer);
            break;
        }
    }
}
```
`SDL_EVENT_QUIT` 表示点击了关闭窗口按钮所以收到该事件后跳出循环；

`SDL_EVENT_KEY_DOWN` 键盘按下事件，这里我们实现了按 'Esc' 键退出的能力；

`SDL_EVENT_WINDOW_EXPOSED` 表示需要对窗口进行重绘，所以我们把绘制背景色的代码放在这个事件中执行。

可以在 while 循环中增加一行日志来观察收到了哪些事件：
```
SDL_Log("Event: %d", event.type);
```

注意上面这个事件循环的写法和大多数 SDL 的示例不同，实际上在这里我们是把 SDL 当作一个正经的窗口系统在使用，而不是当作一个游戏引擎。两者一个重要的区别是使用游戏引擎时一般是按照固定帧率持续进行窗口重绘，而一般的 GUI 软件使用窗口系统时只进行必要的重绘，以最大程度节省 CPU 和 GPU 的使用。视频虽然也有帧率的概念，但是采用的是第二种方式，只在视频帧刷新时执行重绘。

## 完整代码
添加退出前清理资源的代码后，完整的代码如下：
```
#include <SDL3/SDL.h>
#include <SDL3/SDL_main.h>

int main(int argc, char* argv[])
{
    SDL_SetLogPriorities(SDL_LOG_PRIORITY_VERBOSE);

    if (SDL_Init(SDL_INIT_VIDEO) < 0) {
        SDL_Log("SDL_Init failed: %s", SDL_GetError());
        return -1;
    }

    SDL_Window* window =
        SDL_CreateWindow("Hello, SDL3!", 800, 600, SDL_WINDOW_RESIZABLE);
    if (!window) {
        SDL_Log("Could not create a window: %s", SDL_GetError());
        return -1;
    }

    SDL_SetWindowPosition(window, SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED);

    SDL_Renderer* renderer = SDL_CreateRenderer(window, nullptr);
    if (!renderer) {
        SDL_Log("Create renderer failed: %s", SDL_GetError());
        return -1;
    }

    SDL_Event event{};
    bool keep_going = true;

    while (keep_going) {
        SDL_WaitEvent(&event);

        switch (event.type) {
            case SDL_EVENT_QUIT: {
                keep_going = false;
                break;
            }

            case SDL_EVENT_KEY_DOWN: {
                keep_going = keep_going && (event.key.keysym.sym != SDLK_ESCAPE);
                break;
            }

            case SDL_EVENT_WINDOW_EXPOSED: {
                SDL_SetRenderDrawColor(renderer, 16, 0, 16, 255);
                SDL_RenderClear(renderer);
                SDL_RenderPresent(renderer);
                break;
            }
        }
        SDL_Log("Event: %d", event.type);
    }

    SDL_DestroyRenderer(renderer);
    SDL_DestroyWindow(window);
    SDL_Quit();

    return 0;
}
```
输出效果如图：

<img src=../img/sdl3_02_window.png width="300"/>