# 目的
本流程以 Raspberry Pi OS Lite (32-bit) 為基底，帶你從燒錄、內核 KMS/CMA 設定、編譯 github上社群提供的rpi-vk-driver，到最後直接顯示執行 Vulkan 範例到螢幕上。

硬體設備 **Raspberry Pi 3 Model B V1.2**
![|300](https://i.meee.com.tw/FJdFv99.png)
	CPU:	1.2 GHz 64-bit quad-core ARM Cortex-A53
	GPU:	Dual Core VideoCore IV® Multimedia Co-Processor; Open GL ES 2.0; hardware-accelerated OpenVG; 1080p60 H.264 high-profie decode
	記憶體:	1GB LPDDR2（和 GPU 共享）


# 映像檔燒錄與開機配置
1. 下載並開啟 **Raspberry Pi Imager**。
2. **Device** 選擇 `Raspberry Pi 3`。
3. **OS** 選擇 `Raspberry Pi OS (other)` $\rightarrow$ **`Raspberry Pi OS Lite (32-bit)`**。
4. 點擊「Next」並在自訂設定中：
    - 啟用 **SSH**（設定使用者名稱與密碼，例如 `pi`）。
    - 設定 Wi-Fi（若不使用實體網路線）。
5. 寫入 MicroSD 卡。

> [!NOTE]- 為什麼不用 64-bit？
> Raspberry Pi 3 的 CPU（Broadcom BCM2837 / Cortex-A53）雖然硬體支援 64 位元架構（AArch64），但在本專案中**強烈建議使用 32-bit (armhf)**，原因如下：
> #### 1. `rpi-vk-driver` 的底層代碼相容性（最關鍵原因）
>- **沒有官方 64 位元 Vulkan 驅動**：Mesa 官方的 Vulkan 驅動（V3DV）**僅支援 Pi 4 (VC6) 與 Pi 5 (VC7)**，完全不支援 Pi 3 的 VideoCore IV (VC4)。
 >   
>- **民間專案基於 32 位元開發**：目前唯一能讓 Pi 3 跑 Vulkan 的只有 Martin Thomas 寫的 `rpi-vk-driver`。作者最初就是在 32 位元環境下開發與除錯。
 >   
>- **指標與 DRM IOCTL 對齊問題**：VC4 的暫存器映射、DRM 核心介面呼叫以及 SPIR-V 轉 QPU 組合語言的核心資料結構中，含有許多基於 32 位元指標（`uintptr_t` 為 4 bytes）的假設。若強行在 64-bit 環境編譯，經常會遇到指標截斷、記憶體對齊錯誤，甚至引發 Kernel IOCTL 參數解析異常。
 >   
>
>#### 2. 1GB 記憶體限制與指標膨脹（Pointer Overhead）
>
>- **記憶體開銷加倍**：在 64 位元模式下，所有記憶體指標從 4 bytes 增加為 8 bytes，使得資料結構變大、CPU L1/L2 快取命中率下降，整個系統的靜態與動態記憶體佔用會增加 20%～30%。
 >   
>- **CMA 連續記憶體擠壓**：為了讓 Vulkan 順利分配 Framebuffer 與紋理，我們在 `config.txt` 中切出了 **256MB** 給 GPU CMA（連續記憶體分配器）。
 >   
>- **極易觸發 OOM**：1GB 的實體記憶體扣除 CMA 後，作業系統與應用程式僅剩約 **700MB** 可用。若跑 64-bit 系統，編譯 `rpi-vk-driver` 或載入較複雜的著色器時，極易因記憶體耗盡而被系統 OOM Killer 強制關閉。
>
>


## Linux 開發環境設置
存檔後將 SD 卡插入 Raspberry Pi 3，接上 HDMI 螢幕與電源開機[^1]。
透過 SSH 登入樹莓派：`ssh pi@<樹莓派IP>`

Lite OS 沒有GUI，用SSH連只會看到CLI
![|200](https://i.meee.com.tw/fsVyhCa.png)



### 擴大 Swap 空間（防止編譯 OOM）

Pi 3 僅有 1GB RAM，本地編譯 C++ 很容易觸發 OOM Killer。先將 Swap 擴充至 2GB：

```
sudo dphys-swapfile swapoff
sudo sed -i 's/^CONF_SWAPSIZE=.*/CONF_SWAPSIZE=2048/' /etc/dphys-swapfile
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
```
驗證： 執行 `free -m`，確認 Swap 顯示約 2047 MB。

> [!NOTE]- 什麼是 Swap？
> 	**Swap（交換空間）是 Linux 的「虛擬記憶體」機制。**
> 	當系統的實體記憶體（Pi 3 僅有 1GB RAM）快要耗盡時，Linux 核心會把記憶體中暫時不用的資料「換出（Swap out）」並暫存到 MicroSD 卡的檔案中，騰出珍貴的實體記憶體給當前最耗資源的程式使用。
> 
> - **為什麼編譯 C++ 必須要 Swap？**  
>     編譯 `rpi-vk-driver`（包含 SPIR-V 解析器與 QPU 著色器編譯器）時，C++ 編譯器（`g++`）在做模板展開與代碼最佳化時會瞬間吞噬大量記憶體。若 1GB RAM 用盡且沒有足夠的 Swap，Linux 核心的 **OOM Killer (Out-of-Memory Killer)** 會直接無預警強制終止編譯程序，拋出 `c++: fatal error: Killed signal terminated program cc1plus` 的錯誤。

在基於 Debian 12 的 Raspberry Pi OS（Bookworm） 中，官方已經淘汰並移除了傳統的
dphys-swapfile管理套件，改回使用標準 Linux 原生 Swap 機制。

從`free -m`輸出可以看出：
```
               total        used        free      shared  buff/cache   available
Mem:             931         122         742           3         118         809
Swap:            930           0         930
```
系統目前已經自動掛載了約 930 MB（1GB）的預設 Swap，只是尚未達到建議的 2GB 安全值。

#### 將 Swap 擴充至 2GB

1. 檢查目前 Swap 檔案所在位置Bash

```bash
swapon --show

# Output
NAME       TYPE      SIZE USED PRIO
/dev/zram0 partition 931M   0B  100
```

`/dev/zram0` 不是一般檔案或資料夾，它是 **zram（記憶體壓縮區塊）**。

系統是直接從實體 RAM 中劃出一塊區域做即時壓縮來當作 Swap（速度極快，但不增加真實物理空間），它屬於虛擬區塊設備（Block Device），因此無法對它下 `fallocate` 擴展大小。


2. 暫時關閉目前的 SwapBash

```bash
sudo swapoff -a
```

3. 重新分配 2GB (2048MB) 空間給 Swap 檔BashBash

```bash
sudo fallocate -l 2G /swapfile
```
剛才執行的 `sudo fallocate -l 2G /swapfile`其實已經成功在 SD 卡建立了 2GB 檔案，只是在下完指令後漏掉了「格式化」與「掛載」步驟，所以`swapon --show`才會呈現空白。

4. 掛載
```bash
# 設定檔案安全權限（避免一般使用者讀取）
sudo chmod 600 /swapfile
# 將檔案格式化為 Swap 格式
sudo mkswap /swapfile

# Output
Setting up swapspace version 1, size = 2 GiB (2147479552 bytes) no label, UUID=084a5c20-036b-4de0-94ea-1610933b8789
```

5. 使用
```bash
# 重新啟用系統原本的 zram（若需要讓兩者共存）
sudo swapon -a

# 驗證目前 Swap 狀態
swapon --show

# Output
NAME       TYPE      SIZE USED PRIO
/dev/zram0 partition 931M   0B  100
/swapfile  file        2G   0B   -2
```

##### zram 與 SD 卡 Swapfile 共存的優點
- **優先使用 zram（PRIO 100）**：系統記憶體緊張時，會優先壓縮存入 RAM，保持極高讀寫速度。
- **溢流至 SD 卡 `/swapfile`（PRIO -2）**：當編譯大型 C++ 專案把 zram 也塞滿時，資料會無縫溢流到 SD 卡的 2GB 空間，確保編譯器不會被系統 OOM Killer 強制中斷。


#### 設定開機自動掛載（永久生效）
如果希望重開機後依然保有這 2GB 的 SD 卡 Swap，請將它加入`/etc/fstab`:
```bash
echo '/swapfile none swap sw,pri=10 0 0' | sudo tee -a /etc/fstab
```

Example
```bash
lontoone@lontoone:~ $ free -m
               total        used        free      shared  buff/cache   available
Mem:             931         122         693           3         166         809
Swap:           2978           0        2978

```

完成後，執行`free -m`確認 Swap 總量已接近 3GB（約 930M zram + 2048M swapfile），就可以繼續安裝依賴並編譯rpi-vk-driver。

---

# 跑 Vulkan
Origin repo: [Yours3lf/rpi-vk-driver: VK driver for the Raspberry Pi (Broadcom Videocore IV)](https://github.com/Yours3lf/rpi-vk-driver)

My fixed for new compiler fork: [Lontoone/rpi-vk-driver: VK driver for the Raspberry Pi (Broadcom Videocore IV)](https://github.com/Lontoone/rpi-vk-driver)


## 安裝編譯工具與相依函式庫

```bash
sudo apt update
sudo apt install -y git cmake build-essential ninja-build \
    libdrm-dev libx11-xcb-dev libxcb-dri3-dev libxcb-present-dev \
    libvulkan-dev vulkan-tools
    
# rpi-vk-driver 需要python
sudo apt install python3 python3-pip python3-dev
```

## 下載與編譯 rpi-vk-driver

`rpi-vk-driver` 包含驅動核心、SPIR-V 轉 QPU 組合語言編譯器及測試套件：

```bash
# 遞迴 Clone 專案（包含子模組）
git clone --recursive https://github.com/Yours3lf/rpi-vk-driver.git
cd rpi-vk-driver

# 建立編譯目錄
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_FLAGS="-D_GNU_SOURCE -Wno-error=implicit-function-declaration -Wno-error=int-conversion -Wno-error=incompatible-pointer-types" -Dpython=$(which python)

make

```

編譯完成後，build 目錄下會生成 driver/librpi-vk-driver.so，以及 tests/ 下的測試執行檔。

## 驗證/渲染

```bash
git clone https://github.com/Yours3lf/rpi-vk-driver.git
cd rpi-vk-driver
mkdir build && cd build
cmake ..
sudo make install
sudo make test

```

![|300](https://i.meee.com.tw/5qA6xLK.png)

### Run Test output
```
lontoone@lontoone:~/projects/rpi-vk-driver/build $ sudo make test
Running tests...
Test project /home/lontoone/projects/rpi-vk-driver/build
      Start  1: clear
 1/22 Test  #1: clear ............................   Passed    5.77 sec
      Start  2: triangle
 2/22 Test  #2: triangle .........................   Passed    5.71 sec
      Start  3: texturing
 3/22 Test  #3: texturing ........................***Exception: SegFault  0.56 sec
      Start  4: varyings
 4/22 Test  #4: varyings .........................   Passed    5.70 sec
....

73% tests passed, 6 tests failed out of 22

Total Test time (real) =  88.63 sec

The following tests FAILED:
          3 - texturing (SEGFAULT)
         11 - HDR (SEGFAULT)
         12 - ETC (SEGFAULT)
         14 - mipmapping (SEGFAULT)
         16 - cubemapping (SEGFAULT)
         21 - cubeMipmapping (SEGFAULT)
Errors while running CTest
Output from these tests are in: /home/lontoone/projects/rpi-vk-driver/build/Testing/Temporary/LastTest.log
Use "--rerun-failed --output-on-failure" to re-run the failed cases verbosely.
make: *** [Makefile:91: test] Error 8
```


**錯誤原因**

沒有提供圖檔， `find ~/projects/rpi-vk-driver -name "*.ppm”` 結果為空。
proof
```bash
lontoone@lontoone:~/projects/rpi-vk-driver/build $ grep -rn "readPPM" ~/projects/rpi-vk-driver/test/
driver/test/cubeMipmapping/cubeMipmapping.cpp:1348:    texDatas[5] = readPPM("cubemapData/negz.ppm");
/home/lontoone/projects/rpi-vk-driver/test/HDR/HDR.cpp:97:char* readPPM(const char* fileName)
....
lontoone@lontoone:~/projects/rpi-vk-driver/build $ find ~/projects/rpi-vk-driver -name "*.ppm"
# Empty output 
```




----





# Purpose
Based on Raspberry Pi OS Lite (32-bit), this guide walks you through flashing the image, configuring kernel KMS/CMA, compiling the community-provided `rpi-vk-driver` from GitHub, and ultimately running Vulkan examples directly to a display output.

Hardware Device: **Raspberry Pi 3 Model B V1.2**
![|300](https://i.meee.com.tw/FJdFv99.png)
	CPU:	1.2 GHz 64-bit quad-core ARM Cortex-A53
	GPU:	Dual Core VideoCore IV® Multimedia Co-Processor; OpenGL ES 2.0; hardware-accelerated OpenVG; 1080p60 H.264 high-profile decode
	Memory:	1GB LPDDR2 (shared with GPU)


# Image Flashing and Boot Configuration
1. Download and open **Raspberry Pi Imager**.
2. For **Device**, select `Raspberry Pi 3`.
3. For **OS**, select `Raspberry Pi OS (other)` $\rightarrow$ **`Raspberry Pi OS Lite (32-bit)`**.
4. Click "Next" and in the custom OS settings:
    - Enable **SSH** (set a username and password, e.g., `pi`).
    - Configure Wi-Fi (if not using an Ethernet cable).
5. Write the image to the MicroSD card.

> [!NOTE]- Why not 64-bit?
> Although the CPU of the Raspberry Pi 3 (Broadcom BCM2837 / Cortex-A53) natively supports 64-bit architecture (AArch64), **using 32-bit (armhf) is strongly recommended** for this project due to the following reasons:
> #### 1. Low-level Code Compatibility of `rpi-vk-driver` (The Most Critical Reason)
>- **No official 64-bit Vulkan driver**: Mesa's official Vulkan driver (V3DV) **only supports Pi 4 (VC6) and Pi 5 (VC7)**, offering zero support for the VideoCore IV (VC4) on Pi 3.
>    
>- **Community project developed on 32-bit**: Currently, the only solution to run Vulkan on Pi 3 is Martin Thomas's `rpi-vk-driver`. The author originally developed and debugged it exclusively in a 32-bit environment.
>    
>- **Pointer and DRM IOCTL alignment issues**: VC4 register mappings, DRM kernel interface calls, and core data structures for converting SPIR-V to QPU assembly rely heavily on 32-bit pointer assumptions (`uintptr_t` is 4 bytes). Attempting to compile under a 64-bit environment frequently results in pointer truncation, memory alignment faults, and even Kernel IOCTL parameter parsing exceptions.
>    
>
>#### 2. 1GB Memory Limit and Pointer Overhead
>
>- **Doubled memory overhead**: In 64-bit mode, all memory pointers increase from 4 bytes to 8 bytes. This enlarges data structures, decreases CPU L1/L2 cache hit rates, and increases the overall system static and dynamic memory footprint by 20%–30%.
>    
>- **CMA contiguous memory squeeze**: To allow Vulkan to allocate framebuffers and textures smoothly, we allocate **256MB** to the GPU CMA (Contiguous Memory Allocator) in `config.txt`.
>    
>- **High risk of OOM**: After subtracting the CMA allocation from the 1GB physical RAM, only about **700MB** remains available for the OS and applications. Running a 64-bit system makes it extremely easy to trigger the system OOM Killer when compiling `rpi-vk-driver` or loading complex shaders.


## Linux Development Environment Setup
Insert the SD card into the Raspberry Pi 3, connect an HDMI display and power supply to boot up[^1].
Log in via SSH: `ssh pi@<Raspberry-Pi-IP>`

Lite OS does not come with a GUI; connecting via SSH only presents a CLI:
![|200](https://i.meee.com.tw/fsVyhCa.png)



### Expanding Swap Space (Preventing Compilation OOM)

The Pi 3 has only 1GB of RAM, and compiling C++ locally easily triggers the OOM Killer. First, expand Swap to 2GB:

```bash
sudo dphys-swapfile swapoff
sudo sed -i 's/^CONF_SWAPSIZE=.*/CONF_SWAPSIZE=2048/' /etc/dphys-swapfile
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
```
Verification: Run `free -m` and verify Swap displays approximately 2047 MB.

> [!NOTE]- What is Swap?
> 	**Swap is the virtual memory mechanism in Linux.**
> 	When the physical memory of the system (only 1GB RAM on Pi 3) is near exhaustion, the Linux kernel swaps inactive data out of memory and temporarily writes it to a file on the MicroSD card, freeing up precious physical RAM for the most resource-intensive active process.
> 
> - **Why is Swap mandatory for compiling C++?**  
>      When compiling `rpi-vk-driver` (which includes the SPIR-V parser and QPU shader compiler), the C++ compiler (`g++`) consumes a massive amount of memory during template expansion and code optimization. If the 1GB RAM runs out without sufficient Swap, the Linux kernel's **OOM Killer (Out-of-Memory Killer)** will abruptly kill the compilation process without warning, throwing the error: `c++: fatal error: Killed signal terminated program cc1plus`.

In Raspberry Pi OS (Bookworm) based on Debian 12, the legacy `dphys-swapfile` management package has been deprecated and removed in favor of the standard native Linux Swap mechanism.

As seen from the `free -m` output:
```plaintext
               total        used        free      shared  buff/cache   available
Mem:             931         122         742           3         118         809
Swap:            930           0         930
```
The system currently mounts an automatic default Swap of around 930 MB (~1GB), which has not yet reached the recommended safe margin of 2GB.

#### Expanding Swap to 2GB

1. Check the current Swap location:

```bash
swapon --show

# Output
NAME       TYPE      SIZE USED PRIO
/dev/zram0 partition 931M   0B  100
```

`/dev/zram0` is not a standard file or directory; it is **zram (compressed RAM block)**.

The system allocates a block directly from physical RAM for on-the-fly compression as Swap (extremely fast, but does not add real physical capacity). Because it is a virtual block device, running `fallocate` to expand its size is not possible.


2. Temporarily disable the current Swap:

```bash
sudo swapoff -a
```

3. Allocate a 2GB (2048MB) file for Swap:

```bash
sudo fallocate -l 2G /swapfile
```
The command `sudo fallocate -l 2G /swapfile` successfully created a 2GB file on the SD card; however, `swapon --show` appears blank because the formatting and mounting steps have not been performed yet.

4. Mount the Swap file:
```bash
# Set secure file permissions (prevent unauthorized reading)
sudo chmod 600 /swapfile
# Format the file as Swap
sudo mkswap /swapfile

# Output
Setting up swapspace version 1, size = 2 GiB (2147479552 bytes) no label, UUID=084a5c20-036b-4de0-94ea-1610933b8789
```

5. Enable and verify:
```bash
# Re-enable the system's original zram (if keeping both concurrently)
sudo swapon -a

# Verify current Swap status
swapon --show

# Output
NAME       TYPE      SIZE USED PRIO
/dev/zram0 partition 931M   0B  100
/swapfile  file        2G   0B   -2
```

##### Advantages of Coexisting zram and SD Card Swapfile
- **Prioritize zram (PRIO 100)**: When memory pressure rises, the system compresses and caches data into RAM first, maintaining peak read/write speeds.
- **Overflow to SD card `/swapfile` (PRIO -2)**: When building large C++ codebases fills up the zram, memory pages seamlessly spill over into the 2GB SD card space, preventing the compiler from being terminated by the OOM Killer.


#### Enable Auto-mount on Boot (Persistent Configuration)
To persist this 2GB SD card Swap across reboots, append it to `/etc/fstab`:
```bash
echo '/swapfile none swap sw,pri=10 0 0' | sudo tee -a /etc/fstab
```

Example:
```bash
lontoone@lontoone:~ $ free -m
               total        used        free      shared  buff/cache   available
Mem:             931         122         693           3         166         809
Swap:           2978           0        2978
```

After completion, run `free -m` to confirm the total Swap is close to 3GB (~930M zram + 2048M swapfile). You can now proceed to install dependencies and compile `rpi-vk-driver`.

---

# Running Vulkan
Original repo: [Yours3lf/rpi-vk-driver: VK driver for the Raspberry Pi (Broadcom Videocore IV)](https://github.com/Yours3lf/rpi-vk-driver)

My fork fixed for modern compilers: [Lontoone/rpi-vk-driver: VK driver for the Raspberry Pi (Broadcom Videocore IV)](https://github.com/Lontoone/rpi-vk-driver)


## Installing Build Tools and Dependencies

```bash
sudo apt update
sudo apt install -y git cmake build-essential ninja-build \
    libdrm-dev libx11-xcb-dev libxcb-dri3-dev libxcb-present-dev \
    libvulkan-dev vulkan-tools
    
# rpi-vk-driver requires Python
sudo apt install python3 python3-pip python3-dev
```

## Downloading and Compiling rpi-vk-driver

`rpi-vk-driver` includes the driver core, the SPIR-V to QPU assembly compiler, and a test suite:

```bash
# Clone the repository recursively (including submodules)
git clone --recursive [https://github.com/Yours3lf/rpi-vk-driver.git](https://github.com/Yours3lf/rpi-vk-driver.git)
cd rpi-vk-driver

# Create build directory
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_FLAGS="-D_GNU_SOURCE -Wno-error=implicit-function-declaration -Wno-error=int-conversion -Wno-error=incompatible-pointer-types" -Dpython=$(which python)

make
```

Once compilation completes, `driver/librpi-vk-driver.so` and test executables under `tests/` will be generated inside the `build` directory.

## Verification / Rendering

```bash
git clone [https://github.com/Yours3lf/rpi-vk-driver.git](https://github.com/Yours3lf/rpi-vk-driver.git)
cd rpi-vk-driver
mkdir build && cd build
cmake ..
sudo make install
sudo make test
```

![|300](https://i.meee.com.tw/5qA6xLK.png)

### Run Test output
```plaintext
lontoone@lontoone:~/projects/rpi-vk-driver/build $ sudo make test
Running tests...
Test project /home/lontoone/projects/rpi-vk-driver/build
      Start  1: clear
 1/22 Test  #1: clear ............................   Passed     5.77 sec
      Start  2: triangle
 2/22 Test  #2: triangle .........................   Passed     5.71 sec
      Start  3: texturing
 3/22 Test  #3: texturing ........................***Exception: SegFault  0.56 sec
      Start  4: varyings
 4/22 Test  #4: varyings .........................   Passed     5.70 sec
....

73% tests passed, 6 tests failed out of 22

Total Test time (real) =  88.63 sec

The following tests FAILED:
	  3 - texturing (SEGFAULT)
	 11 - HDR (SEGFAULT)
	 12 - ETC (SEGFAULT)
	 14 - mipmapping (SEGFAULT)
	 16 - cubemapping (SEGFAULT)
	 21 - cubeMipmapping (SEGFAULT)
Errors while running CTest
Output from these tests are in: /home/lontoone/projects/rpi-vk-driver/build/Testing/Temporary/LastTest.log
Use "--rerun-failed --output-on-failure" to re-run the failed cases verbosely.
make: *** [Makefile:91: test] Error 8
```


**Cause of Error**

Image assets were missing; running `find ~/projects/rpi-vk-driver -name "*.ppm"` returned no results.
Proof:
```bash
lontoone@lontoone:~/projects/rpi-vk-driver/build $ grep -rn "readPPM" ~/projects/rpi-vk-driver/test/
driver/test/cubeMipmapping/cubeMipmapping.cpp:1348:    texDatas[5] = readPPM("cubemapData/negz.ppm");
/home/lontoone/projects/rpi-vk-driver/test/HDR/HDR.cpp:97:char* readPPM(const char* fileName)
....
lontoone@lontoone:~/projects/rpi-vk-driver/build $ find ~/projects/rpi-vk-driver -name "*.ppm"
# Empty output 
```