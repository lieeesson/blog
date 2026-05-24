# C++ 跨平台 CI/CD 血泪史：CMake + Conan + GitHub Actions 的十二道陷阱

## 前言

说来惭愧，我原本只是想让一个小的 C++ 项目可以在 Windows 和 Linux 上面同时编译通过。这项目很简单：用 OpenCV 处理图像，用 Boost 做效能优化，C++17 标准，本来应该一天就能搞定的事，结果我折腾了整整两周。

这篇文章我要把整个过程中踩过的每一个坑都记录下来，包括错误信息、根本原因、以及最终的解法。希望后人不要重蹈我的覆辙。

---

## 第一阶段：vcpkg 的坑

一切从 Termux 说起。那时候我人在外面，手边只有一台 Android 手机，装了 Termux 环境。想先在行动设备上把项目跑起来，结果这个决定开启了为期两天的地狱。

### bootstrap-vcpkg.sh 的 TLS 错误

一开始我想用 vcpkg 来管理依赖。下了 vcpkg 源码，执行 `./bootstrap-vcpkg.sh`，然后就爆了：

```
curl#60 - SSL/TLS handshake failed
```

折腾半天才找到原因：Termux 的 `libcurl.so` 是用 **no TLS backend** 编译的，也就是说根本没有 SSL 支持，所以没办法跟 GitHub 服务器建立 HTTPS 连接。

### 从源码编译也没救

我想，那就直接从源码编译 vcpkg 好了，反正它也有离线模式。

```bash
# 尝试编译 vcpkg
git clone https://github.com/microsoft/vcpkg
./bootstrap-vcpkg.sh -disableMetrics
```

结果编译过程中冒出更多函数库链接的问题。折腾到一半我放弃了——在 Termux 上面折腾 vcpkg 完全是在浪费时间。

### 果敢放弃，改用 Conan

我意识到在 Linux 环境下，**Conan** 是更合理的选择。Conan 是专门做 C++ 依赖管理的工具，官方对 Linux 的支持也比 vcpkg 更好。最重要的是，Conan 2.x 已经正式 release，社群活跃文档也齐全。

> 这个阶段的教训：在非主流环境（Termux）上面做事，要先确认工具链的兼容性，否则就是在浪费时间。

---

## 第二阶段：Conan 入门——语法地狱

从 vcpkg 切换到 Conan 以为会很顺利，结果我立刻就撞墙了。

### Conan 2.x vs 1.x 语法差异

Conan 1.x 时代，指定 build type 要用冒号：

```bash
conan install . --settings:build_type=Release
```

升级到 Conan 2.x 之后，这种语法直接失效：

```
ERROR: Unknown argument --settings:build_type
```

正确姿势是**用空白分隔**：

```bash
conan install . --settings build_type=Release
```

或者更懒惰的做法：压根别加这个参数，因为 Conan 2.x **预设就是 Release**，完全不需要指定。

### 我的第一个 CI workflow 就这样爆了

那时候我写了第一版的 GitHub Actions workflow，大致上是这样的：

```yaml
- name: Install Conan packages
  run: |
    conan install . --settings:build_type=Release --build=missing
```

结果在 CI 上面疯狂报错，我还以为是网络问题、CI 环境问题，根本没想到是语法被改了。

> 这个阶段的教训：Conan 2.x 跟 1.x 的语法几乎是两套语言，升级之前要先确认文件用的是哪个版本。

---

## 第三阶段：opencv/4.9.0 的 Android API bug

在本地端折腾得差不多的时候，我想试试新版的 opencv，结果又踩到一个非常隐晦的 bug。

### ValueError: invalid literal for int() with base 10: 'None'

当时我用的是 `opencv/4.9.0`，在 Termux（Android）环境下执行 `conan install` 直接喷这个错误：

```
ERROR: conans.errors.ConanException: ValueError: invalid literal for int() with base 10: 'None'
```

错误信息完全看不懂，Stack Overflow 也找不到。只好去看 Conan Center 的源码。

