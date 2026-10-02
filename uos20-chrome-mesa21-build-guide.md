# UOS 20 1031 ARM64：为 Chrome 编译私有 Mesa 21.1.8 / libgbm 测试包

## 1. 目标

本说明用于验证以下兼容性问题：

- 目标机：UOS 20 1031 ARM64
- glibc：2.28
- kernel：4.19.0-arm64-desktop
- GPU：AMD Oland，内核驱动 `amdgpu`
- 系统 Mesa/libgbm：18.3.6.6
- Chrome/Chromium 有头窗口白屏
- Chrome 日志出现：
  - `Failed to get fd for plane`
  - `Failed to export buffer to dma_buf`

Mesa 21.1 引入了 `gbm_bo_get_fd_for_plane()`。本方案先只编译一份 **Mesa 21.1.8 的 libgbm**，放到 `/opt/mesa21`，只让 Chrome 临时加载它。

**不替换 `/usr/lib` 中任何系统库。**

这是第一阶段验证。它的目的不是一次性升级整套显卡驱动，而是回答：

> Chrome 如果能使用带 `gbm_bo_get_fd_for_plane()` 的新版 libgbm，白屏和 dma-buf 错误是否会消失？

---

## 2. 为什么使用 rsou 的 GitHub Actions runner

本方案使用以下构建环境；执行前应确认仓库有权使用该 runner：

```text
GitHub runner: ubuntu-24.04-arm
源码构建容器: debian:buster（由 docker run 启动）
```

这个组合适合本任务：

- GitHub runner 本身是原生 ARM64；
- `debian:buster` 用户态使用 glibc 2.28；
- 与 UOS 20 1031 的 glibc 2.28 对齐；
- 不需要交叉编译；
- 编译结果可以在 CI 中检查最高 GLIBC 符号版本。

不要直接在 `ubuntu-24.04-arm` 宿主环境编译，否则产物可能要求 glibc 2.39，不能在 UOS 20 上运行。

---

## 3. 选用版本

第一阶段固定：

```text
Mesa 21.1.8
```

原因：

1. `gbm_bo_get_fd_for_plane()` 从 Mesa 21.1.0 开始提供；
2. 21.1.8 是 21.1 系列最后的 bugfix release；
3. 相比更新的 Mesa，和 UOS 20 的年代、依赖差距更小；
4. 第一阶段只需要新版 libgbm，不需要 Vulkan、radeonsi、LLVM 等完整图形栈。

Mesa 21.1.8 源码 SHA256：

```text
5cd32f5d089dca75300578a3d771a656eaed652090573a2655fe4e7022d56bfc
```

---

## 4. 推荐文件

在 `sg8010/rsou` 新增：

```text
.github/workflows/mesa-gbm-arm64.yml
```

不要修改现有：

```text
.github/workflows/linux-arm64.yml
```

独立 workflow 不改变 rsou 的构建配置。首次使用 `workflow_dispatch` 前，需要把该 workflow 提交并推送到仓库默认分支，之后再在 Actions 页面选择要运行的分支。

---

## 5. GitHub Actions workflow

以下是待 CI 验证的候选构建流程，不代表已经在 ARM64 runner 或 UOS 实机跑通。它包含一个明确的 Mesa 构建选择修正，源码断言不满足时立即停止，不能跳过断言继续打包。

将下面内容完整保存为：

```text
.github/workflows/mesa-gbm-arm64.yml
```

