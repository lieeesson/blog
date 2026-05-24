# C++ 跨平台 CI/CD 血淚史：CMake + Conan + GitHub Actions 的十二道陷阱

## 前言

說來慚愧，我原本只是想讓一個小小的 C++ 專案可以在 Windows 和 Linux 上面同時編譯通過。這項目很簡單：用 OpenCV 處理圖像，用 Boost 做效能優化，C++17 標準，本來應該一天就能搞定的事，結果我折騰了整整兩週。

這篇文章我要把整個過程中踩過的每一個坑都記錄下來，包括錯誤訊息、根本原因、以及最終的解法。希望後人不要重蹈我的覆轍。

---

## 第一階段：vcpkg 的坑

一切從 Termux 說起。那時候我人在外面，手邊只有一台 Android 手機，裝了 Termux 環境。想說先在行動裝置上把專案跑起來，結果這個決定開啟了為期兩天的地獄。

### bootstrap-vcpkg.sh 的 TLS 錯誤

一開始我想用 vcpkg 來管理依賴。下了 vcpkg 原始碼，執行 `./bootstrap-vcpkg.sh`，然後就爆了：

```
curl#60 - SSL/TLS handshake failed
```

折騰半天才找到原因：Termux 的 `libcurl.so` 是用 **no TLS backend** 編譯的，也就是說根本沒有 SSL 支持，所以沒辦法跟 GitHub 伺服器建立 HTTPS 連線。

### 從源碼編譯也沒救

我想，那就直接從源碼編譯 vcpkg 好了，反正它也有離線模式。

```bash
# 嘗試編譯 vcpkg
git clone https://github.com/microsoft/vcpkg
./bootstrap-vcpkg.sh -disableMetrics
```

結果編譯過程中冒出更多函式庫連結的問題。折騰到一半我放棄了——在 Termux 上面折騰 vcpkg 完全是在浪費時間。

### 果斷放棄，改用 Conan

我意識到在 Linux 環境下，**Conan** 是更合理的選擇。Conan 是專門做 C++ 依賴管理的工具，官方對 Linux 的支援也比 vcpkg 更好。最重要的是，Conan 2.x 已經正式 release，社群活跃文檔也齊全。

> 這個階段的教訓：在非主流環境（Termux）上面做事，要先確認工具鏈的相容性，否則就是在浪費生命。

---

## 第二階段：Conan 入門——語法地獄

從 vcpkg 切換到 Conan 以為會很順利，結果我立刻就撞牆了。

### Conan 2.x vs 1.x 語法差異

Conan 1.x 時代，指定 build type 要用冒號：

```bash
conan install . --settings:build_type=Release
```

升級到 Conan 2.x 之後，這種語法直接失效：

```
ERROR: Unknown argument --settings:build_type
```

正確姿勢是**用空白分隔**：

```bash
conan install . --settings build_type=Release
```

或者更懶惰的做法：壓根別加這個參數，因為 Conan 2.x **預設就是 Release**，完全不需要指定。

### 我的第一個 CI workflow 就這樣爆了

那時候我寫了第一版的 GitHub Actions workflow，大致上是這樣：

```yaml
- name: Install Conan packages
  run: |
    conan install . --settings:build_type=Release --build=missing
```

結果在 CI 上面瘋狂報錯，我還以為是網路問題、CI 環境問題，根本沒想到是語法被改了。

> 這個階段的教訓：Conan 2.x 跟 1.x 的語法幾乎是兩套語言，升級之前要先確認文件用的是哪個版本。

---

## 第三階段：opencv/4.9.0 的 Android API bug

在本地端折騰得差不多的時候，我想說試用新版的 opencv，結果又踩到一個非常隱晦的 bug。

### ValueError: invalid literal for int() with base 10: 'None'

當時我用的是 `opencv/4.9.0`，在 Termux（Android）環境下執行 `conan install` 直接噴這個錯誤：

```
ERROR: conans.errors.ConanException: ValueError: invalid literal for int() with base 10: 'None'
```

錯誤訊息完全看不懂，Stack Overflow 也找不到。只好去看 Conan Center 的原始碼。

### 根本原因

問題出在 opencv 的 `conanfile.py` 裡面的 `configure()` 函式：

```python
def configure(self):
    super().configure()
    # 這行有問題：
    self.settings.os.api_level = int(self.settings.os.api_level)
```