### 根本原因

问题出在 opencv 的 `conanfile.py` 里面的 `configure()` 函数：

```python
def configure(self):
    super().configure()
    # 这行有问题：
    self.settings.os.api_level = int(self.settings.os.api_level)
```

当执行环境是 Termux/Android 时，`self.settings.os.api_level` 的值是 **`None`**（因为 Termux 不像真正的 Android 有 API level），所以 `int(None)` 直接喷 ValueError。

### 解法：降级到 opencv/4.5.5

既然 opencv/4.9.0 有这个 bug，我果敢降级：

```
opencv/4.5.5
```

这个版本没有这个问题，而且功能也足够用。4.5.5 是 2022 年发布的稳定版本，一直到 2025 年还有人在用，社群也稳定。

> 这个阶段的教训：不要盲目追求最新版本。新版本可能隐藏了 regression bug，稳定版才是 production 的好朋友。

---

## 第四阶段：CMake 找不到 OpenCV——CMAKE_PREFIX_PATH 之谜

好不容易把依赖装好了，结果 CMake 编译的时候跟我说找不到 OpenCV。

### find_package 失败

```bash
cmake --preset conan-release
```

输出：

```
CMake Error at CMakeLists.txt:12 (find_package):
  Could not find a package configuration file provided by "OpenCV" with any
  of the named targets above
```

`find_package(OpenCV REQUIRED)` 这个指令失效了。

### 为什么找不到？

折腾了半天我才搞懂：**Conan 2.x 的 CMakeDeps 产生的配置文件不会自动被 CMake 找到**。

Conan 2.x 的配置文件放在：

```
build/generators/
```

而 CMake 的 `find_package()` 预设只搜索：

- `/usr/local/lib/cmake/`
- `/usr/lib/cmake/`
- ...以及CMAKE_MODULE_PATH、CMAKE_PREFIX_PATH 指定的目录

所以根本找不到 `OpenCVConfig.cmake`。

### 解法：在 CMakeLists.txt 加入 CMAKE_PREFIX_PATH

```cmake
# 在 project() 之后加入
list(APPEND CMAKE_PREFIX_PATH "${CMAKE_CURRENT_BINARY_DIR}/generators")

find_package(OpenCV REQUIRED)
```

这样 CMake 就会去 `build/generators/` 寻找配置文件了。

> 这个阶段的教训：Conan 2.x 的 CMake 整合方式跟 1.x 完全不同，不能用旧经验套用新版本。

---

## 第五阶段：CMakeDeps 是必须加的！

找到配置文件的位置之后，我兴冲冲地再次执行 CMake，结果又爆了。

### 问题：`OpenCVConfig.cmake` 不存在

```
Could not find a package configuration file provided by "OpenCV" with any
of the named targets above
```

我明明已经加过 `CMAKE_PREFIX_PATH` 了，怎么还是找不到？

### 根本原因：CMakeToolchain 不会生成 find_package 配置文件

回头检查 `conanfile.txt`：

```ini
[generators]
CMakeToolchain
```

我只有 `CMakeToolchain`！这就是问题所在：

| Generator | 功能 |
|-----------|------|
| `CMakeToolchain` | 产生 toolchain 文件、编译器设定、编译选项 |
| `CMakeDeps` | 产生 `FindXXX.cmake` 或 `XXXConfig.cmake` 配置文件 |

**`CMakeToolchain` 只负责设定编译器环境，它不会产生任何 find_package 配置文件！**

### 解法：两个 generator 都要加

```ini
[generators]
CMakeToolchain
CMakeDeps
```

然后重新执行：

```bash
conan install . --output-folder=build --build=missing
```

这次 `OpenCVConfig.cmake` 终于出现了！

> 这个阶段的教训：Conan 2.x 的 generator 机制是模组化的，每个 generator 只做一件事。要设定编译环境需要 `CMakeToolchain`，要产生 find_package 文件需要 `CMakeDeps`，两者缺一不可。