```yaml
name: Build Mesa GBM for UOS20 arm64

on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  build:
    name: Mesa 21.1.8 GBM arm64 glibc 2.28
    runs-on: ubuntu-24.04-arm

    steps:
      # JavaScript action 在 Ubuntu 宿主执行；仅源码构建放入 Buster。
      # 本流程只下载上游源码，无须 checkout 仓库。
      - name: Write build script
        shell: bash
        run: |
          cat > build-mesa-gbm.sh <<'BUILD_SCRIPT'
          #!/bin/bash
          set -euo pipefail
          printf '%s\n' \
            'deb http://archive.debian.org/debian/ buster main' \
            'deb http://archive.debian.org/debian/ buster-updates main' \
            'deb http://archive.debian.org/debian-security/ buster/updates main' \
            > /etc/apt/sources.list
          apt-get -o Acquire::Check-Valid-Until=false update

          apt-get install -y --no-install-recommends \
            ca-certificates curl xz-utils \
            gcc g++ libc6-dev binutils file \
            pkg-config ninja-build \
            python3 python3-pip python3-mako \
            bison flex \
            libdrm-dev libexpat1-dev zlib1g-dev

          python3 -m pip install --no-cache-dir 'meson==0.59.4'

          echo "=== build environment ==="
          uname -m
          ldd --version | sed -n '1p'
          gcc --version | sed -n '1p'
          python3 --version
          meson --version
          ninja --version
          dpkg-query -W libdrm2 libdrm-dev

          curl --retry 3 -fL \
            https://archive.mesa3d.org/older-versions/21.x/mesa-21.1.8.tar.xz \
            -o mesa-21.1.8.tar.xz

          echo \
            '5cd32f5d089dca75300578a3d771a656eaed652090573a2655fe4e7022d56bfc  mesa-21.1.8.tar.xz' \
            | sha256sum -c -

          tar -xf mesa-21.1.8.tar.xz

          cd mesa-21.1.8

          # 保留空的驱动列表，但显式选择 GBM 所需的 DRI 后端代码。
          # 这是本测试包的局部构建修正，不是上游原样构建。
          python3 - <<'PY_PATCH'
          from pathlib import Path
          path = Path('meson.build')
          source = path.read_text()
          old = 'with_dri = dri_drivers.length() != 0'
          assert source.count(old) == 1, 'Mesa build layout changed; stop and review'
          new = old + " or get_option('gbm') == 'enabled'"
          path.write_text(source.replace(old, new))
          PY_PATCH
          meson setup build \
            --prefix=/opt/mesa21 \
            --libdir=lib \
            --buildtype=release \
            --auto-features=disabled \
            -Dplatforms= \
            -Ddri-search-path=/usr/lib/aarch64-linux-gnu/dri \
            -Dgbm=enabled \
            -Degl=disabled \
            -Dglx=disabled \
            -Dopengl=false \
            -Dgles1=disabled \
            -Dgles2=disabled \
            -Dgallium-drivers= \
            -Ddri-drivers= \
            -Dvulkan-drivers= \
            -Dllvm=disabled \
            -Dosmesa=false \
            -Dzstd=disabled

          # 后端必须实际参与编译，不能只验收导出符号。
          python3 - <<'PY_BACKEND'
          import json
          from pathlib import Path
          commands = json.loads(Path('build/compile_commands.json').read_text())
          backend = [x for x in commands if x['file'].endswith('/gbm_dri.c')]
          assert backend, 'GBM DRI backend not selected; stop'
          assert all('-DHAVE_DRI' in x.get('command', ' '.join(x.get('arguments', [])))
                     for x in backend), 'HAVE_DRI missing; stop'
          PY_BACKEND
          ninja -C build
          DESTDIR=/work/stage ninja -C build install
          cd /work

          LIB=stage/opt/mesa21/lib/libgbm.so.1

          test -e "$LIB"

          echo "=== file ==="
          file "$LIB"

          echo "=== ELF ==="
          readelf -h "$LIB" | grep -E 'Class|Machine'
          readelf -h "$LIB" | grep -q 'Machine:.*AArch64'

          echo "=== required symbol ==="
          nm -D "$LIB" | grep ' gbm_bo_get_fd_for_plane$'

          echo "=== dependencies ==="
          readelf -d "$LIB" | grep NEEDED

          echo "=== GLIBC symbols ==="
          objdump -T "$LIB" 2>/dev/null \
            | grep -oE 'GLIBC_[0-9.]+' \
            | sort -Vu

          MAX=$(
            objdump -T "$LIB" 2>/dev/null \
              | grep -oE 'GLIBC_2\.[0-9]+' \
              | sed 's/GLIBC_//' \
              | sort -V \
              | tail -1
          )

          test -n "$MAX"
          echo "highest required GLIBC: $MAX"

          HIGH=$(printf '%s\n' "$MAX" "2.28" | sort -V | tail -1)
          test "$HIGH" = "2.28"

          # 记录构建配置和依赖，供目标机排查。
          meson configure mesa-21.1.8/build > stage/opt/mesa21/build-options.txt
          readelf -d "$LIB" > stage/opt/mesa21/libgbm-needed.txt
          cp gbm-smoke.c stage/opt/mesa21/
          mkdir -p stage/opt/mesa21/bin
          gcc -Wall -Wextra -Werror -O2 gbm-smoke.c \
            -Istage/opt/mesa21/include -Lstage/opt/mesa21/lib -lgbm \
            -o stage/opt/mesa21/bin/gbm-smoke
          readelf -h stage/opt/mesa21/bin/gbm-smoke | grep -q 'Machine:.*AArch64'
          python3 - <<'PY_ABI'
          import re, subprocess
          for path in ('stage/opt/mesa21/lib/libgbm.so.1',
                       'stage/opt/mesa21/bin/gbm-smoke'):
              output = subprocess.check_output(['readelf', '--version-info', path], text=True)
              versions = [tuple(map(int, x.split('.')))
                          for x in re.findall(r'GLIBC_(\d+(?:\.\d+)+)', output)]
              assert versions and max(versions) <= (2, 28), (path, versions)
          PY_ABI
          # Buster 用户态内加载一次；这里没有目标 GPU，只检查动态链接。
          LD_LIBRARY_PATH=/work/stage/opt/mesa21/lib \
            stage/opt/mesa21/bin/gbm-smoke --load-only
          tar -C stage -czf mesa21-gbm-uos20-arm64.tar.gz opt/mesa21
          sha256sum mesa21-gbm-uos20-arm64.tar.gz \
            > mesa21-gbm-uos20-arm64.tar.gz.sha256

          BUILD_SCRIPT

      - name: Write GBM smoke probe
        shell: bash
        run: |
          cat > gbm-smoke.c <<'C_SOURCE'
          #include <gbm.h>
          #include <fcntl.h>
          #include <stdio.h>
          #include <string.h>
          #include <unistd.h>

          int main(int argc, char **argv) {
            if (argc == 2 && strcmp(argv[1], "--load-only") == 0) {
              puts("dynamic load OK; GPU not tested");
              return 0;
            }
            if (argc != 2) {
              fprintf(stderr, "usage: %s /dev/dri/renderDxxx\n", argv[0]);
              return 2;
            }
            int fd = open(argv[1], O_RDWR | O_CLOEXEC);
            if (fd < 0) { perror("open DRM node"); return 1; }
            struct gbm_device *dev = gbm_create_device(fd);
            if (!dev) { perror("gbm_create_device"); close(fd); return 1; }
            printf("backend: %s\n", gbm_device_get_backend_name(dev));
            struct gbm_bo *bo = gbm_bo_create(dev, 64, 64,
              GBM_FORMAT_ARGB8888, GBM_BO_USE_RENDERING | GBM_BO_USE_LINEAR);
            int failed = 0;
            if (!bo) { perror("gbm_bo_create"); failed = 1; }
            else {
              int planes = gbm_bo_get_plane_count(bo);
              if (planes <= 0) { fprintf(stderr, "invalid plane count\n"); failed = 1; }
              for (int i = 0; i < planes; ++i) {
                int plane_fd = gbm_bo_get_fd_for_plane(bo, i);
                if (plane_fd < 0) { perror("gbm_bo_get_fd_for_plane"); failed = 1; }
                else { printf("plane %d export OK\n", i); close(plane_fd); }
              }
              gbm_bo_destroy(bo);
            }
            gbm_device_destroy(dev);
            close(fd);
            return failed;
          }
          C_SOURCE

      - name: Build in Buster arm64
        shell: bash
        run: |
          docker run --rm \
            -v "$GITHUB_WORKSPACE:/work" -w /work \
            debian:buster bash /work/build-mesa-gbm.sh

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: mesa21-gbm-uos20-arm64
          path: |
            mesa21-gbm-uos20-arm64.tar.gz
            mesa21-gbm-uos20-arm64.tar.gz.sha256
          if-no-files-found: error
```

