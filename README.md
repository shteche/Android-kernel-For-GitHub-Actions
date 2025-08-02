# Android-kernel-For-GitHub-Actions
通过 GitHub Actions 在线构建 Android 内核，其实就是把传统的本地编译流程放到 GitHub 的 CI/CD 自动化流程中。这个方案轻便、自动化、可复用，但需要注意一些限制（如 GitHub Action 的时限、空间限制等）。下面是一个从零开始的标准化步骤，适用于内核 4.4 ~ 5.x（包括旧内核），也适合主线内核构建方案的自定义拓展。


---

✅ 准备工作（一次性）

1. Fork 或新建仓库

创建一个 GitHub 仓库，比如叫 android-kernel-build。



2. 准备内核源码

方法一：GitHub 仓库直接包含 kernel 源码（不推荐，太大）

方法二：在 Actions 中用 git clone 拉取你指定的内核仓库



3. 准备工具链

推荐使用 AOSP 官方 Clang 或 GCC 工具链（适配 Android 内核版本）

可从 GitHub Release 或 AOSP 镜像下载，并在构建时解压使用





---

🧩 .github/workflows/build.yml 编写指南

这是 GitHub Actions 的配置文件，以下是一个标准模板：

name: Build Android Kernel

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Kernel Source
      uses: actions/checkout@v4

    - name: Clone Kernel Source (如果你不直接放在仓库里)
      run: |
        git clone https://github.com/YourUsername/android_kernel_xxx.git kernel
        cd kernel

    - name: Setup Build Environment
      run: |
        sudo apt-get update
        sudo apt-get install -y build-essential bc kmod cpio flex libncurses5-dev libelf-dev git wget unzip

    - name: Download Clang Toolchain
      run: |
        mkdir clang && cd clang
        wget https://github.com/ClangBuiltLinux/tc-build/releases/download/20231229/clang-r450784d.tar.gz
        tar -xzf clang-r450784d.tar.gz
        echo "CLANG_PATH=$PWD" >> $GITHUB_ENV

    - name: Build Kernel
      run: |
        cd kernel
        export ARCH=arm64
        export SUBARCH=arm64
        export CROSS_COMPILE=aarch64-linux-gnu-
        export CLANG_TRIPLE=aarch64-linux-gnu-
        export PATH=$CLANG_PATH/bin:$PATH

        make vendor/xxx_defconfig
        make -j$(nproc) CC=clang

    - name: Upload zImage or Image.gz-dtb
      uses: actions/upload-artifact@v4
      with:
        name: kernel-output
        path: |
          kernel/arch/arm64/boot/Image*
          kernel/arch/arm64/boot/dts/*.dtb


---

🔍 说明

步骤	说明

workflow_dispatch	允许手动触发构建
runs-on: ubuntu-latest	使用 GitHub 提供的虚拟机
actions/checkout@v4	拉取你仓库的内容（比如存放内核脚本）
make vendor/xxx_defconfig	使用厂商内核配置
CC=clang	使用 Clang 编译器，现代内核推荐
upload-artifact	将编译好的 Image.gz-dtb 或 zImage 下载回本地



---

⚠️ 注意事项

构建时间限制：GitHub Free 用户每个 job 最多 6 小时，推荐优化配置只编译必要模块。

存储限制：Actions 每个 artifact 最大 2GB，避免打包整个 out 目录。

缓存加速：可引入 actions/cache 对 Clang 和 prebuilts 缓存，提升构建效率。

防止超时被杀：长时间无输出可能导致 job 被终止，建议输出 log/进度。

如果要构建旧内核（如 3.x），可能需要 GCC + 自定义补丁，需特别处理工具链。


---