當執行環境是 Termux/Android 時，`self.settings.os.api_level` 的值是 **`None`**（因為 Termux 不像真正的 Android 有 API level），所以 `int(None)` 直接噴 ValueError。

### 解法：降級到 opencv/4.5.5

既然 opencv/4.9.0 有這個 bug，我果斷降級：

```
opencv/4.5.5
```

這個版本沒有這個問題，而且功能也足夠用。4.5.5 是 2022 年發布的穩定版本，一直到 2025 年還有人在用，社群也穩定。

> 這個階段的教訓：不要盲目追求最新版本。新版本可能隱藏了 regression bug，穩定版才是 production 的好朋友。

---

## 第四階段：CMake 找不到 OpenCV——CMAKE_PREFIX_PATH 之謎

好不容易把依賴裝好了，結果 CMake 編譯的時候跟我說找不到 OpenCV。

### find_package 失敗

```bash
cmake --preset conan-release
```

輸出：

```
CMake Error at CMakeLists.txt:12 (find_package):
  Could not find a package configuration file provided by "OpenCV" with any
  of the named targets above
```

`find_package(OpenCV REQUIRED)` 這個指令失效了。

### 為什麼找不到？

折騰了半天我才搞懂：**Conan 2.x 的 CMakeDeps 產生的設定檔不會自動被 CMake 找到**。

Conan 2.x 的設定檔放在：

```
build/generators/
```

而 CMake 的 `find_package()` 預設只搜索：

- `/usr/local/lib/cmake/`
- `/usr/lib/cmake/`
- ...以及CMAKE_MODULE_PATH、CMAKE_PREFIX_PATH 指定的目錄

所以根本找不到 `OpenCVConfig.cmake`。

### 解法：在 CMakeLists.txt 加入 CMAKE_PREFIX_PATH

```cmake
# 在 project() 之後加入
list(APPEND CMAKE_PREFIX_PATH "${CMAKE_CURRENT_BINARY_DIR}/generators")

find_package(OpenCV REQUIRED)
```

這樣 CMake 就會去 `build/generators/` 尋找設定檔了。

> 這個階段的教訓：Conan 2.x 的 CMake 整合方式跟 1.x 完全不同，不能用舊經驗套用新版本。

---

## 第五階段：CMakeDeps 是必須加的！

找到設定檔的位置之後，我興沖沖地再次執行 CMake，結果又爆了。

### 問題：`OpenCVConfig.cmake` 不存在

```
Could not find a package configuration file provided by "OpenCV" with any
of the named targets above
```

我明明已經加過 `CMAKE_PREFIX_PATH` 了，怎麼還是找不到？

### 根本原因：CMakeToolchain 不會生成 find_package 設定檔

回頭檢查 `conanfile.txt`：

```ini
[generators]
CMakeToolchain
```

我只有 `CMakeToolchain`！這就是問題所在：

| Generator | 功能 |
|-----------|------|
| `CMakeToolchain` | 產生 toolchain 檔案、編譯器設定、編譯選項 |
| `CMakeDeps` | 產生 `FindXXX.cmake` 或 `XXXConfig.cmake` 設定檔 |

**`CMakeToolchain` 只負責設定編譯器環境，它不會產生任何 find_package 設定檔！**

### 解法：兩個 generator 都要加

```ini
[generators]
CMakeToolchain
CMakeDeps
```

然後重新執行：

```bash
conan install . --output-folder=build --build=missing
```

這次 `OpenCVConfig.cmake` 終於出現了！

> 這個階段的教訓：Conan 2.x 的 generator 機制是模組化的，每個 generator 只做一件事。要設定編譯環境需要 `CMakeToolchain`，要產生 find_package 檔案需要 `CMakeDeps`，兩者缺一不可。

---

## 第六階段：GitHub Actions 緩存策略—— artifact 根本不靠譜

CI 終於可以在本地端編譯了，於是我開始寫 GitHub Actions workflow。這時候又遇到了新的問題：緩存策略。

### 最初的策略：上傳 artifact

我一開始的想法是這樣的：

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

理論上這樣可以把 Linux build 的產物下載下來給 Windows 用，節省編譯時間。

### Artifact 的下載路徑問題