---

## 6. CI 成功标准

GitHub Actions 必须同时满足以下条件。CI 没有目标机 GPU，成功只表示候选产物通过构建和动态加载检查。

### 6.1 架构正确

应看到类似：

```text
Machine: AArch64
```

### 6.2 glibc 不超过 2.28

应看到：

```text
highest required GLIBC: 2.28
```

或更低。

### 6.3 必须存在目标符号

这一条必须成功：

```bash
nm -D libgbm.so.1 | grep ' gbm_bo_get_fd_for_plane$'
```

应出现：

```text
gbm_bo_get_fd_for_plane
```

没有这个符号，则这个产物没有测试价值。

### 6.4 后端与动态加载检查

必须通过编译命令中的 `HAVE_DRI`、`gbm_dri.c` 编译参与检查，以及探针 `--load-only` 的动态加载检查。打包内容应包括 `bin/gbm-smoke`、`gbm-smoke.c`、`build-options.txt` 和 `libgbm-needed.txt`。

该流程通过局部修改 `with_dri` 的构建选择，保留 DRI 后端并继续关闭所有驱动的编译。**尚未在 Mesa 21.1.8 ARM64 上实编验证**：若配置、后端检查或链接失败，保存完整日志、停止部署，先修正构建选择；不能把这种失败解释为目标机驱动不兼容。