---

## 第六阶段：GitHub Actions 缓存策略—— artifact 根本不靠谱

CI 终于可以在本地端编译了，于是我开始写 GitHub Actions workflow。这时候又遇到了新的问题：缓存策略。

### 最初的策略：上传 artifact

一开始我的想法是这样的：

```yaml
jobs:
  build-linux:
    steps:
      - name: Build
        run: |
          conan install . --output-folder=build
          cmake --preset conan-release
          cmake --build --preset conan-release
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-linux
          path: |
            build/
            CMakeUserPresets.json

  build-windows:
    needs: build-linux
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-linux
```

理论上这样可以把 Linux build 的产物下载下来给 Windows 用，节省编译时间。

### Artifact 的下载路径问题

结果 artifact 下载下来之后，路径完全对不上。GitHub Actions 的 artifact 下载会把文件放到一个 UUID 命名的目录里面，根本不是原来的 `build/` 路径结构。

折腾了半天，我放弃了 artifact 策略。

### 更好的方案：缓存 ~/.conan2

最终我改成缓存**整个 Conan 本地库**：

```yaml
- name: Cache Conan packages
  uses: actions/cache@v4
  with:
    path: |
      ~/.conan2
    key: conan2-${{ runner.os }}-${{ hashFiles('conanfile.txt') }}
    restore-keys: |
      conan2-${{ runner.os }}-
```

这样每一个 job 都可以共享同一个 Conan 缓存，无需手动搬运 artifact。

> 这个阶段的教训：GitHub Actions artifact 的设计是给一般产物用的，不适合用来共享编译缓存。直接缓存对应的目录（如 `~/.conan2`）才是正确做法。

---

## 第七阶段：Windows/Linux preset 名称统一问题

CI 跑起来之后，我发现另一个问题：Windows 和 Linux 的 preset 名称不一致。

### Windows 用 `conan-default`，Linux 用 `conan-release`

一开始我以为是正常的，因为 Windows 用 Visual Studio 而 Linux 用 Ninja，Conan 产生的 preset 名称可能不同。

结果 CMake configure 的时候喷错：

```
CMake Error: Could not find CMake preset file.
```

### 根本原因：单一配置 vs 多元配置生成器

问题在于 Conan 的 generator 设定：

- **Visual Studio** 是**多配置生成器**（multi-config），Conan 产生的 preset 叫 `conan-release`（只设定 Release）
- **Ninja** 是**单一配置生成器**（single-config），Conor 产生的 preset 也叫 `conan-release`

一开始 Windows workflow 用了错误的 preset 名称，导致 CMake 找不到文件。

### 解法：统一preset名称为 `conan-release`

修正 workflow：

```yaml
# Linux
- name: Configure CMake
  run: cmake --preset conan-release

# Windows
- name: Configure CMake
  run: cmake --preset conan-release
```

只要在 `conanfile.txt` 设定好 `default_options`，两边都会产生相同名称的 preset。

> 这个阶段的教训：Conan 2.x 搭配不同生成器时，preset 的命名规则要搞清楚。Visual Studio 多配置生成器只产生 Release 一种，Ninja 单一配置也是 Release，不能搞混。

---

## 第八阶段：PowerShell 不会报错——$ErrorActionPreference

CI 终于可以在两个平台都跑了，但我注意到一个诡异的问题：CMake configure 明明失败了，CI 却显示绿色通关。

### 问题：cmake --preset 失败了，但 PowerShell 继续执行

当时的 workflow 大致是这样的：

```yaml
- name: Configure CMake
  run: |
    cmake --preset conan-release
    cmake --build --preset conan-release
    ctest --preset conan-release
```

在 Windows 上面，如果 `cmake --preset` 失败了，PowerShell **预设会继续执行下一条指令**，根本不会中断！

结果就是：
- `cmake --preset conan-release` 失败了（找不到 preset）
- PowerShell 假装没事，继续执行 `cmake --build`
- 当然也编译不出东西，但 CI 就这样假装成功了