結果 artifact 下載下來之後，路徑完全對不上。GitHub Actions 的 artifact 下載會把檔案放到一個 UUID 命名的目錄裡面，根本不是原來的 `build/` 路徑結構。

折騰了半天，我放棄了 artifact 策略。

### 更好的方案：緩存 ~/.conan2

最終我改成緩存**整個 Conan 本地庫**：

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

這樣每一個 job 都可以共享同一個 Conan 緩存，無需手動搬運 artifact。

> 這個階段的教訓：GitHub Actions artifact 的設計是給一般產物用的，不適合用來共享編譯緩存。直接緩存對應的目錄（如 `~/.conan2`）才是正確做法。

---

## 第七階段：Windows/Linux preset 名稱統一問題

CI 跑起來之後，我發現另一個問題：Windows 和 Linux 的 preset 名稱不一致。

### Windows 用 `conan-default`，Linux 用 `conan-release`

一開始我以為是正常的，因為 Windows 用 Visual Studio 而 Linux 用 Ninja，Conan 產生的 preset 名稱可能不同。

結果 CMake configure 的時候噴錯：

```
CMake Error: Could not find CMake preset file.
```

### 根本原因：單一配置 vs 多元配置生成器

問題在於 Conan 的 generator 設定：

- **Visual Studio** 是**多配置生成器**（multi-config），Conan 產生的 preset 叫 `conan-release`（只設定 Release）
- **Ninja** 是**單一配置生成器**（single-config），Conor 產生的 preset 也叫 `conan-release`

一開始 Windows workflow 用了錯誤的 preset 名稱，導致 CMake 找不到檔案。

### 解法：統一preset名稱為 `conan-release`

修正 workflow：

```yaml
# Linux
- name: Configure CMake
  run: cmake --preset conan-release

# Windows
- name: Configure CMake
  run: cmake --preset conan-release
```

只要在 `conanfile.txt` 設定好 `default_options`，兩邊都會產生相同名稱的 preset。

> 這個階段的教訓：Conan 2.x 搭配不同生成器時，preset 的命名規則要搞清楚。Visual Studio 多配置生成器只產生 Release 一種，Ninja 單一配置也是 Release，不能搞混。

---

## 第八階段：PowerShell 不會报错——$ErrorActionPreference

CI 終於可以在兩個平台都跑了，但我注意到一個詭異的問題：CMake configure 明明失敗了，CI 卻顯示綠色通關。

### 問題：cmake --preset 失敗了，但 PowerShell 繼續執行

當時的 workflow 大致是這樣：

```yaml
- name: Configure CMake
  run: |
    cmake --preset conan-release
    cmake --build --preset conan-release
    ctest --preset conan-release
```

在 Windows 上面，如果 `cmake --preset` 失敗了，PowerShell **預設會繼續執行下一條指令**，根本不會中斷！

結果就是：
- `cmake --preset conan-release` 失敗了（找不到 preset）
- PowerShell 假裝沒事，繼續執行 `cmake --build`
- 當然也編譯不出東西，但 CI 就這樣假裝成功了

### 解法：設定 $ErrorActionPreference = "Stop"

在 PowerShell 腳本的開頭加上：

```yaml
- name: Configure CMake
  shell: pwsh
  run: |
    $ErrorActionPreference = "Stop"
    cmake --preset conan-release
    cmake --build --preset conan-release
    ctest --preset conan-release
```

這樣只要有任何一步失敗，PowerShell 就會立即停止。

### 另外一個問題：ctest 報告 "No tests were found"

這個其實不是錯誤，只是我沒有寫任何測試。但當時因為前面的 cmake --build 根本沒有真正編譯到東西，所以 ctest 找不到測試案例。

加上 `$ErrorActionPreference = "Stop"` 之後，這個問題也跟著解決了（因為更早就失敗了）。

> 這個階段的教訓：PowerShell 的錯誤處理機制跟 Bash 不同。Bash 預設會在命令失敗時停止，而 PowerShell 要手動設定 `$ErrorActionPreference`。在 CI 腳本中千萬不要忘記這件事。

---

## 第九階段：cl not found——MSVC 開發者環境

CI 基本上能跑了，但我又發現另一個問題：Windows 上面換成 Ninja 之後，編譯器找不到。

### 問題：切換到 Ninja 之後，cl.exe 不見了