### 6.5 产生 Artifact

Actions 页面应出现：

```text
mesa21-gbm-uos20-arm64
```

里面有：

```text
mesa21-gbm-uos20-arm64.tar.gz
mesa21-gbm-uos20-arm64.tar.gz.sha256
```

---

## 7. UOS 目标机安装

把 Artifact 中的：

```text
mesa21-gbm-uos20-arm64.tar.gz
mesa21-gbm-uos20-arm64.tar.gz.sha256
```

两个文件一起拷到 UOS 20 机器，并在它们所在目录执行校验。

先校验：

```bash
sha256sum -c mesa21-gbm-uos20-arm64.tar.gz.sha256
```

先用 `tar -tzf mesa21-gbm-uos20-arm64.tar.gz` 查看内容，应只有 `opt/mesa21/` 下的测试文件。若 `/opt/mesa21` 已存在，先确认其来源并另行处理，避免覆盖其他安装。

安装到 `/opt`：

```bash
test ! -e /opt/mesa21 && sudo tar -C / -xzf mesa21-gbm-uos20-arm64.tar.gz
```

检查：

```bash
nm -D /opt/mesa21/lib/libgbm.so.1|grep get_fd_for_plane
```

应有输出。

---

## 8. 目标机预检和 GBM 功能测试

先记录环境，确认实际浏览器路径和版本；若安装的是 Chromium 或定制 ARM64 包，将后续命令中的 `google-chrome-stable` 换成真实启动命令。本文中的 Chrome 151/154 仅是原问题的版本标签，不代表已验证其包来源、架构或兼容性。

```bash
uname -a
getconf GNU_LIBC_VERSION
command -v google-chrome-stable
google-chrome-stable --version
dpkg-query -W libgbm1 libdrm2
ls -l /dev/dri
ls -l /usr/lib/aarch64-linux-gnu/dri/radeonsi_dri.so
LD_LIBRARY_PATH=/opt/mesa21/lib ldd -r /opt/mesa21/lib/libgbm.so.1
```

