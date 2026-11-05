

# Termux 環境建置與 Android 原生開發筆記

- **官方文件**：[Termux Getting Started](https://wiki.termux.com/wiki/Getting_started)         
- **原始碼**：[GitHub - termux/termux-app](https://github.com/termux/termux-app)  
從 [Releases 頁面](https://github.com/termux/termux-app/releases/tag/v0.118.3) 下載對應架構的 APK：
- **32 位元設備**：下載 `armeabi-v7a`    
- **64 位元設備**：下載 `arm64-v8a`

安裝並開啟後畫面如下：
![](https://i.meee.com.tw/VlDkzyJ.png)
> **核心觀念補充**
> - **NDK (Native Development Kit)**：Google 提供的工具集，讓開發者能在 Android 上直接使用 C/C++ 撰寫原生程式碼或建置二進位執行檔。
> - **JNI (Java Native Interface)**：Java/Kotlin 與 C/C++ 之間的橋接協定。若要被 Android Runtime (ART) 調用，通常會編譯成 `.so` 動態連結庫；若要在終端機獨立執行，則編譯為原生二進位檔（Executable）。
> - **ABI (Application Binary Interface)**：定義二進位檔如何與系統底層（如 CPU 指令集）溝通，常見有 `armeabi-v7a` (32-bit ARM) 與 `arm64-v8a` (64-bit ARM)。

## 多工終端：tmux 配置

```
pkg install tmux
tmux
```

- **水平分割**：`Ctrl + b` 後按 `Shift + %`      
- **垂直分割**：`Ctrl + b` 後按 `Shift + "`    

# Git 環境配置

%% col-start %% 
%% col-break:b:primary %%
```
# 更新套件庫並安裝 Git 與 GitHub CLI
pkg update
pkg install git gh openssh -y

# 登入 GitHub
gh auth login
```

%% col-break:b:primary %% 
![|300](https://i.meee.com.tw/cH6bTVH.png)
%% col-end %%


# 開發工具鏈安裝

```bash
# 賦予儲存空間存取權限
termux-setup-storage

# 安裝編譯工具與 Java 環境
pkg install clang make cmake git openjdk-17 aapt apksigner ecj -y
```

確認當前硬體與使用者空間架構：

```bash
uname -m
```

- `armv7l` 或 `armv8l`：執行於 **32-bit ARM** 模式。
- `aarch64`：執行於 **64-bit ARM** 模式。

# [實作] Hello Native / JNI 專案

### 目錄結構

Plaintext

```
project/
└── jni/
    ├── Android.mk   # NDK 建置腳本（64-bit 建議）
    ├── Makefile     # 傳統 make 腳本（32-bit 或輕量環境建議）
    └── main.c       # C 原始碼
```

### 原始碼與設定檔

%% col-start %% 
%% col-break:b:primary %% 
**main.c**
```c
#include <stdio.h>

int main() {
    printf("Hello from Android native executable!\n");
    return 0;
}
```

%% col-break:b:primary %%
**Android.mk**

```Makefile
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := my_executable
LOCAL_SRC_FILES := main.c

include $(BUILD_EXECUTABLE)
```

%% col-break:b:primary %%
**Makefile**

```Makefile
CC = clang
# -pie / -fPIE: 產生位置無關可執行檔（Android 5.0+ 強制要求）
CFLAGS = -O2 -pie -fPIE -Wall
TARGET = my_executable
SRC = main.c

all: $(TARGET)

$(TARGET): $(SRC)
	$(CC) $(CFLAGS) $(SRC) -o $(TARGET)
	chmod +x $(TARGET)

clean:
	rm -f $(TARGET)
```

%% col-end %%

  

# 編譯方式

### 方式 A：使用官方 Android NDK 編譯（適用 aarch64）

適合使用 `Android.mk` 進行模組化建置：

```Bash
cd ~
wget https://dl.google.com/android/repository/android-ndk-r30-linux.zip
unzip android-ndk-r30-linux.zip
rm android-ndk-r30-linux.zip

# 加入環境變數
echo 'export ANDROID_NDK_HOME=$HOME/android-ndk-r30-linux' >> ~/.bashrc
echo 'export PATH=$ANDROID_NDK_HOME:$PATH' >> ~/.bashrc
source ~/.bashrc
```

切換至 `project/` 目錄後執行：

```Bash
ndk-build
```

### 方式 B：使用 Makefile 與 ndk-multilib 編譯

> **名詞解析：ndk-multilib**
> Termux 官方倉庫提供的輕量級交叉編譯套件，內含不同架構（32-bit / 64-bit）的 Android C 函式庫標頭檔與目標庫，讓單一 Clang 編譯器能同時支援多種 ABI 目標，無需下載龐大的完整版 NDK。
> 

```Bash
# 安裝 multilib 與 Android 系統標頭庫
pkg install clang make ndk-multilib ndk-sysroot -y

# 進入目錄並編譯
cd ~/project/jni
make
./my_executable
```

### 方式 C：直接使用 Clang 命令行編譯

若為 32-bit 設備找不到預編譯 NDK，或僅需編譯單一原始碼時最直觀：

  

- **編譯 32-bit 可執行檔 (armeabi-v7a)**：
    
    ```bash
    clang -target armv7a-linux-androideabi29 -O2 -pie -fPIE main.c -o my_executable_arm32
    ```
    
- **編譯 64-bit 可執行檔 (arm64-v8a)**：    
    
    ```bash
    clang -target aarch64-linux-android29 -O2 -pie -fPIE main.c -o my_executable_arm64
    ```
    
- **編譯供 Android App 調用的 JNI 動態函式庫 (.so)**：    
          
    ```bash
    clang -target aarch64-linux-android29 -shared -fPIC main.c -o libnative.so
    ```
    

%% col-start %% 
%% col-break:b:primary %%
**執行流程** 

```Bash
cd ~/path/to/your/project/jni
clang -O2 -pie -fPIE main.c -o my_executable
chmod +x my_executable
./my_executable
```

%% col-break:b:primary %% 
**終端輸出** 
```Plaintext
~/hello_jni/jni $ ls -al
total 23
drwx------. 2 u0_a460 u0_a460 3452 Nov  1 11:04 .
drwx------. 3 u0_a460 u0_a460 3452 Nov  1 11:04 ..
-rw-------. 1 u0_a460 u0_a460  345 Nov  1 11:04 Android.mk
-rw-------. 1 u0_a460 u0_a460  104 Nov  1 11:04 main.c
-rwx------. 1 u0_a460 u0_a460 4396 Nov  1 11:04 my_executable
~/hello_jni/jni $ ./my_executable
Hello from Android native executable!
```

%% col-end %%

  

# 注意事項

- **檔案系統權限限制**：Android 系統將外部儲存裝置（如 `/sdcard/`、`/storage/emulated/0/`）掛載為 `noexec`，**無法在此路徑直接執行二進位程式**。專案請務必放在 Termux 的私有目錄下（`$HOME` 或 `~`）。
          
- 若專案已建立在外部儲存空間，請搬移至家目錄：        
    
    ```Bash
    cp -r ~/storage/shared/.../Projects/hello_jni $HOME/hello_jni
    cd $HOME/hello_jni/jni
    ```



---

# Termux Environment Setup & Android Native Development Notes

- **Official Documentation**: [Termux Getting Started](https://wiki.termux.com/wiki/Getting_started)
- **Source Code**: [GitHub - termux/termux-app](https://github.com/termux/termux-app)
    Download the APK corresponding to your architecture from the [Releases page](https://github.com/termux/termux-app/releases/tag/v0.118.3):
- **32-bit devices**: Download `armeabi-v7a`    
- **64-bit devices**: Download `arm64-v8a`

After installation and launch, the interface appears as follows:
![](https://i.meee.com.tw/VlDkzyJ.png)

> **Core Concepts**
> 
> - **NDK (Native Development Kit)**: A toolset provided by Google that enables developers to implement native code in C/C++ or build binary executables directly on Android.
>     
> - **JNI (Java Native Interface)**: A foreign function interface bridge between Java/Kotlin and C/C++. When invoked by the Android Runtime (ART), native code is typically built as a `.so` shared library; for standalone command-line execution, it is built as a native executable.
>     
> - **ABI (Application Binary Interface)**: Defines the binary interface contract with the underlying system and CPU instruction set; common architectures include `armeabi-v7a` (32-bit ARM) and `arm64-v8a` (64-bit ARM).

## Multiplexing Terminal: tmux Setup

```
pkg install tmux
tmux
```

- **Horizontal split**: Press `Ctrl + b`, then `Shift + %`
    
- **Vertical split**: Press `Ctrl + b`, then `Shift + "`      
    

# Git Environment Configuration

%% col-start %%
%% col-break:b:primary %%

```
# Update package repositories and install Git and GitHub CLI
pkg update
pkg install git gh openssh -y

# Log in to GitHub
gh auth login
```

%% col-break:b:primary %%
![|300](https://i.meee.com.tw/cH6bTVH.png)
%% col-end %%

  

# Development Toolchain Installation

```Bash
# Grant storage access permissions
termux-setup-storage

# Install build tools and Java environment
pkg install clang make cmake git openjdk-17 aapt apksigner ecj -y
```

Check the current hardware and user-space architecture:

```Bash
uname -m
```

- `armv7l` or `armv8l`: Running in **32-bit ARM** mode.    
- `aarch64`: Running in **64-bit ARM** mode.    

# [Hands-on] Hello Native / JNI Project

### Directory Structure  

```Plaintext
project/
└── jni/
    ├── Android.mk   # NDK build script (Recommended for 64-bit)
    ├── Makefile     # Standard makefile (Recommended for 32-bit or lightweight setups)
    └── main.c       # C source file
```

### Source Code and Configuration Files

%% col-start %% 
%% col-break:b:primary %% 
**main.c**

```c
#include <stdio.h>

int main() {
    printf("Hello from Android native executable!\n");
    return 0;
}
```

%% col-break:b:primary %% 
**Android.mk**

```makefile
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := my_executable
LOCAL_SRC_FILES := main.c

include $(BUILD_EXECUTABLE)
```

%% col-break:b:primary %%
**Makefile** 

```makefile
CC = clang
# -pie / -fPIE: Generates Position-Independent Executables (required by Android 5.0+)
CFLAGS = -O2 -pie -fPIE -Wall
TARGET = my_executable
SRC = main.c

all: $(TARGET)

$(TARGET): $(SRC)
	$(CC) $(CFLAGS) $(SRC) -o $(TARGET)
	chmod +x $(TARGET)

clean:
	rm -f $(TARGET)
```

%% col-end %%


# Compilation Methods

### Method A: Build with Official Android NDK (for aarch64)

Suitable for modular builds using `Android.mk`:


```bash
cd ~
wget https://dl.google.com/android/repository/android-ndk-r30-linux.zip
unzip android-ndk-r30-linux.zip
rm android-ndk-r30-linux.zip

# Add to environment variables
echo 'export ANDROID_NDK_HOME=$HOME/android-ndk-r30-linux' >> ~/.bashrc
echo 'export PATH=$ANDROID_NDK_HOME:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Navigate to the `project/` directory and execute:

  
```
ndk-build
```

### Method B: Build with Makefile and ndk-multilib

> **Terminology Note: ndk-multilib**
> 
> A lightweight cross-compilation package provided in the official Termux repository. It contains Android C library headers and target libraries for multiple architectures (32-bit / 64-bit), allowing a single Clang compiler to target various ABIs without requiring the full, bulky NDK package.
> 
>   

```
# Install multilib and Android system sysroot headers
pkg install clang make ndk-multilib ndk-sysroot -y

# Navigate to directory and build
cd ~/project/jni
make
./my_executable
```

### Method C: Compile Directly with Clang CLI

Most straightforward when prebuilt NDK binaries are unavailable for 32-bit devices, or when compiling a single source file:

- **Compile 32-bit executable (armeabi-v7a)**:
    
    ```Bash
    clang -target armv7a-linux-androideabi29 -O2 -pie -fPIE main.c -o my_executable_arm32
    ```
    
- **Compile 64-bit executable (arm64-v8a)**:
    
    ```Bash
    clang -target aarch64-linux-android29 -O2 -pie -fPIE main.c -o my_executable_arm64
    ```
    
- **Compile JNI shared library (.so) for Android App integration**:
    
    ```Bash
    clang -target aarch64-linux-android29 -shared -fPIC main.c -o libnative.so
    ```
    

%% col-start %% 
%% col-break:b:primary %%
**Execution Workflow**

```bash
cd ~/path/to/your/project/jni
clang -O2 -pie -fPIE main.c -o my_executable
chmod +x my_executable
./my_executable
```

%% col-break:b:primary %% 
**Terminal Output**
```
~/hello_jni/jni $ ls -al
total 23
drwx------. 2 u0_a460 u0_a460 3452 Nov  1 11:04 .
drwx------. 3 u0_a460 u0_a460 3452 Nov  1 11:04 ..
-rw-------. 1 u0_a460 u0_a460  345 Nov  1 11:04 Android.mk
-rw-------. 1 u0_a460 u0_a460  104 Nov  1 11:04 main.c
-rwx------. 1 u0_a460 u0_a460 4396 Nov  1 11:04 my_executable
~/hello_jni/jni $ ./my_executable
Hello from Android native executable!
```

%% col-end %%

  

# Notes

- **Filesystem Permission Restrictions**: Android mounts external storage partitions (such as `/sdcard/` and `/storage/emulated/0/`) with `noexec`, **preventing direct binary execution on these paths**. Ensure project files are placed under Termux's private directory (`$HOME` or `~`).
    
    
- If a project was created on external storage, move it to the home directory:
    
    ```
    cp -r ~/storage/shared/.../Projects/hello_jni $HOME/hello_jni
    cd $HOME/hello_jni/jni
    ```