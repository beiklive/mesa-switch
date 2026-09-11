面向 Nintendo Switch 的 Mesa
============================

这是一个将 `Mesa <https://mesa3d.org>`_ 移植到 Nintendo Switch（Horizon 系统）的项目，
通过 Mesa 的 Nouveau 驱动，在 Tegra X1 的 ``GM20B`` GPU 上提供原生的 EGL、OpenGL、
OpenGL ES 和 Vulkan 支持。

基于 Mesa 26.2.1。

本仓库位于 https://github.com/danfromtico/mesa-switch。


你将获得什么
------------

* **Vulkan** —— NVK，以无加载器（loaderless）方式构建。应用程序直接链接
  ``libvulkan.a``；不需要 Vulkan loader，也不需要安装 ICD。
* **OpenGL 与 OpenGL ES 1/2/3** —— 默认使用 Gallium NVC0，或在 NVK 之上叠加 Zink。
* **EGL** —— Switch 前端，支持 NWindow 表面和 pbuffer。

所有内容都静态链接进一个 NRO/NSO。显式 Vulkan 与任意一种 OpenGL 后端可以在同一进程中
共存，因为两者都构建在同一套 Horizon 后端之上。


与上游 Mesa 的差异
------------------

``src/nouveau/horizon`` 是一个直接基于 libnx 编写的 Horizon GPU 后端。它负责设备、
地址空间、内存、通道提交、syncpoint 同步、缓存维护以及错误处理。Gallium/NVC0 和 NVK
都是构建在这一共享后端之上的适配层。

构建前值得了解的影响：

* 不需要外部的 ``switch-libdrm_nouveau`` 库，也不需要打过补丁的 libnx ABI。
* Horizon 是统一内存平台，因此 CPU 可见的分配带有显式的缓存策略，GPU 工作通过原生
  syncpoint fence 进行排序。
* 未知的 GPU 完成状态会以失败关闭（fail closed）——资源会保持被持有或隔离，而不会在
  没有完成证据的情况下被回收。
* Horizon 没有 POSIX 文件描述符，也没有 DMA-BUF，因此 NVK 不暴露
  ``VK_KHR_external_memory_fd``、``VK_EXT_external_memory_dma_buf`` 或
  ``VK_EXT_map_memory_placed``。
* 在 Khronos 一致性测试完成之前，EGL 配置会被标记为不符合规范（non-conformant）。


依赖要求
--------

devkitA64 与 libnx，外加 libelf、expat、zlib、zstd、Meson、Ninja，以及一个包含
``aarch64-unknown-linux-gnu`` 标准库的 Rust 工具链。


构建
----

根据你的需要，有三种入口：

.. code-block:: sh

  ./build-switch.sh     # 仅 NVK Vulkan，通过 Docker（Docker.rust）构建
  ./build-opengl.sh     # 仅 EGL / OpenGL / OpenGL ES
  ./build-unified.sh    # 将两种 API 打包进同一个发布版 SDK

``build-switch.sh`` 在容器内完成整个交叉构建，因此宿主机唯一的要求就是 Docker。
它会在 ``builddir-switch`` 下产出静态归档文件。

``build-opengl.sh`` 基于 ``switch_cross_file.txt`` 进行交叉编译，并输出到
``mesa-install``。

``build-unified.sh`` 目前要求使用经过校验的 MSYS2 工具链，在其他环境下会拒绝运行。
它会把静态的 GL、GLES、EGL 和 Vulkan 库、Khronos 头文件、pkg-config 元数据以及
CMake 包文件安装到同一个 Switch portlibs 前缀下，然后写出一个确定性的 SDK ZIP 到
``dist``。除非设置 ``ALLOW_DIRTY=1``，否则它会拒绝在脏工作区（dirty checkout）上构建。

安装的 CMake 包会导出 ``OpenGL::GL``、``OpenGL::EGL``、``OpenGL::GLES1``、
``OpenGL::GLES2``、``Vulkan::Headers`` 和 ``Vulkan::Vulkan``。


在运行时选择 GL 后端
--------------------

默认使用 NVC0。在 ``eglInitialize`` 之前，通过 ``MESA_SWITCH_GL_DRIVER=zink`` 或
``MESA_LOADER_DRIVER_OVERRIDE=zink`` 选择 Zink；传入 ``MESA_SWITCH_GL_DRIVER=nvc0``
可强制使用原生 Gallium 驱动。一个 EGL display 在其生命周期内只使用一种后端。


持续集成
--------

``.github/workflows/switch-vulkan.yml`` 在推送 ``v*`` 形式的 tag（例如 ``v0.0.1``）
时触发，也可以在 GitHub Actions 页面手动触发。它会在 GitHub 托管的 runner 上通过
Docker 运行 ``build-switch.sh``，将 NVK 的静态库打包为
``mesa-switch-vulkan-<tag>.tar.gz``，上传为 workflow artifact，并附加到对应的
GitHub Release 上。


文档
----

`docs/switch-opengl.rst <docs/switch-opengl.rst>`_ 是本移植版的参考文档：架构、
完整的运行时环境变量集合、Zink 与呈现（presentation）行为，以及平台限制。

关于 Mesa 本身，请参阅上游文档 https://docs.mesa3d.org。


上游
----

上游 Mesa 位于 https://gitlab.freedesktop.org/mesa/mesa，非 Switch 移植专属的
bug 与补丁都应提交到那里。而 Horizon 后端、Switch WSI 或本仓库构建脚本相关的问题，
应提交到本仓库。


许可证
------

MIT，与上游 Mesa 一致。各个组件带有各自的许可证；详见 ``licenses/`` 目录以及文件
本身的头部声明。