`ldd -r` 不应出现 `not found`、版本缺失或未解析符号。它只检查直接加载的库，不能代替后续 DRI 动态加载检查。

确认系统 DRI 目录。如果上面的路径不存在，用以下命令查找，再更新后续的 `DRI_DIR`，不要假定 UOS 一定采用标准 Debian 路径：

```bash
find /usr/lib -name radeonsi_dri.so -print
```

在同一个终端设置测试变量，并保留该终端供第 9 节使用：

```bash
DRI_DIR=/usr/lib/aarch64-linux-gnu/dri
TEST_DIR=$(mktemp -d /tmp/mesa21-test.XXXXXX)
export DRI_DIR TEST_DIR
```

从 sysfs 确认 DRM 节点归属，选择 AMD 显卡对应、当前用户可读写的 render 节点，不要默认它必然是 `renderD128`：

```bash
for node in /sys/class/drm/renderD*; do
  test -e "$node" || continue
  printf '%s: ' "${node##*/}"
  cat "$node/device/vendor"
  readlink -f "$node/device/driver"
done
```

AMD PCI vendor 通常为 `0x1002`。把下面的节点换成实际结果，再运行探针：

```bash
DRM_NODE=/dev/dri/renderD128
LD_LIBRARY_PATH=/opt/mesa21/lib \
LIBGL_DRIVERS_PATH="$DRI_DIR" LIBGL_DEBUG=verbose \
  /opt/mesa21/bin/gbm-smoke "$DRM_NODE" \
  >"$TEST_DIR/gbm-smoke.log" 2>&1
SMOKE_STATUS=$?
cat "$TEST_DIR/gbm-smoke.log"
printf 'probe exit status: %s\n' "$SMOKE_STATUS"
```

探针应打印后端和各 plane 的 `export OK`，退出码为 0。权限、驱动查找、设备创建或导出失败时，先定位失败环节。探针只覆盖一组简单的 ARGB8888/linear buffer；成功不保证 Chrome 的全部格式、modifier 或 sandbox 路径也成功，失败也不能直接归因于 ABI 不兼容。

---

## 9. Chrome 对照测试与实际加载检查

### 9.1 正常退出与保存日志

先保存工作，通过菜单正常退出 Chrome。若退出后仍有残留，先检查进程再向确认的进程发送 `TERM`。不要默认对所有 Chrome 执行 `pkill -9`。

在第 8 节的同一终端，使用两个全新的 profile、同一个页面和相同参数进行对照。每次观察并记录后正常关闭窗口，再开始下一轮。以下命令在前台运行；`tee` 保存完整日志，管道的最后退出码不代表 Chrome 的退出码。

系统库基线：

```bash
env -u LD_LIBRARY_PATH -u LIBGL_DRIVERS_PATH -u GBM_DRIVERS_PATH \
  google-chrome-stable \
  --user-data-dir="$TEST_DIR/profile-system" \
  --enable-logging=stderr --v=1 \
  'chrome://gpu' 2>&1 | tee "$TEST_DIR/chrome-system.log"
```

私有 GBM 测试：

```bash
env -u GBM_DRIVERS_PATH \
  LD_LIBRARY_PATH=/opt/mesa21/lib LIBGL_DRIVERS_PATH="$DRI_DIR" \
  google-chrome-stable \
  --user-data-dir="$TEST_DIR/profile-mesa21" \
  --enable-logging=stderr --v=1 \
  'chrome://gpu' 2>&1 | tee "$TEST_DIR/chrome-mesa21.log"
```

第一轮不叠加 `--disable-gpu`、`--no-sandbox`、`--no-xshm` 或强制 SwiftShader。测试 GPU 页面后，两轮都打开同一个能够复现白屏的页面。记录 `chrome://gpu` 中的 Graphics Feature Status、GL_VENDOR、GL_RENDERER 和 GL_VERSION；若测试轮转为软件渲染，不能据此证明硬件 GBM 路径修复。