### 解法：设定 $ErrorActionPreference = "Stop"

在 PowerShell 脚本的开头加上：

```yaml
- name: Configure CMake
  shell: pwsh
  run: |
    $ErrorActionPreference = "Stop"
    cmake --preset conan-release
    cmake --build --preset conan-release
    ctest --preset conan-release
```

这样只要有任何一步失败，PowerShell 就会立即停止。

### 另外一个问题：ctest 报告 "No tests were found"

这个其实不是错误，只是我没有写任何测试。但当时因为前面的 cmake --build 根本没有真正编译到东西，所以 ctest 找不到测试案例。

加上 `$ErrorActionPreference = "Stop"` 之后，这个问题也跟着解决了（因为更早就失败了）。

> 这个阶段的教训：PowerShell 的错误处理机制跟 Bash 不同。Bash 预设会在命令失败时停止，而 PowerShell 要手动设定 `$ErrorActionPreference`。在 CI 脚本中千万不要忘记这件事。

---

## 第九阶段：cl not found——MSVC 开发者环境

CI 基本能跑了，但我又发现另一个问题：Windows 上面换成 Ninja 之后，编译器找不到。

### 问题：切换到 Ninja 之后，cl.exe 不见了

原本用 Visual Studio Generator 的时候，Conan 会自动帮我设定好 VS 开发者环境，`cl.exe` 在 PATH 里面。

但当我改成 `generator=Ninja` 之后，Conan 不再自动设定 VS 开发者环境，结果 `cl.exe` 根本不在 PATH 里面。

错误信息：

```
'cl' is not recognized as an internal or external command
```

### 解法：使用 ilammy/msvc-dev-cmd@v1

在 CMake 步骤之前加入 MSVC 开发者环境初始化：

```yaml
- name: Configure MSVC environment
  uses: ilammy/msvc-dev-cmd@v1
  with:
    toolset: 14.3
    arch: x64

- name: Configure CMake
  run: |
    cmake --preset conan-release
```

`msvc-dev-cmd` 会帮你把 VS 的开发者环境设定好，包括：
- `cl.exe` 的路径
- 必要的 include 目录
- 链接库路径

> 这个阶段的教训：使用 Ninja 生成器时，Conan 不会帮你设定 MSVC 开发环境。Windows 上面必须手动引入 VS Developer Environment，否则 compiler 完全找不到。

---

## 第十阶段：C++ 标准不一致——ABI 不兼容

所有问题都解决了，CI 终于绿了。但几天后我发现一个隐蔽的问题：组件之间的 ABI 不兼容。

### 问题：linking error 在执行期爆发

错误信息像是这样的：

```
undefined reference to 'cv::Mat::deallocate()'
```

这是很典型的 ABI 不兼容问题。

### 根本原因：CMAKE_CXX_STANDARD 11 vs C++17

回头看 `CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.15)
project(cv-boost-demo)

set(CMAKE_CXX_STANDARD 11)  # 这行是问题所在！
```

但是 OpenCV 4.5.5 是用 C++17 编译的。当你的代码用 C++11 编译，而链接到的函数库是用 C++17 编译，ABI 就不兼容。

### 解法：设定为 C++17

```cmake
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

`CMAKE_CXX_STANDARD_REQUIRED ON` 可以确保如果目标 compiler 不支持 C++17，CMake 会直接报错，而不是默默降级。

> 这个阶段的教训：C++ 标准版本必须跟依赖函数库保持一致。如果依赖是用 C++17 编译的，你的项目也必须用 C++17，否则迟早会遇到 ABI 不兼容的问题。

---

## 最终的设定档

经过两周的折腾，我的最终设定如下：

### conanfile.txt

```ini
[requires]
opencv/4.5.5
boost/1.87.0

[generators]
CMakeToolchain
CMakeDeps

