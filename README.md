# 學習 開發筆記

(目前以Linux為主)
整理 Linux 工作環境、Android 原生程式開發，以及 Raspberry Pi 3 圖形驅動實驗的操作紀錄。內容以實作流程、指令與遇到的問題為主。

## 筆記索引

### Linux 基礎與版本控制

- [[Projects/Linux/Basics/Linux cli git|Linux CLI 與 Git：新裝置 SSH 設定]]：設定 GitHub SSH 金鑰、SSH 權限與連線，並處理曾以 `sudo` 操作 Git 後的檔案擁有權。

### Android 與 JNI

- [[Projects/Linux/Coding on Android/Android coding setup (Jni)|Android Termux 原生開發與 JNI]]：在 Termux 安裝 Git、編譯工具與 Java，並以 NDK、Makefile 或 Clang 建置 Android 原生執行檔及 `.so`；也整理 ABI 與外部儲存空間的執行限制。

### Raspberry Pi 與 Vulkan

- [[Projects/Linux/Raspberry Pi 3 with Vulkan DDK/DDK install with Lite OS|Raspberry Pi 3：Lite OS 與 Vulkan DDK]]：以 Raspberry Pi OS Lite 32-bit 建置環境，處理 Swap，並編譯及測試社群維護的 `rpi-vk-driver`。

	- **目前紀錄的結果**：`make test` 有 22 個案例，其中 6 個 SegFault。筆記推測與缺少 `.ppm` 測試素材有關，尚未記錄修復後的驗證結果。

## 建議閱讀順序

1. 先閱讀 Linux CLI 與 Git，完成新裝置的 SSH/Git 設定。
2. 若要直接在 Android 裝置上開發，接著閱讀 Termux 與 JNI 筆記。
3. 若要在 Raspberry Pi 3 嘗試 Vulkan，閱讀 Lite OS 與 DDK 筆記。


---


# Learning & Development Notes

(Currently focused primarily on Linux)
A collection of operational logs covering Linux development environments, Android native development, and Raspberry Pi 3 graphics driver experiments. Focuses primarily on hands-on workflows, commands, and encountered issues.

## Note Index

### Linux Basics & Version Control

- [[Projects/Linux/Basics/Linux cli git|Linux CLI & Git: New Device SSH Setup]]: Setting up GitHub SSH keys, SSH permissions, and connections, as well as fixing file ownership issues caused by previously running Git with `sudo`.

### Android & JNI

- [[Projects/Linux/Coding on Android/Android coding setup (Jni)|Android Termux Native Development & JNI]]: Installing Git, build tools, and Java in Termux, and compiling Android native executables and `.so` shared libraries using the NDK, Makefile, or Clang; also covers ABI considerations and execution restrictions on external storage.

### Raspberry Pi & Vulkan

- [[Projects/Linux/Raspberry Pi 3 with Vulkan DDK/DDK install with Lite OS|Raspberry Pi 3: Lite OS & Vulkan DDK]]: Setting up the environment on Raspberry Pi OS Lite (32-bit), managing Swap space, and compiling and testing the community-maintained `rpi-vk-driver`.

- 	**Currently Recorded Results**: `make test` ran 22 test cases, with 6 failing due to SegFaults. The notes speculate this is caused by missing `.ppm` test assets; verification results after fixing have not yet been documented.

## Suggested Reading Order

1. Start with **Linux CLI & Git** to complete the SSH and Git setup on a new device.
2. If developing directly on an Android device, continue with the **Termux & JNI** notes.
3. If experimenting with Vulkan on the Raspberry Pi 3, read the **Lite OS & DDK** notes.