### 9.2 检查正在运行的 GPU 子进程

保持私有 GBM 测试窗口打开，在另一个终端执行：

```bash
ps -eo pid,ppid,args | grep '[t]ype=gpu-process'
```

通过进程树、浏览器启动命令和任务管理器核对，找到本次独立 profile 对应的 GPU 子进程 PID。不要直接选择机器上第一个 GPU 进程。将下面的 PID 替换成实际值：

```bash
GPU_PID=12345
readlink -f "/proc/$GPU_PID/exe"
grep -E 'libgbm|/dri/.*_dri\.so|libdrm|libEGL|libGL' "/proc/$GPU_PID/maps"
```

如果读取权限不足，可用 `sudo grep` 读取该进程的 maps；不需要关闭 Chrome sandbox。GPU 子进程重启后，应重新确认 PID 和映射。

有效测试需要看到该 GPU 子进程映射了 `/opt/mesa21/lib/libgbm.so.1.0.0` 或其实际版本文件，并记录 DRI 驱动来自系统目录。`maps` 通常显示真实文件名，不一定显示 `libgbm.so.1` 软链接名。

若使用软件渲染、GPU 进程持续退出、私有库未映射或系统 DRI 驱动未加载，先解释这些现象，不能直接判定新版 GBM 无效。`LD_DEBUG=libs` 可辅助排查启动器加载，但仅出现搜索路径不能证明 GPU 子进程已使用该库；复跑时仍须使用全新的独立 profile。

---

## 10. 结果如何判定

### 情况 A：窗口恢复，GBM 错误消失

若同时确认私有 GBM、系统 DRI 驱动和硬件 renderer，则支持“这组私有 GBM 与系统驱动组合改善了问题”的结论。仅凭窗口恢复，不能证明 `gbm_bo_get_fd_for_plane()` 被调用或回退完全没有发生。

Chromium 上游实现会优先调用这个接口，但返回负值时仍可能回退到 `drmPrimeHandleToFD()`。还需要核对实际浏览器构建是否包含该逻辑。

先用同版本重复对照；若还要测试原问题中的其他版本，记录准确版本、安装包来源，按第 9 节重做两轮对照。不要仅凭文档中的版本号盲目升级。

### 情况 B：窗口仍白，但 GBM 错误消失

记录为“观察到该日志消失”。确认硬件渲染仍启用、GPU 进程稳定、页面确实触发同一操作后，才能认为导出问题可能改善。抓取完整日志继续定位显示问题，不能直接断言已修复一个确定的 GBM 问题、另有一个独立 X11 问题。

### 情况 C：白屏和错误均不变

依次检查 GPU 子进程库映射、DRI 目录、探针结果、实际浏览器是否支持新接口，以及完整的 errno。新接口存在但导出失败时，Chromium 仍可能走回退。

只有环境和测试路径均得到确认，才记录“仅替换 GBM 未解决此机器上的问题”。错误不变本身不能证明必须升级 radeonsi 或整套 Mesa。

### 情况 D：缺库、缺符号或驱动加载失败

记录完整报错、实际映射路径和 `ldd -r` 结果，区分库路径、libdrm 符号、DRI 加载及浏览器自身依赖。先修复已定位的最小问题，不要自动进入整套升级。

---

## 11. 回滚

本方案没有替换任何 UOS 系统文件。

先正常退出测试 Chrome，再启动不带私有环境变量的新进程，即可恢复使用系统库。已运行的进程不会因为删除文件自动切换库。

如需删除测试安装，先确认 `/opt/mesa21` 是本次专用目录，再执行：

```bash
sudo rm -rf /opt/mesa21
```

或者直接不用：

```text
LD_LIBRARY_PATH=/opt/mesa21/lib
```

