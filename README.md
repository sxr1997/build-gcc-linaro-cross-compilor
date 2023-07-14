# Build Cross-compilor gcc-linaro from Source（从源码编译gcc-linaro cross-compilor）

Ref1: <https://blog.csdn.net/ComputerInBook/article/details/105510452>

Ref2: <https://blog.csdn.net/yangtze_1006/article/details/46740911>

Ref3: <https://preshing.com/20141119/how-to-build-a-gcc-cross-compiler/>

编译环境：

```bash
$ cat /proc/version
Linux version 5.15.0-76-generic (buildd@bos02-arm64-019) (gcc (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #83-Ubuntu SMP Thu Jun 15 19:21:56 UTC 2023
```

## 下载必须的Packages

```bash
wget http://ftpmirror.gnu.org/binutils/binutils-2.40.tar.gz
wget http://ftpmirror.gnu.org/gcc/gcc-5.4.0/gcc-5.4.0.tar.gz
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.17.2.tar.xz
wget http://ftpmirror.gnu.org/glibc/glibc-2.24.tar.xz
wget http://ftpmirror.gnu.org/mpfr/mpfr-3.1.5.tar.xz
wget http://ftpmirror.gnu.org/gmp/gmp-6.1.2.tar.xz
wget http://ftpmirror.gnu.org/mpc/mpc-1.0.3.tar.gz
wget ftp://gcc.gnu.org/pub/gcc/infrastructure/isl-0.12.2.tar.bz2
wget ftp://gcc.gnu.org/pub/gcc/infrastructure/cloog-0.18.1.tar.gz
```

版本应该可以不同，可以按需变更。Ref3说有些Packages也可以用包管理器下载，这里为了统一处理全部源码编译。

下载完成后在```${BUILD_ROOT}```目录下解压所有文件。

```bash
for f in *.tar*; do tar xf $f; done
```

## 添加安装目录到环境变量

```bash
vi /etc/profile
```

加入如下内容

```bash
export PATH=/usr/local/arm-linux/bin:$PATH
```

退出文件后不要忘了

```bash
source /etc/profile
```

## 安装前面下载的几个Packages

其实步骤都差不多，无非是配置(congigure)->编译(compile)->安装(install)。注意一下安装顺序。

### 1. gmp

```bash
cd $BUILD_ROOT/gmp-6.1.2
./configure --prefix=/usr/local/gmp-6.1.2
make -j8
sudo make install
```

### 2. mpfr

```bash
cd $BUILD_ROOT/mpfr-3.1.5
./configure --prefix=/usr/local/mpfr-3.1.5 --with-gmp=/usr/local/gmp-6.1.2
make -j8
sudo make install
```

### 3. mpc

```bash
cd $BUILD_ROOT/mpc-1.0.3
./configure --prefix=/usr/local/mpc-1.0.3 --with-gmp=/usr/local/gmp-6.1.2 --with-mpfr=/usr/local/mpfr-3.1.5 
make -j8
sudo make install
```

### 4. isl

```bash
cd $BUILD_ROOT/isl-0.12.2
./configure --prefix=/usr/local/isl-0.12.2 --with-gmp-prefix=/usr/local/gmp-6.1.2
make -j8
sudo make install
```

### 5. cloog

```bash
cd $BUILD_ROOT/cloog-0.18.1
./configure --prefix=/usr/local/cloog-0.18.1 --with-gmp-prefix=/usr/local/gmp-6.1.2
make -j8
sudo make install
```

## 编译binutils

需要主机gcc，自测高版本也可以编译通过，如果有问题建议降到gcc5以下。

```bash
mkdir ${BUILD_ROOT}/build_binutils
cd ${BUILD_ROOT}/build_binutils
../binutils-2.40/configure --prefix=/usr/local/arm-linux --build=$MACHTYPE --target=arm-linux-gnueabihf --enable-shared --disable-multilib
make -j8
sudo make install
```

然后就可以在```/usr/local/arm-linux/bin```目录下看到目前安装好的交叉编译工具。

## 编译GCC：生成c、c++交叉编译工具（这里把GCC分成三部分编译1/3）

```bash
mkdir ${BUILD_ROOT}/build_gcc
cd ${BUILD_ROOT}/build_gcc
../gcc-linaro-4.9-2017.01/configure --prefix=/usr/local/arm-linux --build=$MACHTYPE --target=arm-linux-gnueabihf --with-gmp=/usr/local/gmp-6.1.2 --wiht-mpfr=/usr/local/mpfr-3.1.5 --with-mpc=/usr/local/mpc-1.0.3 --with-isl=/usr/local/isl-0.12.2 --with-cloog=/usr/local/cloog-0.18.1 --disable-multilib --enable-languages=c,c++ --with-float=hard
make -j8 all-gcc
sudo make install-gcc
```

可以在```/usr/local/arm-linux/bin```目录下观察到新增了一些交叉编译工具。

## 编译Linux Kernel

```bash
cd ${BUILD_ROOT}/linux-3.17.2
make ARCH=arm headers_check
sudo make ARCH=arm INSTALL_HDR_PATH=/usr/local/arm-linux-gnueabihf headers_install
```

## 编译glibc，分成两次编译（1/2）

```bash
mkdir ${BUILD_ROOT}/build-glibc
cd ${BUILD_ROOT}/build-glibc
../glibc-2.24/configure --prefix=/usr/local/arm-linux/arm-linux-gnueabihf --build=$MACHTYPE --host=arm-linux-gnueabihf --target=arm-linux-gnueabihf --with-headers=/usr/local/arm-linux/arm-linux-gnueabihf/include libc_cv_forced_unwind=yes --disable-werror
sudo make install-bootstrap-headers=yes install-headers
make -j8 csu/subdir_lib
sudo install csu/crt1.o csu/crti.o csu/crtn.o /usr/local/arm-linux/arm-linux-gnueabihf/lib
sudo arm-linux-gnueabihf-gcc -nostdlib -nostartfiles -shared -x c /dev/null -o /usr/local/arm-linux/arm-linux-gnueabihf/lib/libc.so
sudo touch /usr/local/arm-linux/arm-linux-gnueabihf/include/gnu/stubs.h
```

## 第二次编译GCC（2/3）

```bash
cd ${BUILD_ROOT}/build-gcc
make -j8 all-target-libgcc
sudo make install-target-libgcc
```

## 第二次编译glibc（2/2）

```bash
cd ${BUILD_ROOT}/build-glibc
sudo make -j8
sudo make install
```

## 第三次编译GCC（3/3）

```bash
cd ${BUILD_ROOT}/build-gcc
make -j8
sudo make install
```

完成，可以使用以前编写的程序、u-boot和linux kernel试验一下。

## Known issues

### 编译GCC：生成c、c++交叉编译工具（Unable to find a usable ISL）

配置时有如下错误信息：

```text
configure: error: Unable to find a usable ISL. See config.log for details.
```

You can check config.log for other sulotions. Anyway, here is mine.

```bash
sudo vim /etc/ld.so.conf.d/ld.isl.so.conf
# Add '/usr/local/isl-0.12.2/lib' to this file.
sudo ldconfig
```

Any problems like this can be solved by this procedure.

### 编译Linux Kernel（./usr/include/linux/kexec.h:61: userspace cannot reference function or variable defined in the kernel）

执行```make ARCH=arm headers_check```命令后有如下信息：

```text
./usr/include/linux/kexec.h:61: userspace cannot reference function or variable defined in the kernel
```

似乎是一个很有历史的非恶性BUG，可以忽略。
