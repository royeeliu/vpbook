# SDL3 入门（3）：三角形

SDL3 提供了 `SDL_RenderGeometry` 函数绘制几何图形，用法和 OpenGL 差不多，先定义顶点数据，然后根据顶点数据绘制几何图形。

绘制三角形的代码如下：
```
std::array<SDL_Vertex, 3> origin_vertices = {
    SDL_Vertex { { 150, 100 }, { 1.0f, 0.0f, 0.0f, 1.0f } }, // top
    SDL_Vertex { { 000, 300 }, { 0.0f, 1.0f, 0.0f, 1.0f } }, // left bottom
    SDL_Vertex { { 300, 300 }, { 0.0f, 0.0f, 1.0f, 1.0f } } // right bottom
};

SDL_RenderGeometry(renderer, nullptr, origin_vertices.data(), origin_vertices.size(), nullptr, 0);
```
效果如图：

<img src=../img/sdl3_03_triangle.png width="300"/>

注意上面的顶点定义，坐标虽然是浮点数但表示的是像素值，颜色则做了归一化（取值范围 0.0~1.0）这一点和 SDL2 不同，SDL2 中颜色取值范围是 0~255.

可以通过实时修改顶点坐标实现运动效果。首先修改事件循环，由重绘事件 `SDL_EVENT_WINDOW_EXPOSED` 触发渲染改成空闲时自动渲染，对应的读取事件的方式也由 `SDL_WaitEvent` 改为 `SDL_PollEvent`：
```
SDL_Event event {};
bool keep_going = true;

while (keep_going) {
    while (SDL_PollEvent(&event)) {
        switch (event.type) {
        case SDL_EVENT_QUIT:
            keep_going = false;
            break;

        case SDL_EVENT_KEY_DOWN: {
            keep_going = keep_going && (event.key.keysym.sym != SDLK_ESCAPE);
            break;
        }
        }
    }

    SDL_SetRenderDrawColor(renderer, 16, 0, 16, 255);
    SDL_RenderClear(renderer);
    SDL_RenderGeometry(renderer, nullptr, origin_vertices.data(), origin_vertices.size(), nullptr, 0);
    SDL_RenderPresent(renderer);
}
```
使用 `SDL_GetTicks` 获取时间，定义计算三角形位置所需变量：
```
uint64_t last_tickets = SDL_GetTicks();
float position = 0.0f;
float direction = 1.0f;
```
在 `while` 循环中增加位置计算和修改顶点数据的代码：
```
while(keep_going) {
    ...

    uint64_t current_ticks = SDL_GetTicks();
    float delta_time = (current_ticks - last_tickets) / 1000.0f;
    last_tickets = current_ticks;
    position += 200.0f * delta_time * direction;

    int width = 0;
    SDL_GetRenderOutputSize(renderer, &width, nullptr);
    float max_position = static_cast<float>(width) - (origin_vertices[2].position.x - origin_vertices[1].position.x);

    if (position > max_position) {
        direction = -1.0f;
    } else if (position < 0.0f) {
        position = 0.0f;
        direction = 1.0f;
    }

    std::vector<SDL_Vertex> vertices;
    for (const auto& vertex : origin_vertices) {
        vertices.push_back(vertex);
        vertices.back().position.x += position;
    }

    ...
    SDL_RenderGeometry(renderer, nullptr, vertices.data(), vertices.size(), nullptr, 0);
    ...
}
```
这样我们就得到了一个在窗口上水平往复运动的三角形：

<img src=../img/sdl3_03_moving_triangle.png width="300"/>