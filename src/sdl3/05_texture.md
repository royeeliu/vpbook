# SDL3 入门（5）：纹理渲染

## 创建纹理

有三个 API 可以用来创建纹理：
- `SDL_CreateTexture` 参数少，使用方便，适用于创建简单的纹理
- `SDL_CreateTextureFromSurface` 
适用于从已有图像数据创建纹理
- `SDL_CreateTextureWithProperties` 可以指定各种属性，功能强大，用起来也比较复杂，适用于另外两个 API 无法满足需求的情况

实际上前两个 API 内部都是通过调用 `SDL_CreateTextureWithProperties` 实现纹理创建的。这也是 SDL API 设计的特点，对于常用操作有简洁的 API 实现，同时也有使用复杂但是功能更灵活强大的 API 提供。

这里我们准备创建一个最小的四种颜色的纹理，像素尺寸 2x2，也就是总计只有 4 个像素。首先使用数组定义图像数据：
```
uint8_t pixels[4 * 2 * 2] = {
    0,   0,   255, 255,  // b, g, r, a
    0,   255, 0,   255,  //
    255, 0,   0,   255,  //
    0,   255, 255, 255   //
};
```
SDL 对像素格式的定义是按照从高位到低位的颜色命名的，所以上面的数据对应的格式是 `SDL_PIXELFORMAT_ARGB8888`。

由于已经有像素数据，所以我们可以从图像数据创建 Surface 然后调用 `SDL_CreateTextureFromSurface` 从 Surface 创建纹理：
```
SDL_Surface* surface = SDL_CreateSurfaceFrom(pixels, 2, 2, 4 * 2, SDL_PIXELFORMAT_ARGB8888);
if (!surface) {
    SDL_Log("Create surface failed: %s", SDL_GetError());
    return -1;
}

SDL_Texture* texture = SDL_CreateTextureFromSurface(renderer, surface);
if (!texture) {
    SDL_Log("Create texture failed: %s", SDL_GetError());
    return -1;
}
```
图像混合模式选择 None 也就是忽略透明通道，缩放模式选择临近点插值：
```
SDL_SetTextureBlendMode(texture, SDL_BLENDMODE_NONE);
SDL_SetTextureScaleMode(texture, SDL_SCALEMODE_NEAREST);
```
## 渲染纹理
渲染纹理使用的 API 是 `SDL_RenderTexture`，可以通过参数指定渲染纹理的矩形区域，也可以指定目标矩形区域。这里我们选择渲染整个纹理到目标中心附近宽高各为窗口一般的矩形区域：
```
// 计算目标矩形
int width = 0;
int height = 0;
SDL_GetRenderOutputSize(renderer, &width, &height);
SDL_FRect dst_rect{};
dst_rect.w = width * 0.5f;
dst_rect.h = height * 0.5f;
dst_rect.x = (width - dst_rect.w) * 0.5f;
dst_rect.y = (height - dst_rect.h) * 0.5f;

// 渲染
SDL_SetRenderDrawColor(renderer, 16, 0, 16, 255);
SDL_RenderClear(renderer);
SDL_RenderTexture(renderer, texture, nullptr, &dst_rect);
SDL_RenderPresent(renderer);
```
渲染效果如下：

<img src=../img/sdl3_05_texture_nearest.png width="300"/>

图中每种颜色对应的是最初的 2x2 图像中的一个像素，因为前面我们选择的缩放模式是临近点插值。实际图像处理中使用更多的是双线性插值，我们可以修改前面的代码看下效果：
```
SDL_SetTextureScaleMode(texture, SDL_SCALEMODE_LINEAR);
```
渲染效果：

<img src=../img/sdl3_05_texture_linear.png width="300"/>

## 纹理格式

可以使用如下代码查询当前渲染器支持的纹理格式：
```
void PrintSupportedTextureFormats(SDL_Renderer* renderer)
{
    SDL_PixelFormatEnum* texture_format = static_cast<SDL_PixelFormatEnum*>(SDL_GetProperty(
        SDL_GetRendererProperties(renderer), SDL_PROP_RENDERER_TEXTURE_FORMATS_POINTER, nullptr));
    int index = 0;
    while (texture_format && *texture_format != SDL_PIXELFORMAT_UNKNOWN) {
        SDL_Log("Texture format[%d]: %s", index, SDL_GetPixelFormatName(*texture_format));
        ++texture_format;
        ++index;
    }
}
```

使用不同的图形引擎创建渲染器时支持的纹理格式也不相同，下面是在一台 Windows 11 系统笔记本上的测试结果：
| Format |direct3d11|direct3d12|direct3d(9)|opengl|opengles2|vulkan|software|
| -- | -- | -- | -- | -- | -- | -- | -- |
| SDL_PIXELFORMAT_ARGB8888 | Y | Y | Y | Y | Y | Y | Y |
| SDL_PIXELFORMAT_ABGR8888 | - | - | - | Y | Y | - | - |
| SDL_PIXELFORMAT_XRGB8888 | Y | Y | - | Y | Y | Y | Y |
| SDL_PIXELFORMAT_XBGR8888 | - | - | - | Y | Y | - | - |
| SDL_PIXELFORMAT_XBGR2101010 | Y | Y | - | - | - | Y | - |
| SDL_PIXELFORMAT_RGBA64_FLOAT | Y | Y | - | - | - | Y | - |
| SDL_PIXELFORMAT_YV12 | Y | Y | Y | Y | Y | Y | - |
| SDL_PIXELFORMAT_IYUV | Y | Y | Y | Y | Y | Y | - |
| SDL_PIXELFORMAT_NV12 | Y | Y | - | Y | Y | Y | - |
| SDL_PIXELFORMAT_NV21 | Y | Y | - | Y | Y | Y | - |
| SDL_PIXELFORMAT_P010 | Y | Y | - | - | - | Y | - |

可以看到 `SDL_PIXELFORMAT_ARGB8888` 格式被包括软件实现在内的所有图形引擎支持，这正是我们前面选择使用该格式创建纹理的原因，可以方便的选择各个图形引擎进行测试。