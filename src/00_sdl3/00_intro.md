# 关于 SDL3

SDL(Simple DirectMedia Layer) 是一个跨平台开发库，设计目标是提供对音频、键盘、鼠标、操纵杆和图形硬件的低级访问，主要用于视频渲染和游戏开发。

SDL3 目前已在发布预览阶段，相比 SDL2，SDL3 提供了对各种新技术的支持，如 HDR 支持、更好了高 DPI 支持等。API 也进行了大幅的改进提供了更多的自由度，比如可以选择图形引擎（Direct3D、OpenGL、Vulkan 等），又比如在 Windows 下可以实现与外部 D3D 代码的交互，这对实现高效的视频渲染是不可或缺的。

SDL 官网：https://www.libsdl.org/

SDL 源码：https://github.com/libsdl-org/SDL