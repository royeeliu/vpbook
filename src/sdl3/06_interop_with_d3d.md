# SDL3 入门（6）：和外部 D3D 交互

## 获取 D3D 设备

前面讲过在创建渲染器时可以指定图像引擎，例如在 Windows 平台下可以指定使用 D3D11：
```
SDL_Renderer* renderer = SDL_CreateRenderer(window, "direct3d11");
```
创建渲染器成功后可以通过 `SDL_GetRendererProperties` 取出内部的 D3D 设备接口：
```
ComPtr<ID3D11Device> d3d11_device = static_cast<ID3D11Device*>(SDL_GetProperty(
        SDL_GetRendererProperties(renderer), SDL_PROP_RENDERER_D3D11_DEVICE_POINTER, nullptr));
```
遗憾的是我们无法在外部创建 D3D 设备然后传递给 SDL 用来创建渲染器，也没有更多的 D3D 创建参数可以设置，这导致一些特殊需求无法实现。比如说系统有两张显卡时无法自由的选择使用哪张显卡而只能使用默认的。不过无论如何，SDL3 能指定图形引擎和取出 D3D 设备已经是一个巨大的进步。

更重要的一个特性是我们可以使用类似的方式从 SDL 纹理中取出原始的 D3D 纹理或者把已有的 D3D 纹理包装成 SDL 纹理然后交给 SDL 渲染器执行渲染。这个特性非常重要，因为这意味着我们可以丝滑的把已有的 D3D 渲染代码和 SDL 结合起来使用，在 SDL2 里这是做不到的。

接下来我们用一个简单的例子来做演示。

## 渲染 D3D 纹理

首先使用前面拿到的 `d3d11_device` 创建一个 2D 纹理并填充图像数据，依然是 2x2 大小，`SDL_PIXELFORMAT_ARGB8888` 格式，对应 D3D 中的 `DXGI_FORMAT_B8G8R8A8_UNORM` 格式:
```
ComPtr<ID3D11Texture2D> d3d11_texture = CreateTexture2D(d3d11_device.Get(), 2, 2);
if (!d3d11_texture) {
    SDL_Log("Create D3D11 texture failed");
    return -1;
}

uint8_t pixels[4 * 2 * 2] = {
    0,   0,   255, 255,  // b, g, r, a
    0,   255, 0,   255,  //
    255, 0,   0,   255,  //
    0,   255, 255, 255   //
};

d3d11_context->UpdateSubresource(d3d11_texture.Get(), 0, nullptr, pixels, 4 * 2, 0);
```

其中 `CreateTexture2D` 函数定义如下：
```
ComPtr<ID3D11Texture2D> CreateTexture2D(ID3D11Device* d3d11_device, int width, int height)
{
    D3D11_TEXTURE2D_DESC desc{};
    desc.Width = width;
    desc.Height = height;
    desc.MipLevels = 1;
    desc.ArraySize = 1;
    desc.Format = DXGI_FORMAT_B8G8R8A8_UNORM;
    desc.SampleDesc.Count = 1;
    desc.Usage = D3D11_USAGE_DEFAULT;
    desc.BindFlags = D3D11_BIND_SHADER_RESOURCE;
    desc.CPUAccessFlags = 0;
    desc.MiscFlags = 0;

    ComPtr<ID3D11Texture2D> texture;
    HRESULT hr = d3d11_device->CreateTexture2D(&desc, nullptr, &texture);
    if (FAILED(hr)) {
        SDL_Log("Create texture failed: 0x%08X", hr);
        return nullptr;
    }

    return texture;
}
```

接下来是最重要的一步，将 D3D 纹理包装为 SDL 纹理：
```
SDL_PropertiesID props = SDL_CreateProperties();
SDL_SetNumberProperty(props, SDL_PROP_TEXTURE_CREATE_WIDTH_NUMBER, 2);
SDL_SetNumberProperty(props, SDL_PROP_TEXTURE_CREATE_HEIGHT_NUMBER, 2);
SDL_SetNumberProperty(props, SDL_PROP_TEXTURE_CREATE_FORMAT_NUMBER, SDL_PIXELFORMAT_ARGB8888);
SDL_SetProperty(props, SDL_PROP_TEXTURE_CREATE_D3D11_TEXTURE_POINTER, d3d11_texture.Get());
SDL_Texture* texture = SDL_CreateTextureWithProperties(renderer, props);
SDL_DestroyProperties(props);

if (!texture) {
    SDL_Log("Create texture failed: %s", SDL_GetError());
    return -1;
}
```
设置采样模式并执行渲染，渲染代码和之前的纹理渲染代码完全相同，这里不再重复：
```
SDL_SetTextureBlendMode(texture, SDL_BLENDMODE_NONE);
SDL_SetTextureScaleMode(texture, SDL_SCALEMODE_NEAREST);
```

渲染效果和前面纯 SDL 渲染的纹理一模一样：

<img src=../img/sdl3_06_d3d_texture.png width="300"/>


至此，对 SDL3 的技术调研就暂告一段落了。我们的目的是找一个轻量简单的跨平台窗口系统，用于实现视频播放演示程序的输出窗口。已经考察的特性说明 SDL3 是胜任的，无论软解解码还是硬件解码的图像数据（在 Windows 平台下解码器输出的是 D3D 纹理）都可以在简单处理后直接渲染到 SDL 窗口基本没有额外的性能损失。

实际上视频输出应该是 SDL3 设计的目标之一，所以 SDL3 源码中自带了一个基于 FFmpeg 的示例程序，使用 FFmpeg 实现音视频解码，然后使用 SDL 渲染输出。其中对 HDR、颜色空间等视频高级特性都做了处理，所以可以放心使用，基本不用担心后续某些特殊需求无法满足（除了显卡选择😂），实在不行还可以修改 SDL 源码。