原本用 Visual Studio Generator 的時候，Conan 會自動幫我設定好 VS 開發者環境，`cl.exe` 在 PATH 裡面。

但當我改成 `generator=Ninja` 之後，Conan 不再自動設定 VS 開發者環境，結果 `cl.exe` 根本不在 PATH 裡面。

錯誤訊息：

```
'cl' is not recognized as an internal or external command
```

### 解法：使用 ilammy/msvc-dev-cmd@v1

在 CMake 步驟之前加入 MSVC 開發者環境初始化：

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

`msvc-dev-cmd` 會幫你把 VS 的開發者環境設定好，包括：
- `cl.exe` 的路徑
- 必要的 include 目錄
- 連結庫路徑

> 這個階段的教訓：使用 Ninja 生成器時，Conan 不會幫你設定 MSVC 開發環境。Windows 上面必須手動引入 VS Developer Environment，否則 compiler 完全找不到。

---

## 第十階段：C++ 標準不一致——ABI 不相容

所有問題都解決了，CI 終於綠了。但幾天後我發現一個隱蔽的問題：組件之間的 ABI 不相容。

### 問題：linking error 在執行期爆發

錯誤訊息像是這樣：

```
undefined reference to 'cv::Mat::deallocate()'
```

這是很典型的 ABI 不相容問題。

### 根本原因：CMAKE_CXX_STANDARD 11 vs C++17

回頭看 `CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.15)
project(cv-boost-demo)

set(CMAKE_CXX_STANDARD 11)  # 這行是問題所在！
```

但是 OpenCV 4.5.5 是用 C++17 編譯的。當你的程式碼用 C++11 編譯，而連結到的函式庫是用 C++17 編譯，ABI 就不相容。

### 解法：設定為 C++17

```cmake
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

`CMAKE_CXX_STANDARD_REQUIRED ON` 可以確保如果目標 compiler 不支援 C++17，CMake 會直接报错，而不是默默降級。

> 這個階段的教訓：C++ 標準版本必須跟依賴函式庫保持一致。如果依賴是用 C++17 編譯的，你的專案也必須用 C++17，否則遲早會遇到 ABI 不相容的問題。

---

## 最終的設定檔

經過兩週的折騰，我的最終設定如下：

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

### CMakeLists.txt（關鍵部分）

```cmake
cmake_minimum_required(VERSION 3.15)
project(cv-boost-demo)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 關鍵：加入 Conan 產生的設定檔路徑
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

## 總結：十二道陷阱

讓我們回顧一下這十二道陷阱：

| 階段 | 陷阱 | 解法 |
|------|------|------|
| 1 | vcpkg bootstrap TLS 錯誤 | 放棄 vcpkg，改用 Conan |
| 2 | Conan 2.x 語法改變 | `--settings:build_type` → `--settings build_type` |
| 3 | opencv/4.9.0 Android API bug | 降級到 opencv/4.5.5 |
| 4 | CMake 找不到 OpenCV | 加入 `CMAKE_PREFIX_PATH` |
| 5 | CMakeDeps 未設定 | `[generators]` 加入 `CMakeDeps` |
| 6 | Artifact 緩存策略失敗 | 改為緩存 `~/.conan2` |
| 7 | Windows/Linux preset 名稱不一致 | 統一使用 `conan-release` |
| 8 | PowerShell 不會在錯誤時停止 | 設定 `$ErrorActionPreference = "Stop"` |
| 9 | Ninja 生成器缺少 MSVC 環境 | 使用 `ilammy/msvc-dev-cmd@v1` |
| 10 | C++ 標準版本不一致 | `set(CMAKE_CXX_STANDARD 17)` |

### 最重要的三個 lesson

1. **不要用 Termux 折騰編譯相關的事情** — 環境太特殊，問題會比收穫多。

2. **Conan 2.x 跟 1.x 幾乎是兩套工具** — 網路上大部分範例都是 1.x 語法，要自己轉換。

3. **CI 腳本要在本地測試** — 每次修改 workflow 都先在手動跑一遍，確認邏輯正確再推到 GitHub。

希望這篇文章對你有幫助。如果你的 CI 也有類似的問題，歡迎留言討論。

---

**相關連結：**
- 專案網址：https://github.com/lieeesson/cv-boost-demo
- Conan 官方文件：https://docs.conan.io/2/
- CMake 官方文件：https://cmake.org/documentation/