同时去掉 `LIBGL_DRIVERS_PATH`、`GBM_DRIVERS_PATH` 等测试变量后，新启动的 Chrome 应重新加载系统库；可用第 9 节的进程映射确认：

```text
/usr/lib/aarch64-linux-gnu/libgbm.so.1
```

---

## 12. 明确禁止的操作

第一阶段不要执行：

```bash
cp libgbm.so.1 /usr/lib/aarch64-linux-gnu/
```

不要修改：

```text
/usr/lib/aarch64-linux-gnu/libgbm.so.1
/usr/lib/aarch64-linux-gnu/libdrm.so.2
/usr/lib/aarch64-linux-gnu/dri/*
```

不要运行：

```bash
ldconfig
```

来强行把 `/opt/mesa21` 注册成系统默认图形库。

这些操作可能让 UOS 桌面、登录界面或其他 GTK/Qt 程序一起受到影响。

---

## 13. 第二阶段什么时候再做

先满足以下前提，再讨论第二阶段：

1. 候选构建通过 CI 后端、动态加载和 ABI 检查；
2. 目标机的实际 GPU 子进程已加载私有 GBM，系统 DRI 驱动路径及 renderer 已确认；
3. 相同浏览器版本和操作下，问题可以重复出现；
4. 完整日志或功能探针表明剩余问题涉及现有 libdrm/DRI 的具体能力或兼容性，而不是路径、权限、profile 或浏览器启动问题。

仅“加载了新 GBM”不是进入第二阶段的理由。应先评估已定位问题是否能通过更小的修正解决。

第二阶段可能需要：

```text
新 libdrm
Mesa 21.x
libgbm
libEGL
radeonsi_dri.so
LLVM
```

这一阶段复杂度明显更高，不建议和第一阶段混在一个 CI 中。

---

## 14. 推荐执行顺序

1. 将独立 workflow 推送到默认分支，手动触发。
2. 通过构建配置、DRI 后端、AArch64、GLIBC ≤ 2.28、符号和动态加载检查。
3. 下载 tar 包与校验文件，校验后安装到专用 `/opt/mesa21`。
4. 确认目标机依赖、DRI 目录、AMD DRM 节点，运行 GBM 功能探针。
5. 相同浏览器版本分别用全新 profile 运行系统库和私有库两轮测试。
6. 保存完整日志、GPU 子进程映射、renderer 和白屏复现结果。
7. 根据已定位的失败环节决定后续工作。

## 15. 验收记录与验证边界

| 项目 | 必须记录的内容 |
| --- | --- |
| 构建 | Actions run 链接、源码 SHA256、构建修正、产物 SHA256 |
| 环境 | UOS/内核/架构/glibc、libdrm/Mesa 包版本、DRI 目录、DRM 节点 |
| 浏览器 | 实际启动命令、准确版本、安装包来源 |
| GBM 探针 | 完整日志、退出码、设备创建与 per-plane FD 导出结果 |
| 对照 | 同一复现页面、两轮启动参数和新 profile、完整日志 |
| 实际路径 | GPU 子进程 PID、GBM/DRI/libdrm 映射、renderer |
| 结果 | 白屏是否复现、两条错误是否出现、GPU 进程是否稳定 |

**本说明的状态：** workflow 和探针是候选实现，尚未执行 ARM64 CI 或 UOS 实机验收。静态检查、CI 构建成功、探针成功与 Chrome 硬件路径恢复是不同层级的证据，应分别报告。

参考依据：

- [Mesa 21.1.8 发布说明与源码校验值](https://docs.mesa3d.org/relnotes/21.1.8.html)
- [Chromium 当前 GBM 导出实现](https://chromium.googlesource.com/chromium/src/+/HEAD/ui/gfx/linux/gbm_wrapper.cc)；实际安装包需按版本核对。
- [Mesa 驱动安装与搜索路径说明](https://docs.mesa3d.org/faq.html)
- [GitHub workflow_dispatch 条件](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#onworkflow_dispatch)
