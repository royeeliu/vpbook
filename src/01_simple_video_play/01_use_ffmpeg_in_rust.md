# 在 Rust 中使用 FFmpeg (1)：环境搭建

写播放器最快的方式是用各个操作系统平台提供的多媒体组件，一般十几行几十行代码就能搞出一个勉强可用的播放器来，但是要做更深入更强大的功能就难了，如果要做跨平台那更不可能，这时候就需要用到 FFmpeg 了，Rust 也不例外。接下来就让我们的自虐之路从 FFmpeg 开始。

[creates.io](https://crates.io/) 上提供 FFmpeg 接口绑定或者包装的 crate 有不少，其中人气最高的应该是 [ffmpeg-sys-next](https://crates.io/crates/ffmpeg-sys-next)，该项目提供了对 FFmpeg 接口直接绑定。

另外一个 crate [ffmpeg-next](https://crates.io/crates/ffmpeg-next) 是在 ffmpeg-sys-next 基础上做了一定的抽象和包装，不必和底层的 unsafe C 接口打交道，使用更加方便。

## 环境搭建
Rust 开发环境搭建可以参考[《Rust语言圣经(Rust Course)》](https://course.rs/first-try/installation.html)，这里不再赘述。

搭建好开发环境后在终端依次执行下列命令创建项目：
```
md testffmpeg
cd ./testffmpeg
cargo init .
```
可以编译运行一次确认环境正常
```
cargo run
``` 
一切正常的话输出如下：
```
Hello, world!
```

依次执行以下命令添加对 ffmpeg-next 的依赖，并再次编译运行
```
cargo add ffmpeg-next
cargo run
```
不出意外的话出现了意外，编译报错了，报错信息大概长下面这个样子：
```
   Compiling ffmpeg-sys-next v7.0.2
error: failed to run custom build command for `ffmpeg-sys-next v7.0.2`

Caused by:
  process didn't exit successfully: `C:\Users\User\Develop\testffmpeg\target\debug\build\ffmpeg-sys-next-00f4c3eae2634634\build-script-build` (exit code: 101)
  --- stdout
  Could not find ffmpeg with vcpkg: Could not find Vcpkg tree: No vcpkg installation found. Set the VCPKG_ROOT environment variable or run 'vcpkg integrate install'
  cargo:rerun-if-env-changed=LIBAVUTIL_NO_PKG_CONFIG
  cargo:rerun-if-env-changed=PKG_CONFIG_x86_64-pc-windows-msvc
  cargo:rerun-if-env-changed=PKG_CONFIG_x86_64_pc_windows_msvc
  ...
```
对 ffmpeg-sys-next 的依赖是由 ffmpeg-next 引入的，报错的正是这个 crate，报错原因是找不到 FFmpeg 库。
可以参考 ffmpeg-next 项目 wiki 上文档 [Notes on building](https://github.com/zmwangx/rust-ffmpeg/wiki/Notes-on-building) 安装 FFmpeg 环境。不过文档写的比较简略，下面是我使用的方法：

### Windows
从报错信息可知，ffmpeg-sys-next 在 Windows 系统下通过 vcpkg 这个包管理工具查找 ffmpeg，所以我们需要先安装 vcpkg，首先克隆 vcpkg 仓库：
```
git clone https://github.com/microsoft/vcpkg.git
```
注意选择合适的目录，接下来执行：
```
cd vcpkg
.\bootstrap-vcpkg.bat
```
记得将 vcpkg 目录添加到系统环境变量 Path 中，然后就可以用 vcpkg 安装 ffmpeg 了:
```
vcpkg install ffmpeg
```
安装过程如果提示找不到合适版本的 CMake 请先安装或升级 CMake，重启终端后重新执行上面的命令。

安装 ffmpeg 完毕后重新执行 `cargo run`，好吧，依然报错：
```
 ...
  --- stderr
  cl : Command line warning D9035 : option 'o' has been deprecated and will be removed in a future release
  thread 'main' panicked at C:\Users\User\.cargo\registry\src\index.crates.io-6f17d22bba15001f\bindgen-0.69.4\lib.rs:622:31:
  Unable to find libclang: "couldn't find any valid shared libraries matching: ['clang.dll', 'libclang.dll'], set the `LIBCLANG_PATH` environment variable to a path where one of these files can be found (invalid: [])"
  note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
这次是缺少了 clang 环境。到 [LLVM 官网](https://llvm.org/) 下载合适的安装包，安装过程选择 "Add LLVM to the system PATH for all users"：
<img src=../img/01_svp_01_llvm.png />
安装完毕后重启终端以便 PATH 生效，执行：
```
clang -v
```
执行成功说明 clang 环境已 OK，切换到我们测试项目的目录再次执行 `cargo run`：
<img src=../img/01_svp_01_win_done.png />
终于成功了。

### Linux(Ubuntu)
Linux 下搭建环境就简单多了，按照提示安装依赖即可：
```
sudo apt install libavutil-dev
sudo apt install libavformat-dev
sudo apt install libavdevice-dev
sudo apt install libavfilter-dev
```

## 验证代码
接下来写一些简单的代码验证对 FFmpeg 接口的调用是否能正确运行。
```
extern crate ffmpeg_next as ffmpeg;

fn main() {
    ffmpeg::init().unwrap();

    if let Some(codec) = ffmpeg::decoder::find_by_name("h264") {
        println!("Codec: {} ({})", codec.name(), codec.description());
    }
}
```
编译执行，输出如下，说明成功调用了 FFmpeg：
```
Codec: h264 (H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10)
```
环境搭建完成，后续我们就可以写一些真正有趣的代码了，比如视频解码。

## ffmpeg-sys-next

ffmpeg-next 是 FFmpeg 接口的一个高层包装，做了一定的抽象使其更符合 Rust 的使用习惯。不过如果我们本来就熟悉 FFmpeg 接口的话可能反而觉得不习惯，无法直接找到 FFmpeg 接口的对应包装，这时候就可以绕过 ffmpeg-next 直接使用 ffmpeg-sys-next，后者实际上就是 FFmpeg C 接口的 Rust 版本，可以和我们过往所熟悉的知识一一对应起来。

另外，FFmpeg 太庞大了，ffmpeg-next 的包装并不完整，只实现了常用的部分，后续随着使用程度的加深需要用到一些不太常用的特性时同样需要直接调用 ffmpeg-sys-next。

（未完待续）