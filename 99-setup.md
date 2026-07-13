# 附录：开发环境搭建

## 1. 安装基础依赖

```bash
sudo apt update
sudo apt install build-essential pkg-config libglib2.0-dev
```

## 2. 编译安装 libglibutil

libgbinder 依赖 Jolla 的 `libglibutil`（提供 `gutil_log.h`、`gutil_hexdump` 等）。

```bash
cd ~/GITROOT
git clone https://github.com/sailfishos/libglibutil.git
cd libglibutil
make
sudo make install-dev
sudo ldconfig
```

验证：

```bash
pkg-config --cflags --libs libglibutil
# 应输出类似: -I/usr/local/include/gutil -lglibutil
```

## 3. 编译 libgbinder

```bash
cd ~/GITROOT/gbinder-note/libgbinder-master
make
```

验证：

```bash
ls build/debug/*.so
# 应出现 libgbinder.so
```

