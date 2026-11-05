
# Termux

- [document](https://wiki.termux.com/wiki/Getting_started)
- [GitHub - termux/termux-app: Termux - a terminal emulator application for Android OS extendible by variety of packages. · GitHub](https://github.com/termux/termux-app)

從 [這裡](https://github.com/termux/termux-app/releases/tag/v0.118.3) 下載 
32位元的老設備下載v7a，64位元的下載v8a。

安裝apk執行後，
![[termux_interface.png]]


## [補充] Multi-screen
安裝tmux套件
```
pkg install tmux 
tmux
```
- `CTRL + b + Shift + %` 水平分割
- `CTRL + b + Shift + "` 垂直分割


# Git

%% col-start %%
%% col-break:b:primary %%
```
pkg update
pkg install git gh openssh -y

```

%% col-break:b:primary %%
![git img](https://i.meee.com.tw/cH6bTVH.png)
%% col-end %%


登入github
```
gh auth login ​
```


# [練習] Hello Jni project
資料夾結構
```
project/
└── jni/
    ├── Android.mk (or Makefile)
    └── main.c
```

*如果是64bit設備，使用Android.mk，32bit使用Makefile*
- Example code

%% col-start %%
%% col-break:b:primary %%
main.c
```
#include <stdio.h>

int main() {
    printf("Hello from Android native executable!\n");
    return 0;
}
```
%% col-break:b:primary %%
Android.mk
```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := my_executable
LOCAL_SRC_FILES := main.c

include $(BUILD_EXECUTABLE)
```
%% col-break:b:primary %%
Makefile
```
CC = clang
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


# 安裝clang cmake, ndk, jdk 等 tool

確認硬體架構
```
uname -m
```

若顯示 armv8l or armv7l: Your Termux environment is running in 32-bit ARM mode.
若顯示 aarch64: Your Termux environment is running in 64-bit ARM mode.

```
termux-setup-storage
pkg install clang make cmake git openjdk-17 aapt apksigner ecj -y

```

## 使用NDK編譯
**只支援aarch64等架構**

```
cd~
wget https://dl.google.com/android/repository/android-ndk-r30-linux.zip
unzip android-ndk-r30-linux.zip
rm android-ndk-r30-linux.zip

```

將ndk tool chain加入環境變數
```
echo 'export ANDROID_NDK_HOME=$HOME/android-ndk-r30-linux' >> ~/.bashrc
echo 'export PATH=$ANDROID_NDK_HOME:$PATH' >> ~/.bashrc
source ~/.bashrc
```

接著在project/ 資料夾下`ndk-build` 即可輸出可執行檔。


## 使用Clang編譯
原因 armv8 (32bits) 找不到prebuilt ndk可改用clang，即可編譯單個檔案 (不搭配Makefile)

- Targeting 32-bit ARM (armeabi-v7a):
	- `clang -target armv7a-linux-androideabi29 -O2 -pie -fPIE main.c -o my_executable_arm32`
- Targeting 64-bit ARM (arm64-v8a):
	- `clang -target aarch64-linux-android29 -O2 -pie -fPIE main.c -o my_executable_arm64`
- Targeting a JNI Shared Library (.so) for an APK
	- `clang -target aarch64-linux-android29 -shared -fPIC main.c -o libnative.so`


%% col-start %%
%% col-break:b:primary %%

```
cd ~/path/to/your/project/jni
clang -O2 -pie -fPIE main.c -o my_executable
chmod +x my_executable
./my_executable

```

%% col-break:b:primary %%
輸出
```
drwx------. 2 u0_a460 u0_a460 3452 Nov  1 11:04 jni
~/hello_jni $ cd jni/
~/hello_jni/jni $ ls -al
total 23
drwx------. 2 u0_a460 u0_a460 3452 Nov  1 11:04 .
drwx------. 3 u0_a460 u0_a460 3452 Nov  1 11:04 ..
-rw-------. 1 u0_a460 u0_a460  345 Nov  1 11:04 Android.mk
-rw-------. 1 u0_a460 u0_a460  104 Nov  1 11:04 main.c
-rw-------. 1 u0_a460 u0_a460 4396 Nov  1 11:04 my_executable
~/hello_jni/jni $ chmod +x my_executable
~/hello_jni/jni $ ./my_executable
Hello from Android native executable!
```
%% col-end %%

## 使用Makefile編譯 JNI code

甚麼是 ndk-multilib?

```
pkg install clang make ndk-multilib ndk-sysroot -y
```

編譯&執行
```
make
./my_executable

```



## 注意
`/storage/emulated/0/` ,`/sdcard/` 等資料夾路徑不能跑可執行檔案，建議移到termux Home資料夾下。

如果不小心建錯位置，使用cp指令移動過去。

```
cp -r ~/storage/shared/.../Projects/hello_jni $HOME/hello_jni
cd $HOME/hello_jni/jni
```