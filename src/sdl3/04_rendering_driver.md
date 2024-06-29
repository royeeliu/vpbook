# SDL3 入门（4）：选择图形引擎

SDL2 创建渲染器时只能指定使用软件渲染还是硬件加速，无法选择使用哪种图形引擎实现硬件加速。SDL3 对此做了优化，可以在创建渲染器时指定 *rendering driver* 也就是图形引擎，比如在 Windows 平台下可以指定使用 D3D11 也可以指定使用 OpenGL 或者 Vulkan。

## 指定图形引擎
`SDL_CreateRenderer` 函数的第二个参数 `name` 表示指定使用的 *rendering driver name*，传 `NULL` 表示使用第一个支持的 *rendering driver*，在 Windows 系统下通常是 D3D11。
```
SDL_Renderer * SDLCALL SDL_CreateRenderer(SDL_Window *window, const char *name)
```

SDL3 接口文件中没有预定义 *rendering driver name*，可以通过 `SDL_GetNumRenderDrivers` 和 `SDL_GetRenderDriver` 两个函数枚举当前所支持的图形引擎：
```
int count = SDL_GetNumRenderDrivers();
for (int i = 0; i < count; ++i) {
    const char* name = SDL_GetRenderDriver(i);
    SDL_Log("Render driver[%d]: %s", i, name);
}
```
在 Windows 系统下执行结果如下：
```
INFO: Render driver[0]: direct3d11
INFO: Render driver[1]: direct3d12
INFO: Render driver[2]: direct3d
INFO: Render driver[3]: opengl
INFO: Render driver[4]: opengles2
INFO: Render driver[5]: vulkan
INFO: Render driver[6]: software
```
其中 *direct3d* 指的是 D3D9，*software* 指软件渲染，我们可以通过这些名字指定渲染器使用的渲染引擎。

## 渲染性能测试
实现一个简单的类用来测试渲染帧率
```
// performance.h
...

class Performance final
{
public:
    Performance();
    ~Performance();

    void Reset();
    void IncreaseFrameCount();
    void PrintEverySecond();

private:
    using Clock = std::chrono::high_resolution_clock;
    using TimePoint = std::chrono::time_point<Clock>;

    TimePoint start_time_;
    uint64_t frame_count_ = 0;

    TimePoint last_print_time_;
    uint64_t last_frame_count_ = 0;
};
```
```
// performance.cpp
...

Performance::Performance()
{
    Reset();
}

Performance::~Performance() {}

void Performance::Reset()
{
    start_time_ = Clock::now();
    last_print_time_ = start_time_;
    frame_count_ = 0;
}

void Performance::IncreaseFrameCount()
{
    frame_count_++;
}

void Performance::PrintEverySecond()
{
    assert(start_time_.time_since_epoch().count() > 0);
    assert(last_print_time_ >= start_time_);
    auto now = Clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(now - last_print_time_);
    if (duration.count() >= 1000) {
        double elapsed_seconds =
            std::chrono::duration_cast<std::chrono::milliseconds>(now - start_time_).count() /
            1000.0;
        double average_fps = frame_count_ / elapsed_seconds;
        double realtime_fps = (frame_count_ - last_frame_count_) / (duration.count() / 1000.0);
        last_print_time_ = now;
        last_frame_count_ = frame_count_;

        fprintf(stderr, "Performance: FPS(AVR|RT): %.2f|%.2f      \r", average_fps, realtime_fps);
    }
}
```

我们先测试 D3D11 的渲染帧率：
```
INFO: Created renderer: direct3d11
INFO: VSync: 0
Performance: FPS(AVR|RT): 32805.16|34470.00   
```
3 万多帧，还不错。这是全力渲染的结果，没有开启垂直同步，可以调用 `SDL_SetRenderVSync` 函数设置垂直同步：
```
int vsync = disable_vsync ? 0 : 1;
SDL_SetRenderVSync(renderer, vsync);
SDL_Log("VSync: %d", vsync);
```
再次测试渲染帧率，60fps，和屏幕刷新率一致：
```
INFO: Created renderer: direct3d11
INFO: VSync: 1
Performance: FPS(AVR|RT): 59.95|60.04   
```

关闭垂直同步，对各渲染引擎分别进行帧率测试，结果如下：
|direct3d11|direct3d12|direct3d(9)|opengl|opengles2|vulkan|software|
| -------- | -------- | ------ | ---- | ------- | ---- | ------- |
| 33113.44 | 1155.43 | 1729.66 | 1673.66 | 1716.95 | 1565.44 | 1668.42 |

D3D11 一骑绝尘，看来优化的不错。

注意这只是一个简单的测试，性能瓶颈主要在从 CPU 提交渲染指令到 GPU 的过程，所以不代表 D3D11 的渲染性能和其他图形引擎真的有这么大的差距。实际上对于复杂的图形渲染，除软件渲染外所有基于 GPU 的渲染性能上不会有太大的差距。