[options]
opencv*:with_jpeg=True
opencv*:with_png=True
boost*:shared=False
boost*:header_only=False
```

### CMakeLists.txt（关键部分）

```cmake
cmake_minimum_required(VERSION 3.15)
project(cv-boost-demo)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 关键：加入 Conan 产生的配置文件路径
list(APPEND CMAKE_PREFIX_PATH "${CMAKE_CURRENT_BINARY_DIR}/generators")

find_package(OpenCV REQUIRED)
find_package(Boost REQUIRED COMPONENTS system filesystem)

add_executable(cv-boost-demo src/main.cpp)
target_link_libraries(cv-boost-demo
    OpenCV::opencv_imgcodecs
    OpenCV::opencv_core
    Boost::filesystem
    Boost::system
)
```

### GitHub Actions workflow

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        include:
          - os: ubuntu-latest
            shell: bash
          - os: windows-latest
            shell: pwsh

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Install Conan
        run: pip install conan

      - name: Configure Conan
        run: |
          conan profile detect --force
        shell: ${{ matrix.shell }}

      - name: Cache Conan packages
        uses: actions/cache@v4
        with:
          path: ~/.conan2
          key: conan2-${{ matrix.os }}-${{ hashFiles('conanfile.txt') }}
          restore-keys: |
            conan2-${{ matrix.os }}-

      - name: Install dependencies
        run: |
          conan install . --output-folder=build --build=missing
        shell: ${{ matrix.shell }}

      - name: Configure MSVC environment
        if: matrix.os == 'windows-latest'
        uses: ilammy/msvc-dev-cmd@v1
        with:
          toolset: 14.3
          arch: x64

      - name: Configure CMake
        run: |
          $ErrorActionPreference = "Stop"
          cmake --preset conan-release
        shell: ${{ matrix.shell }}

      - name: Build
        run: |
          $ErrorActionPreference = "Stop"
          cmake --build --preset conan-release
        shell: ${{ matrix.shell }}

      - name: Test
        run: |
          $ErrorActionPreference = "Stop"
          ctest --preset conan-release --output-on-failure
        shell: ${{ matrix.shell }}
```

---

## 总结：十二道陷阱

让我们回顾一下这十二道陷阱：

| 阶段 | 陷阱 | 解法 |
|------|------|------|
| 1 | vcpkg bootstrap TLS 错误 | 放弃 vcpkg，改用 Conan |
| 2 | Conan 2.x 语法改变 | `--settings:build_type` → `--settings build_type` |
| 3 | opencv/4.9.0 Android API bug | 降级到 opencv/4.5.5 |
| 4 | CMake 找不到 OpenCV | 加入 `CMAKE_PREFIX_PATH` |
| 5 | CMakeDeps 未设定 | `[generators]` 加入 `CMakeDeps` |
| 6 | Artifact 缓存策略失败 | 改为缓存 `~/.conan2` |
| 7 | Windows/Linux preset 名称不一致 | 统一使用 `conan-release` |
| 8 | PowerShell 不会在错误时停止 | 设定 `$ErrorActionPreference = "Stop"` |
| 9 | Ninja 生成器缺少 MSVC 环境 | 使用 `ilammy/msvc-dev-cmd@v1` |
| 10 | C++ 标准版本不一致 | `set(CMAKE_CXX_STANDARD 17)` |

### 最重要的三个 lesson

1. **不要用 Termux 折腾编译相关的事情** — 环境太特殊，问题会比收获多。

2. **Conan 2.x 跟 1.x 几乎是两套工具** — 网络上大部分范例都是 1.x 语法，要自己转换。

3. **CI 脚本要在本地测试** — 每次修改 workflow 都先在手动的跑一遍，确认逻辑正确再推到 GitHub。

希望这篇文章对你有帮助。如果你的 CI 也有类似的问题，欢迎留言讨论。

---

**相关连结：**
- 项目网址：https://github.com/lieeesson/cv-boost-demo
- Conan 官方文件：https://docs.conan.io/2/
- CMake 官方文件：https://cmake.org/documentation/