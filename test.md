aFHIaims 的程序优化

[第一性原理code的最新总结和进展](https://sci.scientific-direct.net/view_online.asp?1652737&0541c208c5cdc790&18)

https://awards.acm.org/bell/covid-19-nominations

# 需求分析

## 任务目标

+ 把 `src/DFPT_dielectric/sum_up_whole_potential_shanghui_dielectric.f90` 里面调用的 subroutine 用 C 语言重写（fortran 项目调用 C 语言写的函数）
+ porting 到神威超算的从核上

## 环境搭建测试

编译 make scalapack.mpi -j 8 生成可执行文件

+ OS：把项目 clone 到 Linux 环境下
+ 编译器：【intel 全家桶】 或 【gfortran + openMPI 】

测试代码：mpirun 即可，测试文件在对应 make 目录下

## 具体翻译列表

426-791,796-877

+ 仅翻译 i_batch loop 里面的函数，外部的不翻
+ 与 non_peridic_ewald 相关的也不翻译

| 翻译 | 单元测试 | 函数名                                                 | 负任人 |
| ---- | -------- | ------------------------------------------------------ | ------ |
| 已展 | 已测     | **integrate_delta_v_hartree**                          | 吴     |
| 已翻 | 已测     | spline_angular_integral_log                            | 吴     |
| 已翻 | 已测     | spline_vector_v2                                       | 段     |
| 已翻 | 已测     | invert_radial_grid                                     | 吴     |
| 已翻 | 已测     | compensating_density                                   | 吴     |
| 已翻 | 已测     | cubic_spline_v2                                        | 吴     |
| 已翻 | 已测     | integrate_delta_v_hartree_internal                     | 吴     |
| 已翻 | 已测     | **far_distance_hartree_Fp_cluster_single_atom_p2**     | 吴     |
| 已翻 | 已测     | **far_distance_hartree_Fp_periodic_single_atom**       | 吴     |
| 已翻 | 已测     | **far_distance_real_hartree_potential_single_atom**    | 吴     |
| 已翻 | 已测     | **far_distance_real_hartree_potential_single_atom_p2** | 吴     |
| 已翻 | 已测     | **get_rho_multipole_spl**                              | 吴     |
| 已翻 | 已测     | **update_hartree_potential_recip**                     | 吴     |
| 不翻 |          | **aims_shm_get**                                       |        |
| 不翻 |          | **calculate_extended_hartree_non_periodic_ewald**      |        |
| 不翻 |          | **evaluate_hartree_recip_coef**                        |        |
| 不翻 |          | **hartree_potential_real_coeff**                       |        |
| 不翻 |          | **interpolate_extended_hartree_non_periodic_ewald**    |        |
| 不翻 |          | **localorb_info**                                      |        |
| 不翻 |          | **update_outer_radius_l**                              |        |

integrate_first_order_H_dielectric.f90

| 翻译 | 单元测试 | 函数名                                 | 负责人 |
| ---- | -------- | -------------------------------------- | ------ |
| 已翻 |          | prune_density_matrix_sparse_dielectric | 罗     |
| 已翻 |          | prune_radial_basis_p2                  | 罗     |
| 已翻 |          | tab_atom_centered_coords_p0            | 罗     |
| 已翻 |          | evaluate_radial_functions_p0           | 罗     |
| 已翻 |          | prune_basis_p2                         | 罗     |
| 已展 |          | tab_trigonom_p0                        | 吴     |
| 已展 |          | tab_wave_ylm_p0                        | 吴     |
| 已翻 |          | evaluate_waves_p2                      | 吴     |
| 已翻 |          | evaluate_xc_DFPT                       | 吴     |

## 参数解释

#### 命名规则

+ 变量：函数的参数命名（假设 fortran 原有命名为 `fname`）
  + 数组：采用 fortran 原有命名
  + 原子变量(int/real)：在 fortran 原有命名加上前缀 `p_`，例如 `p_fname`
+ 函数：在 fortran 函数名加上后缀 `_c_`，例如 `fun() => fun_c_`
+ 文件：在 fortran 文件名加上后缀 `_c`，例如 `file.f90 => file_c.c`

#### 变量上限

| 变量名              | 所属函数                                              | 取值上限 |
| ------------------- | ----------------------------------------------------- | -------- |
| n_compute_fns       | evaluate_waves_p2_c_                                  |          |
| l_pot_max           | far_distance_real_hartree_potential_single_atom_p2_c_ |          |
| p_max               | f_erfc_table_original_c_                              |          |
| k_points_max[0,1,2] | update_hartree_potential_recip_c_                     |          |
| n_basis_fns         | evaluate_radial_functions_p0_c_                       |          |



## F-C 翻译

提问：Fortran 在数值计算是比 C 快的，主要因为 aliasing 机制，导致 C 不能做到极致的优化，那为何要将 Fortran 翻译为 C 代码呢？

回答：C 在 神威从核上的加速效果远大于 Fortran，因此，翻译后的总体加速效果将更加可观！

### fortran 调用 C 语言函数规则

+ fortran 的子程序命名为 `*_c`，例如 `foo_c`
+ C 语言的函数命名为 `*_c_`，例如 `foo_c_`

### 翻译流程

+ 将 `sum_up_whole_potential_shanghui_dielectric.f90` 中对应的子程序调用（形如 `sub`） 改为 `sub_c` 
+ `sub_c` 增加到对应目录下
+ **修改 src/makefile.backend** 
+ 正常 `make scalapack.mpi -j 8`，生成可执行文件
+ 测试即可

### 翻译流程实例

以下为实例演示：用 C 语言重写 `spline_vector_v2`

+ 修改 `src/DFPT_dielectric/sum_up_whole_potential_shanghui_dielectric.f90`  中的`spline_vector_v2` 为 `spline_vector_v2_c`
+ 用 C 语言重写该函数，保存于 `spline_c.c` 文件中（可用其它命名）
+ 将 `spline_c.c` 置于 `spline.f90` 所在的目录 ` src/basis_sets/` 下（也可以放在其它目录下）
+ 在 `src/Makefile.backend` 中 `C_FILES = \` 下面增加`$(OBIDIR)$/basis_sets/spline_c.o`（`basis_sets ` 为 .c 文件存放目录），结果如下：

```makefile
C_FILES = $(OBJDIR)/get_stacksize.o \
          $(OBJDIR)/check_stacksize.o \
          $(OBJDIR)/change_directory.o \
          $(OBJDIR)/change_directory_c.o \
          $(OBJDIR)/unlimit_stack.o \
          $(OBJDIR)/basis_sets/spline_c.o // 在这增加，$(OBJDIR)表示src的路径
```

+ 在 src 下  `make scalapack.mpi -j 8`，生成可执行文件（ `-j 8` 表示 8 个线程同时编译）
+ 对比测试

# 环境搭建

有两种环境搭配选择，商业版 intel 全家桶和开源软件一条龙。

## Intel 全家桶

环境组合：**debian9 + intel parallel studio 2015**（debian10 和 2015有些不兼容，可将 mpiicc 改为 mpicc）

采用离线安装包的安装形式，注意不可直接将文件拖拽到虚拟机或远程主机上，会照成压缩文件损坏，最好用 FTP 传输文件。（源项目代码也是同理，最好不要使用拖拽方式传输文件）

解压后，进入主目录，输入 `./install.sh` 跟着安装程序进行安装即可。 

### ~/.bashrc 环境变量

需要配置以下内容的环境变量

+ icc
+ ifort
+ mkl
+ mpi：mpiifort，mpiicc，mpicc

```shell
# ~/.bashrc 用下面的
source /opt/intel/composer_xe_2015.0.090/bin/compilervars.sh intel64
export PATH=/opt/intel/impi/5.0.1.035/bin64:$PATH
```

### Makefile / make.sys

关键只需改一句：`my_mkl=/opt/intel/composer_xe_2015.0.090/mkl`，设置 `mkl` 库的路径，完整 `Makefile` 文件中的编译相关内容如下。但是强烈推荐将以下内容写入 `make.sys` 文件，然后将该文件放在 `src/` 目录下，效果与修改 `Makefile` 一样。

```makefile
FC = ifort
 FFLAGS = -O3 -ip -module $(MODDIR) -fp-model precise
#---------may be faster, we can try---------------------------
# FFLAGS = -xSSE4.2 -fp-model  precise -O3 -ip -module $(MODDIR)
 F90FLAGS = $(FFLAGS)
 ARCH = arch_generic.f90
 MPIFC = mpiifort
 USE_MPI      = yes
 my_mkl=/mnt/c/Users/shanghui/Desktop/linux_files/parallel_studio_xe_2015_install/composer_xe_2015.0.090/mkl
 LAPACKBLAS   = -L$my_mkl/lib/intel64 -lmkl_intel_lp64 -lmkl_core -lmkl_sequential -Wl,-rpath,$my_mkl/lib/intel64  -I$my_mkl/include/intel64/
 SCALAPACK    = -L$my_mkl/lib/intel64 -Wl,--start-group -lmkl_scalapack_lp64 -lmkl_blacs_intelmpi_lp64  -Wl,--end-group
#ARCHITECTURE = Generic
 CC           = mpiicc # debian 10 使用 intel mpiicc 会出现libc库不兼容问题,可改为 mpicc
 CFLAGS       = -c -std=c99 # -std=c99 指定C语言标准为c99
 SHM          = shm.o /usr/lib64/librt.a
```

apt-get install build-essential libscalapack-mpi-dev

```shell
# 查看 build-essential 的依赖(包含 libc6-dev,gcc,g++,make,dpkg-dev)
apt-cache depends build-essential
# 安装
sudo apt-get install build-essential -y
```

## gfotran + gcc

**ifort + icc 编译的程序出现无故的错误，可以试试 gfortran + gcc 的组合，特别是本来能跑的，换个地方跑不了的程序**。

#### debian 安装依赖

```shell
apt-get install libscalapack-openmpi-dev libopenblas-dev liblapack-dev 
```

#### 天河版 make.sys

```shell
FC=gfortran
#FFLAGS = -O3 -ffree-line-length-none
#--debug--
FFLAGS = -O0 -g -ffree-line-length-none 

F90FLAGS = $(FFLAGS)
ARCH = arch_generic.f90
MPIFC = /WORK/app/MPICH/mpich3.1.3-gcc4.9.2/bin/mpif90
USE_MPI = yes

CC = gcc
CFLAGS = -c -std=gnu99 -g -O0

LAPACKBLAS = -L/WORK/app/LAPACK/new/lapack-3.8.0 -llapack -L/WORK/app/LAPACK/new/lapack-3.8.0 -lrefblas
SCALAPACK = -L/WORK/app/scalapack/2.0.2-gcc-4.9.2/lib -lscalapack

SHM = shm.o /usr/lib64/librt.a
```



## 开源软件

debian9 开源一条龙编译，安装依赖如下：

```shell
apt-get install libscalapack-openmpi-dev libopenblas-dev liblapack-dev
# 海文
sudo apt-get install libopenmpi-dev openmpi-bin
sudo apt-get install libscalapack-openmpi-dev
apt-get install libblas-dev
sudo apt-get install libblas-dev liblapack-dev
```

编写 `make.sys`，放在 `src/` 下

```makefile
 FC = gfortran
#FC = swafort
# FFLAGS = -O3 -ffree-line-length-none -mieee
 FFLAGS = -O3 -g -ffree-line-length-none
 F90FLAGS = $(FFLAGS)
 ARCH = arch_generic.f90
 MPIFC = mpif90
 USE_MPI      = yes
 LAPACKBLAS = -llapack -lblas
 SCALAPACK =   -lscalapack-openmpi
 CC           = mpicc # debian 10 使用 intel mpiicc 会出现libc库不兼容问题 
 CFLAGS       = -c
 SHM          = shm.o /usr/lib64/librt.a
```

# 编译运行

## 编译

### 编译步骤

1. `cd ~/src`：进入项目的主目录

2. `make clean`：清理项目生成的文件
3. `make scalapack.mpi -j 8`：编译链接生成可执行文件 `scalapack.mpi`，`-j 8` 表示 8 个线程进行编译

### 问题记录

- 从 gitee 直接下载的 zip 包是没有 .git 记录的，必须在本地 git clone，然后在上传到超算上

+ 项目一定不要拖拽的方式传入虚拟机，务必使用 `git clone` 或者 `ftp` 传输

+ 使用 `ftp` 传输项目，可能会遇见 `xxx.pl` 脚本无权限问题
  + 先 `make clean`
  + 然后 `chmod a+x version_stamp _writer.pl`（授权）

+ 编译文件 mpiicc fhi-aims_link/fhi-aims_20191127_swgemm/src/external/libxc-4.0.2/gga_c_bcgp.c 时报错如下：

  ```
  /home/user/intel/compilers_and_libraries_2018.2.199/linux/compiler/include/math.h(1230): error: identifier "_LIB_VERSION_TYPE" is undefined
    _LIBIMF_EXTERN_C _LIB_VERSIONIMF_TYPE _LIBIMF_PUBVAR _LIB_VERSIONIMF;
                     ^
  ```

  分别将 `make.sys` 中的 `CC` 改为以下四种编译器，尝试结果如下：

  - mpiicc 和 icc 报错相同
  - mpicc 和 gcc 没问题

  [报错参考](https://blog.csdn.net/donkeydog/java/article/details/84312795);

  主要是操作系统的 libc 版本与 Intel parallel studio 2015 不兼容，libc-2.7 改了很多东西，glibc 2.27 删除了 `_LIB_VERSION_TYPE`，解决方案有三种：

  + 升级 intel parallel studio 到 2018 或更高版本
  + 降级操作系统版本
  + 令 `CC=mpicc` ，更换编译器

+ > icc error #10106

  - 从 Makefile 开始查找，看包含的所有 Makefile 文件
    - Makefile.backend
    - Makefile.moddeps （自动生成）
    - 发现仅 Makefile 中包含 mpiicc，可能是 mpiicc 调用了 icc
  - intel 环境配置

  /home/export/online3/para030/wuyangjun/00_intel/COM_L___LJDD-J34GX67J.lic

  缺少 32-bit libraries，安装 libstdc++.i386

  缺少 g++，需额外安装，yum install gcc-c++

  

  把配置项写在 make.sys 文件中，就可不用修改 makefile


# Linux 系统命令

[Linux 命令大全](https://man.linuxde.net/apt-get-2)

## debian

### 包管理器 apt 

#### 源配置

[apt 国内清华镜像源配置](https://mirror.tuna.tsinghua.edu.cn/help/debian/)：提高配置速度

+ debian 详细信息查询：cat /etc/os-release

  查看 debian 版本号：more /etc/debian_version

+ 在网站选择对应版本号，复制对应地址到 `/etc/apt/source.list` 中，buster 版本如下：

```
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ buster main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-updates main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-updates main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-backports main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-backports main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security buster/updates main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian-security buster/updates main contrib non-free
```

#### 常用命令

```
# 更新apt源
apt-get update
# 搜索可安装的包
apt-cache search 包名
# 安装包
sudo apt-get install 包名
# 卸载软件包（包括配置）
apt-get remove --purge softname
```

### 软链接

在 WSL 下创建软链接 00_intel_link，直接指向 Windows D盘的文件（其它 OS 方法一致）

```shell
ln -s /mnt/d/Desktop/00_intel/ 00_intel_link
```

## 超算

广州天河超算：切换到对应结点才能编译运行： ssh -p 5577 lon1

查看磁盘容量：`lfs quota -uh grid225 /PARA/grid225`，`grid225` 为用户名

# 编译&链接

+ 编译时指定头文件
+ 链接时指定静态库或动态库

## 编译

### 头文件搜索

编译时指定头文件。

C 语言包含头文件的两种书写格式：

- `#include <head.h>`：直接到系统指定的某些目录去查找头文件
- `#include "head.h"`：现在源文件所在的同级目录查找，然后再到系统指定的某些目录去查找头文件

#### 实例验证

linux 环境下，创建一个 `test_include` 目录，在该目录下创建两个文件，`add.h` 和 `main.c`，内容如下：

`add.h` 文件定义两个整型全局变量

```C
// add.h
int global_a;
int global_b;
```

`main.c` 文件包含 `add.h` 并打印它们的加法过程

```C
#include <stdio.h>
#include "add.h"
// #include <add.h>

int main() {
    global_a = 2;
    global_b = 3;
    printf("%d + %d = %d\n", global_a, global_b, global_a + global_b);
    return 0;
}
```

编译执行过程，结果如下，可见其是正确的。因为 `include "add.h"` 会让 GCC 编译器先在 `test_include` （main.c 所在目录）下查找 `add.h`，所以就找到了

```shell
$ gcc main.c
$ ./a.out 
2 + 3 = 5
```

若将头文件包含写成 `include <add.h>` 形式，编译错误的结果如下，说明编译器无法找到头文件，因为 GCC 默认直接去系统的指定目录查找，而里面必然是没有 `add.h` 头文件的，于是出现报错

```shell
$ gcc main.c 
main.c:3:17: fatal error: add.h: No such file or directory
 #include <add.h>
                 ^
compilation terminated.
```

#### 搜索顺序

1. 通过 `-I` 指定的路径
2. GCC 的环境变量 `C_INCLUDE_PATH/CPLUS_INCLUDE_PATH/OBJC_INCLUDE_PATH`
3. 系统指定的搜索目录
   + `/usr/include`
   + `/usr/local/include`
   + `/usr/lib/gcc-lib/i386-linux/2.95.2/include`（和 gcc-lib 所在路径相关）

### 常见编译选项

```shell
-module path # 指定模块文件生成和查找的路径
           specify path where mod files should be placed and first location to
           look for mod files
-g[level] # 生成调试信息
          Produce symbolic debug information.
          Valid [level] values:
             0  - Disable generation of symbolic debug information.
             1  - Emit minimal debug information for performing stack traces.
             2  - Emit complete debug information. (default for -g)
             3  - Emit extra information which may be useful for some tools.

# 程序间优化选项
Interprocedural Optimization (IPO) 
	# 单文件内部优化
	-[no-]ip  enable(DEFAULT)/disable single-file IP optimization within files 

# 浮点数优化选项
Floating Point 
    -fp-model <name>         
              enable <name> floating point model variation
                [no-]except - enable/disable floating point semantics
                fast[=1|2]  - enables more aggressive floating point optimizations
                precise     - allows value-safe optimizations
                source      - enables intermediates in source precision
                sets -assume protect_parens for Fortran
                strict      - enables -fp-model precise -fp-model except, disables
                contractions and enables pragma stdc fenv_access

# 链接选项
Linking/Linker
	-L<dir>   instruct linker to search <dir> for libraries
	-l<string> instruct the linker to link in the -l<string> library
	-static   prevents linking with shared libraries
	-shared   produce a shared object
	# -Wl,para1,para2 表示将 para1 和 para2 可选项传递给链接器
	-Wl,<o1>[,<o2>,...] pass options o1, o2, etc. to the linker for processing
```



## 链接-库搜索

链接时指定库。

### 基本指令

[链接选项](https://blog.csdn.net/jiafu1115/article/details/8842240?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)

- 库
  - `L`（大写 L）：后跟库文件（静态/动态）的搜索路径（默认搜索路径为 `lib`,`/usr/lib`,`usr/local/lib`）
  - `l`（小写 L）：后跟需要链接的库名称，如库名为 `libm.so`，那么链接写成 `-lm` 即可，因为 `lib` 会自动加上
- 头文件
  - `I`（大写 i）：后跟头文件搜索目录

举例：

```
-L /home/hello/lib 表示将 /home/hello/lib 目录作为第一个寻找库文件的目录，寻找的顺序是：
/home/hello/lib-->/lib-->/usr/lib-->/usr/local/lib

-lworld表示在上面的lib的路径中寻找libworld.so动态库文件（如果gcc编译选项中加入了“-static”表示寻找libworld.a静态库文件）

-I/home/myinclude 链接时会去 /home/myinclude 下查找头文件
```

libc 和 glibc 都是 linux 下 C 语言的函数库（最底层的 API）

libc 是 ANSI 标准，glibc 是 GNU 标准

## Makefile

自动化项目构建工具，写上编译和链接的依赖。

[Makefile 学习参考](https://blog.csdn.net/twc829/article/details/72729799)

[阮一峰简明 Makefile 入门](https://www.ruanyifeng.com/blog/2015/02/make.html) 

[GNU Makefile 手册](https://www.gnu.org/software/make/manual/make.html)

[跟我一起写 Makefile](https://seisman.github.io/how-to-write-makefile/index.html)

`?=` ：左侧变量为空才进行赋值

回声：每个目标后的命令后先输出本身，再打印执行结果，如下实例，执行 `make clean` 时，命令行会先输出 `rm *` ，然后在执行它，在它前面加上 `@` 可以不输出本身

```makefile
# Makefile
clean:
	rm test.c # 会被输出
	# 虽然我是注释，但依旧会被输出
	@ # 我是注释，但我最前面有 @，所有不会被输出
```

输入命令 `make clean` 后，输出如下结果：

```shell
rm test.c  # 会被输出
# 虽然我是注释，但依旧会被输出
```

`make -f makefile_name` 可指定 `makefile_name` 为 `Makefile` 文件

自动变量：

```makefile
$@ # 当前的目标，如 make target,目标为 target
$< # 当前目标的第一个前置规则
```

# Fortran To C

[oracle参考](https://docs.oracle.com/cd/E19205-01/820-1204/aeuku/index.html)：包含数组，数据类型的转换说明。

fortran 参数均为传递地址，比如 `f(a(2))`，传递的是 `a[2]` 的地址，可以看成向量。

```fortran
! module_B 使用 module_A 中的变量，vB=>vA 表示将 模块A的vA改名为vB
use module_A, only vB=>vA, vB2=>vA2, vA3
```



| 对比项目 | Fortran                      | C                            |
| -------- | ---------------------------- | ---------------------------- |
| 数组维度 | 列顺序优先（靠左下标变化快） | 行顺序优先（靠右下标变化快） |
| 数组起点 | 1 开始                       | 0 开始                       |
| 函数传参 | 引用（传址）                 | 传值/传址                    |

+ 函数传递参数时，C 语言的参数均写成一维指针，这样 Fortran 的高维数组可直接转换为一维的数组（在内存中本就是一维线性存放），注意 C 是行优先变化

## 复数

C 语言用 `struct` 存储实部和虚部，承接 fortran 的 complex。

**易错点**：`complex*16` 等价于 `complex(kind=8)`，`*16` 表示总体占用的内存为 16B，`kind=8` 表示最小单位占用内存为 8B

### 翻译实例演示

#### complex.c

```C
#include <stdio.h>
#include <math.h>

typedef struct {
    // float dr, di;
    double dr, di; // double 对应 complex*16 或者 complex(kind=8)
} complex;

void c_complex_c_(complex *c)
{
    // printf("c_complex_c = (%f, %f)\n", c->dr, c->di);
    printf("c_complex_c = (%lf, %lf)\n", c->dr, c->di);
}
```

#### complex.f90

```fortran
program test_complex
    implicit none
    
    complex :: w1
    complex*8 :: w2
    complex*16 :: w3
    complex(kind=8) :: w4
    complex(kind=16) :: w5

    w3 = (2.0,1.0)
    w4 = (2.0,1.0)
    
    print *, kind(w1), kind(w2), kind(w3), kind(w4), kind(w5)
    call c_complex_c(w3)
    call c_complex_c(w4)
    
end program test_complex
```

#### 编译运行

```shell
$ gfortran complex.f90 complex.c
$ ./a.out 
 4           4           8           8          16
c_complex_c = (2.000000, 1.000000)
c_complex_c = (2.000000, 1.000000)
```

### 常用运算

指数表示：$e^{x+iy}=e^xcosy+(e^xsiny) i$

fortran 常用

```fortran
exp((2,1)) = (3.99232411,6.21767616)
```

翻译为 C 语言必须用以上公式进行计算，得到对应实部和虚部。

加法：` (a+bi)+(c+di)=(a+c)+(b+d)i`

乘法：` (a+bi)(c+di)=(ac-bd)+(bc+ad)i`

乘方：`(x+yi)^n = r^n * cos(n*theta) + r^n*sin(n*theta)*i, theta=arctan(y/x), r = sqrt(x^2+y^2)`

#### 乘方和乘法的 C 语言实现

complex.c

```c
#include <stdio.h>
#include <math.h>

typedef struct {
    // float dr, di;
    double dr, di;
} complex;

// res = c ^ n
// (x+yi)^n = r^n * cos(n*theta) + r^n*sin(n*theta)*i, theta=arctan(y/x), r = sqrt(x^2+y^2)
void complex_pow_c_(const complex *c, int *n, complex *res)
{
    double r2 = c->dr * c->dr + c->di * c->di;
    double rn = pow(r2, (double)*n / 2);
    double theta = atan(c->di / c->dr);
    res->dr = rn * cos(*n * theta);
    res->di = rn * sin(*n * theta);
}

// res = c1 * c2
// (x1+y1*i)*(x2+y2*i) = (x1*x2 - y1*y2) + (x1*y2 + x2*y1)i
void complex_multiply_c_(const complex *c1, const complex *c2, complex *res)
{
    res->dr = c1->dr * c2->dr - c1->di * c2->di;
    res->di = c1->dr * c2->di + c2->dr * c1->di;
}
```

complex.f90

```fortran
program test_complex
    implicit none
    
    complex*16 w(4)
    w(1) = (2,1)
    w(2) = (4,3)
    call complex_multiply_c(w(1), w(2), w(3))
    print *, 'multiply origin = ', w(1) * w(2), 'C = ', w(3)

    call complex_pow_c(w(1), -3, w(3))
    print *, 'pow  origin = ', w(1) ** (-3), 'C = ', w(3)

end program test_complex
```

编译运行：

```shell
$ gfortran complex.f90 complex.c
$ ./a.out 
 multiply origin = (5.0000000000000000,10.000000000000000) C =                (5.0000000000000000,10.000000000000000)
 pow  origin =   (1.60000000000000073E-002,-8.80000000000000226E-002) C =   (1.60000000000000107E-002,-8.79999999999999949E-002)
```

## 数组

### 可变下标

#### 基础概念

在 `fortran` 中，每个维度的下标可任意指定起止的范围

```fortran
data_type :: arr(l:r) ! 表示数组 arr 的下标范围是 [l,r]（左闭右闭）
data_type :: arr(len) ! 表示数组 arr 的下标范围是 [0,len]（左闭右闭，参数仅一个默认长度）
```

以下多举几个实例：

```fortran
integer :: a(0:3) 	! [0,3]
integer :: b(-1:2) 	! [-1,2]
integer :: c(4)		! [1,4]
```

#### 实例验证

`idx.f90` 内容如下：

```fortran
program idx
    implicit none
    integer :: a(0:3)
    integer :: b(-1:2)
    integer :: c(4)

    a(:) = 10
    a(0) = 0
    a(3) = 3
    print *, 'a = ', a

    b(:) = 20
    b(-1) = -1
    b(2) = 2
    print *, 'b = ', b

    c(:) = 30
    c(1) = 1
    c(4) = 4
    print *, 'c = ', c

end program idx
```

输出结果为：

```shell
 a =            0          10          10           3
 b =           -1          20          20           2
 c =            1          30          30           4
```



### 维度变化实例

#### C 语言

```C
#include <stdio.h>
int main() {
    int a[2][2] = { // 二维数组
        {1, 2},
        {3, 4}
    };
    int *b = (int *) a; // 强制转换为一维数组
    for (int i = 0; i < 2*2; i ++)
        printf("%d ", b[i]);
    puts("");

    int c[2][2][2] = { // 三维数组
        {
            {1, 2},
            {3, 4}
        },
        {
            {5, 6},
            {7, 8}
        }
    };

    int *d = (int *)  c; // 强制转换为一维数组
    for (int i = 0; i < 2*2*2; i ++)
        printf("%d ", d[i]);
    puts("");
    return 0;
}

/* 打印结果如下：
    1 2 3 4 
    1 2 3 4 5 6 7 8 
*/
```

#### Fortran

```fortran
program main
    implicit none
    integer :: a(2,2);
    a(:,:) = reshape([1, 2, 3, 4], [2,2])
    print *, a(1,1), a(1,2), a(2,1), a(2,2)
    print *, a
    b(:,1) = [1, 2]
    b(:,2) = [3, 4]
    print *, b(1,1), b(1,2), b(2,1), b(2,2)
    print *, b 
end program main
! 输出如下：
!           1           3           2           4
!           1           2           3           4
!           1           3           2           4
!           1           2           3           4
```

+ 逻辑模型：例如  `a[1][2]` 是通过逻辑模型访问数组元素的方式，仅仅为了便于人去记忆和使用
+ 内存模型：在内存中任意维度的数组均是一维线性存放，实际存放的方式

### 高维数组转换实例

全部转换为 C 语言中的一维数组，然后通过下标计算公式来访问。

`fortran` 三维数组

```fortran
real*8  :: spl_param(n_l_dim,n_coeff,n_grid_dim)
real*8  :: out_result(n_vector)
out_result(1:n_vector) = &
              spl_param(1:n_vector,1,i_spl  )*ta &
            + spl_param(1:n_vector,2,i_spl  )*tb &
            + spl_param(1:n_vector,1,i_spl+1)*tc &
            + spl_param(1:n_vector,2,i_spl+1)*td
```

转换为 C 语言

```C
// 调用处定义 real*8
typedef real double;
// spl_param 参数传递(fortran->C)时被强制转换为一维指针，逻辑模型=内存模型
// 列为主元进行地址转换，如逻辑坐标为(i,j,k),对应大小为(u1,u2,u3)，那么内存模型地址为u1*u2*k+u1*j+i
i_spl -= 1; // 下标均从 0 开始; (i,j,k)对应取值需从0开始
for (int i = 0; i < n_vector; i ++){
    out_result[i] = spl_param[(i_spl * n_coeff + 0) * n_l_dim + i]*ta +
        spl_param[(i_spl * n_coeff + 1) * n_l_dim + i]*tb +
        spl_param[((i_spl + 1) * n_coeff + 0) * n_l_dim + i]*tc +
        spl_param[((i_spl + 1) * n_coeff + 1) * n_l_dim + i]*td;
}
```

## logical 参数传递

[类型对应参考](https://docs.oracle.com/cd/E19205-01/820-1204/6nct259sj/index.html#aeuky)

| Fortran 95 数据类型 | C 数据类型      | 大小 | 对齐 |
| ------------------- | --------------- | ---- | ---- |
| LOGICAL x           | int x           | 4    | 4    |
| LOGICAL (KIND=1) x  | signed char x   | 1    | 4    |
| LOGICAL (KIND=2) x  | short x         | 2    | 4    |
| LOGICAL (KIND=4) x  | int x           | 4    | 4    |
| LOGICAL (KIND=8) x  | long long int x | 8    | 4    |

取值对应关系如下：

```fortran
logical :: lg = .true.  ! = int lg_c = 1; 这是错的，所有非0对应的都是.true.所有判断必须用0来处理
logical :: lg = .false. ! = int lg_c = 0;
```

### 源码组织

文件名：`logical.f90`

```fortran
program logical
    implicit none
    logical(4) :: lg

    lg = .true.
    print *, 'fortran lg = ', lg
    call print_lg_c(lg)
    
    lg = .false.
    print *, 'fortran lg = ', lg
    call print_lg_c(lg)

end program logical
```

文件名：`print_lg_c.c`

```C
#include <stdio.h>

void print_lg_c_(int* lg) {
    printf("lg = %d\n", *lg);
}
```

文件名：`Makefile`

```makefile
a.out: logical.o print_lg_c.o
	gfortran print_lg_c.o logical.o
logical.o: logical.f90
	gfortran -c logical.f90 -o logical.o
print_lg_c.o: print_lg_c.c
	gcc -c print_lg_c.c -o print_lg_c.o
clean:
	rm *.o *.out
```

### 编译运行

编译运行命令如下：

```shell
$ make clean
$ make
$ ./a.out
```

输出结果如下：

```shell
 fortran lg =  T
lg = 1
 fortran lg =  F
lg = 0
```

## 数据结构和指针

### 数据结构声明

```fortran
! 简单语法描述
type 结构体名
	变量类型1 :: 变量名1
	...
	变量类型n :: 变量名n
end type 结构体名

! 例如以下实例
type batch_of_points
    integer :: size
    integer :: batch_n_compute
end type batch_of_points
```

### 数据结构使用

```fortran
! 定义结构体变量
type(结构体名) :: 变量名
! 例子1, type之后，可加上基本变量的修饰符，如target，dimension，pointer等
type(batch_of_points) :: b

! 访问结构体成员
结构体变量名 % 成员变量名
! 例子2
b % size = 1
b = batch_of_points(1,10) ! 给整个结构体变量赋值
```

### 指针声明

指向普通变量/结构体的指针：

```fortran
! 类似C语言的引用，指针是变量别名
integer, pointer :: pa ! pointer 修饰指针
integer, target a ! 被指向的目标用 target 修饰
pa => a ! 给指针赋值

type(结构体名), pointer ::pds ! 过程和普通变量一致
```

指向数组的指针（普通变量数组/结构体数组）

```fortran
integer, pointer, dimension(:) :: parr1, parr2 ! 多个dimension修饰，可指向静态/动态数组
integer, target :: arr1(3) ! 静态或动态数组均可
integer, target, allocate(:) :: arr2
parr1 => arr1
parr2 => arr2
```

### 综合实例演示

#### ds.f90

```fortran
program ds
    implicit none
    type batch_of_points
            integer :: size
            integer :: batch_n_compute
    end type batch_of_points
    
    type(batch_of_points) :: b
    type(batch_of_points), target :: arr(2)
    type(batch_of_points), pointer, dimension(:) :: batches

    b % size = 10
    b % batch_n_compute = 20
    call test_ds_c(b)

    arr(1) = batch_of_points(11,21)
    arr(2) = batch_of_points(12,22)
    batches => arr
    call test_ds_c(batches(1))
    call test_ds_c(batches(2))

    call test_ds2_c(batches, 2)

end program ds
```

#### ds.c

```C
#include <stdio.h>
#include <stdlib.h>

typedef struct batch_of_points
{        
    int size;
    int batch_n_compute;
} batch_of_points;

void test_ds_c_(batch_of_points *b)
{
    if (b == NULL)
    {
        printf("batch_of_points NULL\n");
        exit(1);
    }
    puts("f2c: structure");
    printf("size = %d, batch_n_compute = %d\n", b->size, b->batch_n_compute);
}

void test_ds2_c_(batch_of_points *b, int *n)
{
    if (b == NULL)
    {
        printf("batch_of_points NULL\n");
        exit(1);
    }
    puts("\nf2c: structure pointer array");
    for (int i = 0; i < *n; i++)
    {
        printf("i = %d, size = %d, batch_n_compute = %d\n", i, b[i].size, b[i].batch_n_compute);
    }
}
```

#### 编译运行

```shell
$ gfortran ds.f90 ds.c
$ ./a.out 
f2c: structure
size = 10, batch_n_compute = 20
f2c: structure
size = 11, batch_n_compute = 21
f2c: structure
size = 12, batch_n_compute = 22

f2c: structure pointer array
i = 0, size = 11, batch_n_compute = 21
i = 1, size = 12, batch_n_compute = 22
```

### 坑点

只要结构体中包含 `pointer` 或 `allocatable` 关键字修饰的变量，那么 Fortran 无法直接传递给 C 语言。可用 gdb 逆向工程调试，可发现 `pointer` 和 `allocatable` 不是原子类型，而是一个有编译器定义好的结构体，C 语言没有对应的结构体承接，所以出现不兼容问题。

+ 构造结构体：可阅读 Fortran 编译器的源码相关定义，自行定义 C 语言结构体来承接这两种类型数据。
  + 优点：传递简单
  + 缺点：不通用，移植性差。每个编译器前端的实习可能都不同，如 gfortran 和 ifort
+ 抽取数据：把指针指向的数据取出来放在普通数组中。建议通过 loc 函数取出数据的地址，减小空间消耗

## 模块私有变量传递

问题描述：子程序 A 调用模块 MB 中的子程序  B，而且 B 中使用了 MB 的私有变量，如何改写为 C 语言函数（如何传递私有变量？）

### 解决方案

存在两种解决思路：

1. 设置同步 public 变量：在 MB 中新建一个 public 变量，对 public 变量操作完再赋值给 private 变量

2. 现在 MB 中将 private 变量传递给 C 语言的变量，再对 C 语言对应变量操作

### 方法二示例

#### 模块 private_trans.f90

```fortran
module private_trans
    implicit none
    real*8,allocatable,dimension(:,:),private:: Fp  ! private
    integer :: info

    contains
    
    subroutine init()
        implicit none
        ! allocate(Fp( 0:(l_pot_max+1), n_atom_list),stat=info)
        allocate(Fp(3,4), stat=info)
        Fp(1,1) = 11
        Fp(2,2) = 22
        Fp(3,4) = 34
        print *, Fp

        call trans_private_array_c(Fp)  ! 先传递给 C 语言变量
    end subroutine

end module private_trans
```

#### C 语言传递函数

文件名：trans_private_array_c.c

```C
#include <stdio.h>

double *Fp;

void trans_private_array_c_(double *fp) {
    Fp = fp;
    puts("C Fp");
    for (int i = 0; i < 12; i ++) {
        printf("%lf  ", Fp[i]);
    }
    puts("");
}
```

#### Fortran 驱动函数 main.f90

```fortran
program main
    use private_trans

    implicit none

    call init()

end program main
```

#### 编译运行

```shell
$ gfortran main.f90 trans_private_array_c.c private_trans.f90
$ ./a.out
# 输出如下
11.000000000000000        0.0000000000000000        0.0000000000000000        0.0000000000000000        22.000000000000000        0.0000000000000000        0.0000000000000000        0.0000000000000000        0.0000000000000000        0.0000000000000000        0.0000000000000000        34.000000000000000     
C Fp
11.000000  0.000000  0.000000  0.000000  22.000000  0.000000  0.000000  0.000000  0.000000  0.000000  0.000000  34.000000  
```



## subroutine 转 function

C 中函数全用小写

尽量使用 `subroutine` 的形式来调用 C 语言的 `function`，翻译步骤如下：

| 转换项目   | Fortran                         | C                    |
| ---------- | ------------------------------- | -------------------- |
| 函数名     | xxx_c（加后缀 `_c` 便于人区分） | xxx_c_（增加下划线） |
| 返回值类型 | 无                              | void                 |
| 参数传递   | 原子变量/数组/结构体            | 均用指针传递         |

### 实例演示

#### fortran subroutine

```fortran
! 子程序声明
subroutine spline_vector_v2_c &
      ( r_output, spl_param, &
        n_l_dim, n_coeff, n_grid_dim, &
        n_points, n_vector, out_result )
        ! 参数类型声明
        real*8  :: r_output
        integer :: n_l_dim
        integer :: n_coeff
        integer :: n_grid_dim
        integer :: n_points
        integer :: n_vector
        real*8  :: spl_param(n_l_dim,n_coeff,n_grid_dim)
        real*8  :: out_result(n_vector)
     	....
end subroutine spline_vector_v2
```

#### C function

```C
// real *spl_param 强制转换高维指针为一维指针
void spline_vector_v2_c_(real *p_r_output, real *spl_param,
                         int *p_n_l_dim, int *p_n_coeff, int *p_n_grid_dim,
                         int *p_n_points, int *p_n_vector,
                         real *out_result
                         )
```

## Fortran 调用 C++ 函数

本质还是装换为 Fortran 调用 C 函数，增加以下两个步骤：

+ 在 C 文件中用 `extern "C" {要被调用的函数}` 声明要被调用的函数
  + `extern "C" ` 告诉编译器对应的部分用 C 标准来编译链接
+ 链接时加上 `-lstdc++` 选项


## C 语言

### 预定义跟踪调试

+ `__FILE__`：当前位置所在的文件名（字符串）
+ `__LINE__`：当前行数（整型）
+ `__func__`：当前行所在的函数（字符串）

演示实例：`file.c` 

```C
#include <stdio.h>

void test() {
    printf("[test]:  FILE = %s; LINE = %d; func = %s\n", __FILE__, __LINE__, __func__);
}
int main() {
    printf("[main]:  FILE = %s; LINE = %d; func = %s\n", __FILE__, __LINE__, __func__);
    test();
    return 0;
}
/** 输出如下: 
    [main]:  FILE = file.c; LINE = 7; func = main
    [test]:  FILE = file.c; LINE = 4; func = test
*/
```

## Checkilist

翻译后检查一下问题：

+ 语法检查：先用 C 语言编译器进行语法检查，再逐行比对，看变量定义，参数类型，位置是否正确
  + 将新的 `.c` 文件放入项目的对于位置
  + Fortran 对应文件中修改调用函数，**无下划线**（C 中函数必须有下划线），严格检查传入参数类型和位置，引用模块（若有）
  + Makefile.backend 增加对应编译语句
  + 注意命名冲突问题
+ 语义检查：主要靠人脑
  + demo 先行：最快的方法是**逐行比对，确保理解每个 fortran 语句的语义**，如遇到不确定的用法，先写个 demo 确认，然后再动手翻译
  + 数组转换：涉及两种
    + 下标变换：三种类型 `[l,r]`
      + `l=0`：无需变换
      + `l>0`：下标需减去 `l`
      + `l<0`：下标需加上 `l`
    + 维度变换：高维转为低维
  + 私有变量传递的时机（最好单独写个函数，在被翻译的函数前面调用它）
  + 变量初始化（否则容易陷入死循环）

# pthread 模拟多核

用多线程库 pthread 来模拟多核执行，便于在本机调试，检测内存错误。

目的：找到访问冲突的变量

halgrind 可检测多线程内存读写冲突（valgrind 工具集之一）

模拟思路：

一个线程模拟一个核

+ 给结构体参数增加一个变量 tid（取值范围[0,63]），来模拟从核编号 `_MYID`
+ 为了尽可能少的改动代码，定义一个新的函数接口 `sum_up_thread_interface_c_`，参数为结构体，再调用原有从核函数，并在新函数接口中写一个 for 循环，i 从 [0,63] 来模拟从核编号
+ 从核函数内部尽可能不要改动，将 allocate 重定义为 malloc，记得增加对应 free 语句
+ fortran 直接调用 `sum_up_thread_interface_c_`，其内部使用使用 pthread 模拟 64 个从核

Fp 重点检查，Fp_private 虽然分配了空间，但是私有对其没影响吗？

sum_up.f90中注释第一个循环内的赋值语句，无段错误，但结果不对；不注释它，出现段错误

下图标黄的为数组，可他们的地址均为0x0，可直接打印是有值的

![1600795431086](任务二/1600795431086.png)

读写冲突的简单判断方法，每次运行程序结果都会轻微改变：

+ 8781.055
+ 8781.056

thread_local 变量不支持动态分配，但是可以通过如下方法规避（神威也这么干）

- 申请一块静态大空间
- 手动分配它

## sum_up_i_batch_loop2_c_

```
real *coord_current 读写冲突

average_delta_v_hartree_real 本质是全局变量，应该每个从核维护一个，最后累加在一起？
类似全局变量 a=0, 在函数中 a+=1; 实际上会造成读写冲突
解决方案：给结构体增加一个average_delta_v_hartree_reals[64]变量，分别累加不同从核的值，回到fortran中，join之后再进行64个值的累加
提问：fortran中仅知数组的loc，如何访问数组？
回答：数组的基类型和维度，然后用cray pointer

```

## sum_up_i_batch_loop1_c_

```
# swgcc530 编译器可将以下命令加入link.sh进行调试
-Wl,--whole-archive,-wrap,athread_init,-wrap,__expt_handler,-wrap,__real_athread_spawn /home/export/online3/swmore/release/lib/libspc.a -Wl,--no-whole-archive
```

多次运行结果完全一致：说明不存在数据冲突

但是结果与正确有误差0.1左右

仅仅用一个从核来运行，看看结果是否正确，若正确，说明并行有问题，不正确，说明主核版本就跑不通，认真调试即可

- 测试结果表明，一个从核结果和64从核一样，说明并行无问题

是否是数学库的问题？pow在sw9上会导致该问题

原始的CPE版本在sw5跑会出现无法收敛问题，猜测是并行带来的读写冲突问题，若是将其换成单从核，若无问题则说明并行冲突解决过程有问题，否则说明原来的主核版可能不对（每次打开新shell后，必须**module load sw/compiler/gcc710**)

- 测试结果表明，单从核结果正确，说明冲突解决过程有问题
- 改为64从核并行，出现无法收敛的问题，此时Fp用 __thread_local定义（或者将天河上的pthread改为对应的sw版本）
- 将 delta_v_hartree_multipole_component 改为本地变量，解决读写冲突，同时收敛正常，结果正确

```c
real *coord_current 读写冲突
real *coord_current[3];

real *ylm_tab 读写冲突
#define MAX_POTSQ 96;
real ylm_tab[MAX_POTSQ];

2851
real *delta_v_hartree_multipole_component =>delta_v_hartree_multipole_component[MAX_POTSQ];

real *rho_multipole_component => real rho_multipole_component[MAX_POTSQ];

Fp 仅在 hartree_potential_real_p0_c.c 中使用。
函数：spline_vector_c_ 的out_result参数
real *F = &Fp(0, *p_i_center - 1);
Fp(i, *p_i_center - 1) = -Fp(i, *p_i_center - 1);
Fp 是否会影响后续程序？
Fp 仅在以下四个函数出现，并不会相互影响，因为先写后读，Fp本质是个跨函数的临时变量，因此给每个线程拷贝一个Fp或者增加一个参数Fp即可解决问题。
- 若使用 __local real *Fp; 则需要在线程内部进行malloc
- 若使用变量传递的方法，则需要修改多处定义
- Fp 是否可以不用memcpy？：确实可以，因为他仅仅是中间变量，每次用之前都会先计算写入初值，所以无需拷贝。逻辑和实验均可证明。Fp的传递其实也可以消去

far_distance_hartree_fp_cluster_single_atom_p2_c_{
    Fp(0, 0) = 1.0 / *p_dist_tab;
    for (i_l = 1; i_l <= *p_l_max + *p_hartree_force_l_add; i_l++)
    {
        one_minus_2l -= 2;
        Fp(i_l, 0) = Fp(i_l - 1, 0) * ((double)one_minus_2l) / dist_sq;
    }
}

far_distance_real_hartree_potential_single_atom_p2_c_{
	dpot = 0;
	for (n = 1; n <= n_cc_lm_ijk[*p_l_max]; n++)
    {
         /* index_cc(1:,1:) */
         ii = index_cc(n - 1, 2);
         jj = index_cc(n - 1, 3);
         kk = index_cc(n - 1, 4);
         nn = index_cc(n - 1, 5);
 
         dpot += coord_c(ii, 0) * coord_c(jj, 1) * coord_c(kk, 2) * Fp(nn, 0) * multipole_c(n - 1, center_to_atom[*p_i_center - 1] - 1);
     }
}

far_distance_real_hartree_potential_single_atom_c_{
	for (i_l = 0; i_l <= index_ijk_max_cc[(l_max) * 3 + 2]; i_l++)
    {
         for (i_l2 = 0; i_l2 <= l_max; i_l2++)
             rest_mat(i_l, i_l2) = coord_c(i_l, 2) * Fp(i_l2, *p_i_center - 1);
     }
}

far_distance_hartree_fp_periodic_single_atom_c_{
	real *F = &Fp(0, *p_i_center - 1);
	spline_vector_c(F);
	for (int i = 0; i <= lmax; i++)
	{
		Fp(i, *p_i_center - 1) = -Fp(i, *p_i_center - 1);
    }
}

//Fp 占用内存大小: (l_pot_max+2)*n_atom_list
H2: 6 * 28 * 8 = 1344B = 1.3125KB
Si: 6 * 732 * 8 = 35136B = 34.3125 KB
```

将 pthread 版本改为c串行版本时，出现段错误，而pthread版本无问题，推测是对应的创建pthread函数出问题

## 优化建议（段老师）

当前batch的partition_tab, delta_v_hartree, v_hartree_free应该值得拿到LDM里面

我觉得数组分成两类。基于i_full_points访问的用DMA取到LDM里面，因为这个每一个center都要用

addr2line -Cfe aims.x 4fff0+<短PC>

sw9objdump -d aims.x | grep  413cc8

## [Valgrind](https://www.valgrind.org/)

Valgrind 是一个构建动态分析工具的基本框架，能够自动检测许多内存管理和线程bug。当前包含[七个产品质量工具](https://www.valgrind.org/info/tools.html)：

+ a memory error detetor （内存错误检测）
+ two thread error detectors（**Helgrind** 可用于 pthread检测）
+ a call-graph generating cache (调用图生成缓存)
+ a cache and branch-prediction profiler（缓存和分支预测探测器）
+ two different heap profilers（堆探测器）

[Memcheck quickstart](https://www.valgrind.org/docs/manual/quick-start.html)

### 基本使用

+ 安装：直接 `apt-get install valgrind` 或者天河 `module load valgrind`
+ 使用：`valgrind --tool=工具名` 执行脚本。例如 `valgrind --tool=helgrind ./run.sh`。`--log-file=` 指定输出的日志文件（注意要放在执行文件前，否则不生效）

**在天河上运行，注意要 yhrun valgrind，而不是 valgrind yhrun（这样不会输出检查到的问题），可以把 yhrun 看成 shell**

pthread_create 第三参数要对函数名取地址 &

`__thread int a` 该方式可以给每个线程增加一个额外变量，比如用于存储 tid，[参考 Thread-Local](https://gcc.gnu.org/onlinedocs/gcc/Thread-Local.html)

# GDB

提示段错误、生成核心转储文件时可用 GDB 迅速定位。[参考](https://www.cnblogs.com/ims-/p/10529393.html)

CGDB：介于 GUI 和 GDB间（比 GDB+tui 强，可空格设置断点，色彩鲜艳）

DDD 和 Eclipse 也挺优秀，不过需要安装 X11

## 具体步骤

+ 在**编译选项**中加上 `-g`，生成符号信息
+ 在**编译选项**中加上 `-O0`，关闭优化，便于单步调试

+ 完成编译链接，生成可执行文件
+ 进入**测试文件**所在目录，键入 `gdb 可执行文件`，启动 gdb
+ 打断点：让程序在我们期望的位置停下来，有两种常用方法
  + `b 函数名`：在执行中遇见该**函数名**会暂停，例如 `b func` 表示执行中遇见函数 `func` 就暂停
  + `b 文件名:行号`：在执行中遇见**指定文件的指定行号**停止，例如 `b file.c:10`  表示遇见文件 `file.c` 的第 10 行时暂停
+ 键入 `r` ，开始执行程序。若断点设置正确，程序会自动在对应位置停下来
+ 此时进入调试核心，常用以下几个命令（类似在程序中插入 print 语句）
  + `l` ：列出当前所在的代码片段
  + `p 变量名/表达式`：可**打印变量值**或进行**表达式求值**
  + `si`：执行一条**汇编指令/机器指令**。想进入**函数内部**时用它
  + `n`：执行一条**语句（例如 C 语言的一条语句）**。想快速执行当前语句用它

以上步骤即可完成一个可执行文件的 GDB 调试，若是有正确的对照程序，可用以上方法同时开启两个 GDB，左右对照进行单步调试（人肉对比结果），俗称 GDB Difftest。

## 查看命令

info

```
info locals # 查看所有本地变量
info args   # 查看所有参数
```

## 跳转命令

| 命令名       | 命令简写 | 解释说明                                    |
| ------------ | -------- | ------------------------------------------- |
| step         | s        | 单步调试，步入当前函数                      |
| next         | n        | 单步调试，步过当前函数                      |
| until        | u        | 执行到当前循环结束                          |
| finish       |          | 执行到当前函数返回                          |
| continue     | c        | 继续执行到下一个断点或程序结束              |
| return       |          | 强制函数返回，可指定返回值                  |
| set var x=10 |          | 改变变量值；set {int}0x1020=10 表示内存赋值 |

```shell
# 打印数组
(gdb) p *array@len
$1 = {2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30, 32, 34, 36, 38, 40}
```

## 中断机制

### 断点 breakpoints 

程序运行到断点即交出控制权。

#### 设置断点

+ `b xxx.file:lineno`：在文件 `xxx.file` 的第 `lineno` 行加入断点
+ `b xxx.file:func` ：在文件 `xxx.fiile` 的函数 `func` 加入断点
+ `b 0xN` : 在地址 N 出加入断点（N 必须为有效的代码段地址）

#### 查看、启用和禁用断点

+ `info break` 查看断点，简写为 `i b`
+ `disalbe/delete breakpoint-id`：禁用/删除 id 号为 `breakpoint-id` 的断点

#### 保存和加载断点

+ `save break file.bp`：保存断点到文件 `file.bp`
+ `source file.bp`: 加载断点文件 `file.bp`，简写为 `so file.bp`
### 监视点 watchpoints

### 检查点 checkpoints

相当于 Git 的版本回退功能。GDB 会 fork 当前进程映像，回到检查点处就是恢复进程映像的过程。无需再从头开始运行调试，从而达到节约时间的效果。

+ `checkpoint` 表示在当前位置设置检查点（回溯点）
+ `info checkpoints` 表示查看检查点
+ `restart checkpoints-id` 表示回到指定的检查点

## TUI 模式

Terminal User Interface 终端用户界面

+ 启动时加上参数：`gdb -tui 可执行文件`
+ 若已启动 gdb，可用 `Ctrl+X+A` 进入 tui 模式；再按一次可退出 tui 模式

# 正则表达式

采用正则表达式+宏定义快速完成语义转换

```regex
r\*\*(\d{1,2})
rpow[$1]

Ewald_radius\*\*(\d{1,2})
erpow[$1]

0d0
0

arch_erf
erfc

&
空白字符

F_erf_table\(p\)
F_erf_table[p]

case\((\d{1,2})\)
case\($1\):
```

以下为替换后的结果：

```
case(1):

          F_erf_table[p] = ((2.0 * exp(-(rpow[2]/erpow[2]))* r) /  (Ewald_radius *sqrt_pi) - erfc(r/Ewald_radius))/rpow[3]

       case(2):

          F_erf_table[p] = ((exp(-(rpow[2]/erpow[2])) * (((-6.0) * erpow[2]* r - 4.0 * rpow[3]  
               + 3.0 * exp(rpow[2]/erpow[2]) * erpow[3] *sqrt_pi *erfc(r/Ewald_radius)))) 
               / (erpow[3] * sqrt_pi * rpow[5]))

       case(3):

          F_erf_table[p] = ((exp(-(rpow[2]/erpow[2])) *((30.0 * erpow[4] *r  
               + 20.0 * erpow[2] *rpow[3] + 8.0 *rpow[5] 
               - 15.0 * exp(rpow[2]/erpow[2]) *erpow[5] *sqrt_pi *erfc(r/Ewald_radius)))) 
               / (erpow[5] *sqrt_pi * rpow[7]))

       case(4):

          F_erf_table[p] = ((exp(-(rpow[2]/erpow[2])) *(((-2.0)* ((105.0 * erpow[6] *r + 
               70.0 * erpow[4]* rpow[3] + 28.0 *erpow[2] *rpow[5] + 8.0 *rpow[7])) 
               + 105.0 *exp(rpow[2]/erpow[2]) *erpow[7] *sqrt_pi *erfc(r/Ewald_radius)))) 
               /(erpow[7] *sqrt_pi *rpow[9]+epsi))

       case(5):

          F_erf_table[p] = ((exp(-(rpow[2]/erpow[2]))* ((2.0 *((945.0 *erpow[8] *r  
               + 630.0 *erpow[6] *rpow[3] 
               + 252.0 * erpow[4] *rpow[5] + 72.0 *erpow[2] *rpow[7] + 16.0 *rpow[9])) 
               - 945.0 *exp(rpow[2]/erpow[2]) *erpow[9] *  
               sqrt_pi *erfc(r/Ewald_radius))))/(erpow[9] *sqrt_pi *rpow[11]))

       case(6):

          F_erf_table[p] = ((exp(-(rpow[2]/erpow[2])) *(((-2.0) *((10395.0 *erpow[10] *r  
               + 6930.0 *erpow[8] *rpow[3] 
               + 2772.0 *erpow[6] *rpow[5]  
               + 792.0 *erpow[4] *rpow[7] + 176.0 *erpow[2] *rpow[9] + 32.0 *rpow[11])) 
               + 10395.0 *exp(rpow[2]/erpow[2]) *erpow[11] * 
               sqrt_pi *erfc(r/Ewald_radius))))/(erpow[11] *sqrt_pi *rpow[13]))

       case(7):

          F_erf_table[p] = ((exp(-(rpow[2]/erpow[2])) *((270270.0 *erpow[12] *r  
               + 180180.0 *erpow[10] *rpow[3] 
               + 72072.0 *erpow[8] *rpow[5] + 20592.0 *erpow[6] *rpow[7] 
               + 4576.0 *erpow[4] *rpow[9] + 832.0 *erpow[2] *rpow[11] +  
               128.0 *rpow[13] - 135135.0 *exp(rpow[2]/erpow[2]) 
               *erpow[13] *sqrt_pi *erfc(r/Ewald_radius))))/(erpow[13] *sqrt_pi *rpow[15]))

       case(8):
          F_erf_table[p] = ((exp(-(rpow[2]/erpow[2])) 
               * (((-2.0) *((2027025.0 *erpow[14] *r + 1351350.0 *erpow[12] *rpow[3] 
               + 540540.0 *erpow[10] *rpow[5] + 154440.0 *erpow[8] *rpow[7] + 34320.0 *erpow[6] 
               *rpow[9] + 6240.0 *erpow[4] *rpow[11] + 960.0 *erpow[2] *rpow[13] + 128.0 *rpow[15])) + 
               2027025.0 *exp(rpow[2]/erpow[2]) 
               *erpow[15] *sqrt_pi *erfc(r/Ewald_radius))))/(erpow[15] *sqrt_pi *rpow[17]))

       case(9):
          F_erf_table[p] = ((exp(-(rpow[2]/erpow[2])) *((68918850.0 *erpow[16] *r  
               + 45945900.0 *erpow[14] *rpow[3] 
               + 18378360.0 *erpow[12] *rpow[5] + 5250960.0 *erpow[10] *rpow[7] 
               + 1166880.0 *erpow[8] *rpow[9] + 212160.0 *erpow[6] *rpow[11] + 
               32640.0 *erpow[4] *rpow[13] + 4352.0 *erpow[2] *rpow[15] + 512.0 *rpow[17] 
               - 34459425.0 *exp(rpow[2]/erpow[2]) *erpow[17] *sqrt_pi *erfc(r/Ewald_radius)))) 
               /(erpow[17] *sqrt_pi *rpow[19]))

       case(10):
          F_erf_table[p] = (((1.0/(erpow[19] *sqrt_pi *rpow[21]))*((exp(-(rpow[2]/erpow[2])) * 
               (((-2.0) *((654729075.0 *erpow[18] *r + 436486050.0 *erpow[16] *rpow[3] + 
               174594420.0 *erpow[14] *rpow[5] + 49884120.0 *erpow[12] *rpow[7]  
               + 11085360.0 *erpow[10] *rpow[9] 
               + 2015520.0 *erpow[8] *rpow[11] + 310080.0 *erpow[6] *rpow[13] 
               + 41344.0 *erpow[4] *rpow[15] + 4864.0 *erpow[2] *rpow[17] + 512.0 *rpow[19])) + 
               654729075.0 *exp(rpow[2]/erpow[2]) 
               *erpow[19] *sqrt_pi *erfc(r/Ewald_radius)))))))

       case(11):
          F_erf_table[p] = (((1.0/(erpow[21] *sqrt_pi *rpow[23]))*((exp(-(rpow[2]/erpow[2])) * 
               ((2.0 *((13749310575.0 *erpow[20] *r + 9166207050.0 *erpow[18] *rpow[3] 
               + 3666482820.0 *erpow[16] *rpow[5] + 1047566520.0 *erpow[14] *rpow[7] + 
               232792560.0 *erpow[12] *rpow[9] + 42325920.0 *erpow[10] *rpow[11] 
               +  6511680.0 *erpow[8] *rpow[13] + 868224.0 *erpow[6] *rpow[15] + 
               102144.0 *erpow[4] *rpow[17] + 10752.0 *erpow[2] *rpow[19] + 1024.0 *rpow[21])) 
               - 13749310575.0 *exp(rpow[2]/erpow[2]) *erpow[21] *sqrt_pi *erfc(r/Ewald_radius)))))))

       case(12):
          F_erf_table[p] = (((1/(erpow[23] *sqrt_pi *rpow[25]))*((exp(-(rpow[2]/erpow[2])) *(((-2.0) 
               *((316234143225.0 *erpow[22] *r + 210822762150.0 *erpow[20] *rpow[3] + 
               84329104860.0 *erpow[18] *rpow[5] + 24094029960.0 *erpow[16] *rpow[7] + 
               5354228880.0 *erpow[14] *rpow[9] + 973496160.0 *erpow[12] *rpow[11] + 
               149768640.0 *erpow[10] *rpow[13] + 19969152.0 *erpow[8] *rpow[15] +  
               2349312.0 *erpow[6] *rpow[17] + 
               247296.0 *erpow[4] *rpow[19] + 
               23552.0 * erpow[2] *rpow[21] + 2048.0 *rpow[23])) + 
               316234143225.0 *exp(rpow[2]/erpow[2]) *erpow[23] *sqrt_pi *erfc(r/Ewald_radius)))))))

       case(13):
          F_erf_table[p] =(((1/(erpow[25] *sqrt_pi *rpow[27]))*((exp(-(rpow[2]/erpow[2])) 
               *((2.0 *((7905853580625.0 *erpow[24] *r + 5270569053750.0 *erpow[22] *rpow[3] 
               + 2108227621500.0 *erpow[20] *rpow[5] + 602350749000.0 *erpow[18] *rpow[7] + 
               133855722000.0 *erpow[16] *rpow[9] + 24337404000.0 *erpow[14] *rpow[11] 
               + 3744216000.0 *erpow[12] *rpow[13] + 499228800.0 *erpow[10] *rpow[15]  
               + 58732800.0 *erpow[8] *rpow[17] 
               + 6182400.0 *erpow[6] *rpow[19] + 588800.0 *erpow[4] *rpow[21] 
               + 51200.0 *erpow[2] *rpow[23] + 4096.0 *rpow[25])) - 
               7905853580625.0 *exp(rpow[2]/erpow[2]) *erpow[25] *sqrt_pi *erfc(r/Ewald_radius)))))))

       case(14):
          F_erf_table[p] = (((1.0/(erpow[27] *sqrt_pi* rpow[29]))*((exp(-(rpow[2]/erpow[2]))*  
               (((-2.0) *((213458046676875.0 *erpow[26] *r 
               + 142305364451250.0 *erpow[24] *rpow[3] + 
               56922145780500.0 *erpow[22] *rpow[5] + 16263470223000.0 *erpow[20] *rpow[7] + 
               3614104494000.0 *erpow[18] *rpow[9] + 657109908000.0 *erpow[16] *rpow[11] + 
               101093832000.0 *erpow[14] *rpow[13] + 13479177600.0 *erpow[12] *rpow[15] + 
               1585785600.0 *erpow[10] *rpow[17] + 166924800.0 *erpow[8] *rpow[19] + 
               15897600.0 *erpow[6] *rpow[21] + 1382400.0 *erpow[4] *rpow[23] + 
               110592.0 *erpow[2] *rpow[25] + 8192.0 *rpow[27])) + 
               213458046676875.0 *exp(rpow[2]/erpow[2]) *erpow[27] *sqrt_pi *erfc(r/Ewald_radius)))))))

       case(15):
          F_erf_table[p] = (((1.0/(erpow[29] *sqrt_pi *rpow[31]))*((exp(-(rpow[2]/erpow[2]))* 
               ((12380566707258750.0 *erpow[28] *r 
               + 8253711138172500.0 *erpow[26] *rpow[3] + 
               3301484455269000.0 *erpow[24] *rpow[5] + 943281272934000.0 *erpow[22] *rpow[7] + 209618060652000.0 
               *erpow[20] *rpow[9] + 38112374664000.0 *erpow[18] *rpow[11] + 
               5863442256000.0 *erpow[16] *rpow[13] + 781792300800.0 *erpow[14] *rpow[15] + 
               91975564800.0 *erpow[12] *rpow[17] + 9681638400.0 *erpow[10] *rpow[19] + 
               922060800.0 *erpow[8] *rpow[21] + 80179200.0 *erpow[6] *rpow[23] + 
               6414336.0 *erpow[4] *rpow[25] + 475136.0 *erpow[2] *rpow[27] + 32768.0 *rpow[29] - 
               6190283353629375.0 *exp(rpow[2]/erpow[2]) *erpow[29] *sqrt_pi *erfc(r/Ewald_radius)))))))

       case(16):
          F_erf_table[p] = (((1.0/(erpow[31] *sqrt_pi *rpow[33]))*((exp(-(rpow[2]/erpow[2]))* 
               (((-2.0) *((191898783962510625.0 *erpow[30] *r + 
               127932522641673750.0 *erpow[28] *rpow[3] + 51173009056669500.0 *erpow[26] *rpow[5] + 
               14620859730477000.0 *erpow[24] *rpow[7] + 
               3249079940106000.0 *erpow[22] *rpow[9] + 590741807292000.0 *erpow[20] *rpow[11] + 
               90883354968000.0 *erpow[18] *rpow[13] 
               + 12117780662400.0 *erpow[16] *rpow[15] + 
               1425621254400.0 *erpow[14] *rpow[17] + 150065395200.0 *erpow[12] *rpow[19] + 
               14291942400.0 *erpow[10] *rpow[21] 
               + 1242777600.0 *erpow[8] *rpow[23] + 
               99422208.0 *erpow[6] *rpow[25] + 7364608.0 *erpow[4] *rpow[27] + 
               507904.0 *erpow[2] *rpow[29] + 32768.0 *rpow[31])) + 
               191898783962510625.0 *exp(rpow[2]/erpow[2]) *erpow[31] *sqrt_pi  *erfc(r/Ewald_radius)))))))

       
```





```
;
break;
case(0):

          erfc_r = arch_erfc(r*inv_Ewald_radius_to(1))

          F_table[p]  = erfc_r /r

       ;
break;
case(1):

!          F_table[p] =  - (((2.0 * exp(- (rpow[2] /erpow[2] )) * r )  
!               / (Ewald_radius * sqrt_pi ) + erfc(r /Ewald_radius) ) /rpow[3] )

          exp_m = exp(- (rto2 *inv_Ewald_radius_to(2) ))

          F_table[p] =  - (((2.0 * exp_m * r )  
               / (Ewald_radius * sqrt_pi ) + erfc_r ) /rto(1) )

       ;
break;
case(2):


!          F_table[p] = 
!               ( ( exp(-(rpow[2] /erpow[2] ) ) * ((6 * erpow[2] * r + 4 * rpow[3] + 3 *  exp(rpow[2] /erpow[2] ) * 
!               erpow[3] * sqrt_pi * arch_erfc(r/Ewald_radius)) ) ) / (erpow[3] * sqrt_pi* rpow[5] ) ) 

          exp_p = 1.d0/exp_m

          F_table[p] = 
               ( ( exp_m * ((6 * Ewald_radius_to(2) * r + 4 * rto(1) + 3 * exp_p * 
               Ewald_radius_to(3) * sqrt_pi * erfc_r  ) ) ) / (Ewald_radius_to(3) * sqrt_pi* rto(2) ) ) 


       ;
break;
case(3):

!          F_table[p] = 
!               -(( exp (- (rpow[2] /erpow[2] ) )  * ((30 * erpow[4] * r +  20 * Ewald_radius_to(2) * rpow[3] + 8 * rpow[5] +  
!               15  * exp(rpow[2] /erpow[2] ) * erpow[5]  * sqrt_pi * arch_erfc( r /Ewald_radius)) ) ) 
!               / (erpow[5] *  sqrt_pi * rpow[7] ))

          F_table[p] = 
               -(( exp_m  * ((30 * Ewald_radius_to(4) * r +  20 * Ewald_radius_to(2) * rto(1) + 8 * rto(2) +  
               15  * exp_p * Ewald_radius_to(5)  * sqrt_pi * erfc_r    ) ) ) 
               / (Ewald_radius_to(5) *  sqrt_pi * rto(3) ))


       ;
break;
case(4):

!          F_table[p] = 
!               ( ( exp(- (rpow[2] /erpow[2] ) ) * ((210.0 * erpow[6] * r + 140.0 * erpow[4] * rpow[3]   
!               + 56.0 * erpow[2] * rpow[5] + 16.0 * rpow[7] + 105.0 * exp(rpow[2] /erpow[2] ) * erpow[7] 
!               *sqrt_pi* arch_erfc(r/Ewald_radius)) ) ) / (erpow[7]  * sqrt_pi * rpow[9] ) )

           F_table[p] = ( exp_m  * ( 
                ddot(4,P_erfc_4(1:4),1, rto(0:3),1) +  P_erfc_4(5) * exp_p * erfc_r ))  
                / ( P_erfc_4(6) * rto(4) ) 


       ;
break;
case(5):

!          F_table[p] = 
!               ( (- ( ( exp(- (rpow[2] /erpow[2] )) *  ((2.0 *  ((945.0 * erpow[8] * r + 
!               630.0 * erpow[6] * rpow[3] + 252.0 * erpow[4] * rpow[5] + 72.0 * erpow[2] * rpow[7] +  
!               16.0 * rpow[9]) ) +  945.0 * exp(rpow[2] /erpow[2]) * 
!               erpow[9] * sqrt_pi * arch_erfc( r /Ewald_radius)))) / (erpow[9] * sqrt_pi * rpow[11] ) ) ) )

          F_table[p] =  - ( exp_m  * 
               ( ddot(5,P_erfc_5(1:5),1, rto(0:4),1) +   P_erfc_5(6) * exp_p * erfc_r  )) 
               / (  P_erfc_5(7) * rto(5) ) 


       ;
break;
case(6):

!         F_table[p] = 
!               ( ( exp (-(rpow[2] /erpow[2] )) * ((20790.0 * erpow[10] * r + 13860.0 * erpow[8]   
!               * rpow[3] + 5544.0 * erpow[6] * rpow[5] + 1584.0 * erpow[4] * rpow[7] + 
!               352.0 *  erpow[2] * rpow[9] + 64.0 * rpow[11] +   
!               10395.0 * exp(rpow[2] /erpow[2] ) * erpow[11] * 
!               sqrt_pi *  arch_erfc(r/Ewald_radius)) ) ) / (erpow[11]  * sqrt_pi *  rpow[13] ) )

          F_table[p] = ( exp_m * 
               ( ddot(6, P_erfc_6(1:6),1, rto(0:5),1) + P_erfc_6(7) * exp_p * erfc_r )) 
               / (P_erfc_6(8) *  rto(6) ) 


       ;
break;
case(7):


          F_table[p] = 
               ( (- ( ( exp (- (rpow[2] /erpow[2] ) )  * ((270270.0 * erpow[12] *  r + 
               180180.0 * erpow[10] * rpow[3] + 72072.0 * erpow[8] * rpow[5] + 20592.0 * erpow[6] * rpow[7] + 
               4576.0 * erpow[4] * rpow[9] + 832.0 * erpow[2] * rpow[11] + 128.0 * rpow[13] +  
               135135.0 *  exp (rpow[2] /erpow[2] ) * erpow[13] * 
               sqrt_pi * erfc(r /Ewald_radius)) ) ) / (erpow[13] * sqrt_pi * rpow[15] ) ) ) )

       ;
break;
case(8):

          F_table[p] = 
               ( ( exp (- (rpow[2] /erpow[2] ) ) * ((2 * ((2027025.0 * erpow[14] * r +   
               1351350.0 * erpow[12] * rpow[3] + 540540.0 * erpow[10] * rpow[5] + 
               154440.0 * erpow[8] * rpow[7] + 34320.0 * erpow[6] * 
               rpow[9] + 6240.0 * erpow[4] * rpow[11] + 960.0 * erpow[2] * rpow[13] + 128.0 * rpow[15]) ) + 2027025.0 *   
               exp (rpow[2] /erpow[2] ) * erpow[15] *  sqrt_pi * arch_erfc(r/Ewald_radius)) ) ) 
               / (erpow[15]  * sqrt_pi * rpow[17] ) )

       ;
break;
case(9):

          F_table[p] = 
               ( (- ( ( exp(- (rpow[2] /erpow[2] ) ) * ((68918850.0 * erpow[16] * r +  
               45945900.0 *erpow[14] * rpow[3] + 18378360.0 * erpow[12] * rpow[5] + 
               5250960.0 * erpow[10] * rpow[7] + 1166880.0 * erpow[8] * rpow[9] +  
               212160.0 * erpow[6]  *rpow[11] + 32640.0 * erpow[4] * rpow[13] + 4352.0 *  erpow[2]*  rpow[15] +  
               512.0 *  rpow[17] + 
               34459425.0 *exp (rpow[2] /erpow[2] ) * erpow[17]  * 
               sqrt_pi * erfc(r /Ewald_radius)) ) ) / (erpow[17]  * sqrt_pi*  rpow[19] ) ) ) )

       ;
break;
case(10):

          F_table[p] = 
               ( ( (1 / (erpow[19] * sqrt_pi * rpow[21] ) )* (( exp(- (rpow[2] /erpow[2] ) ) *
               ((1309458150.0 * erpow[18] * r + 872972100.0 * erpow[16] * rpow[3] +  
               349188840.0 * erpow[14] * rpow[5] + 
               99768240.0 *erpow[12] * rpow[7] + 22170720.0 * erpow[10] * rpow[9] + 
               4031040.0 * erpow[8] * rpow[11] + 620160.0 * erpow[6] * rpow[13] +  
               82688.0 * erpow[4] * rpow[15] + 
               9728.0 * erpow[2] * rpow[17] + 1024.0 * rpow[19] + 
               654729075.0 *  exp (rpow[2] /erpow[2] ) * erpow[19] *  sqrt_pi * erfc(r /Ewald_radius)) )) ) ) )

       ;
break;
case(11):


          F_table[p] = 
               ( ( (1 / (erpow[21]  * sqrt_pi * rpow[23] ) )* (( exp (- (rpow[2] /erpow[2] ) ) * 
               (( (-2.0)  * ((13749310575.0 * erpow[20] * r + 9166207050.0 * erpow[18]*  rpow[3] + 
               3666482820.0 * erpow[16]*  rpow[5] + 1047566520.0 * erpow[14]*  rpow[7] + 
               232792560.0 * erpow[12] * rpow[9] + 42325920.0 * erpow[10]*  rpow[11] +  
               6511680.0 *  erpow[8]  *rpow[13] + 868224.0 * erpow[6] * rpow[15] + 
               102144.0 * erpow[4] * rpow[17] + 10752.0 *  erpow[2] * rpow[19] + 1024.0 * rpow[21]) ) - 
               13749310575.0 * exp (rpow[2] /erpow[2] ) * erpow[21]  * sqrt_pi*  erfc(r /Ewald_radius)) )) ) ) )

       ;
break;
case(12):

          F_table[p] = 
               ( ( (1 / (erpow[23]  * sqrt_pi * rpow[25] ) )* (( exp (- (rpow[2] /erpow[2] ) ) *
               ((2 *  ((316234143225.0 * erpow[22] * r + 210822762150.0 * erpow[20] * rpow[3] + 
               84329104860.0 * erpow[18] * rpow[5] + 24094029960.0 * erpow[16] * rpow[7] + 
               5354228880.0 * erpow[14] * rpow[9] + 973496160.0 * erpow[12] * rpow[11] + 
               149768640.0 * erpow[10] * rpow[13] + 19969152.0 * erpow[8] * rpow[15] + 
               2349312.0 * erpow[6] * rpow[17] + 247296.0 * erpow[4] * rpow[19] + 
               23552.0 * erpow[2] * rpow[21] + 2048 * rpow[23]) ) + 
               316234143225.0 * exp (rpow[2] /erpow[2] ) * erpow[23]  *  
               sqrt_pi * erfc(r /Ewald_radius)) )) ) ) )

       ;
break;
case(13):

          F_table[p] = 
               ( ( (1 / (erpow[25] *  sqrt_pi * rpow[27] ) ) *(( exp (- (rpow[2] /erpow[2] ) ) * 
               (( (-2 ) *  ((7905853580625.0 * erpow[24] * r + 5270569053750.0 * erpow[22]*  rpow[3] + 
               2108227621500.0 * erpow[20] * rpow[5] + 602350749000.0 * erpow[18] * rpow[7] + 
               133855722000.0 * erpow[16] * rpow[9] + 24337404000.0 * erpow[14] * rpow[11] + 
               3744216000.0 * erpow[12] * rpow[13] + 499228800.0 * erpow[10] * rpow[15] + 
               58732800.0 * erpow[8] * rpow[17] + 6182400.0 * erpow[6]*  rpow[19] + 
               588800.0 * erpow[4] * rpow[21] + 51200.0 * erpow[2] * rpow[23] + 4096.0 * rpow[25]) ) - 
               7905853580625.0 * exp (rpow[2] /erpow[2] ) * erpow[25] *  
               sqrt_pi *  arch_erfc(r/Ewald_radius)) )) ) ) )

       ;
break;
case(14):

          F_table[p] = 
               ( ( (1 / (erpow[27] *  sqrt_pi * rpow[29] ) )* (( exp (- (rpow[2] /erpow[2] ) ) * 
               ((2 *  ((213458046676875.0 * erpow[26] * r + 142305364451250.0 * erpow[24] * rpow[3] + 
               56922145780500.0 * erpow[22] * rpow[5] + 16263470223000.0 * erpow[20] * rpow[7] + 
               3614104494000.0 * erpow[18] * rpow[9] + 657109908000.0 * erpow[16] * rpow[11] + 
               101093832000.0 * erpow[14] * rpow[13] + 13479177600.0 * erpow[12] * rpow[15] + 
               1585785600.0 * erpow[10] * rpow[17] + 166924800.0 * erpow[8] * rpow[19] + 
               15897600.0 * erpow[6] * rpow[21] + 1382400.0 * erpow[4] * rpow[23] + 
               110592.0 * erpow[2] * rpow[25] + 8192 * rpow[27]) ) + 
               213458046676875.0 * exp (rpow[2] /erpow[2] ) * erpow[27]  *  
               sqrt_pi* arch_erfc(r/Ewald_radius)) )) ) ) )


       end select
```



# FAQ

> 1.我们整个课题是做什么？

整个课题是优化 fhiaims 程序，使其在中国最快的超级计算机(神威系列)上达到单核加速快，并行效率高

> 2.这个 FHI-AIMS 和整体课题的关系

同上

> 3.sum_up_whole_potential_shanghui_dielectric.f90 是 FHI-AIMS 的一个小模块，它有什么用？

这个模块是fhiaims的4大计算模块之一，是热点函数，需要优化。它的功能是，给它一个电子密度n, 它通过多级展开的方法得到一个势能 V.

> 4.make scalapack.mpi -j 8  生成的是啥？是一个链接库还是可执行文件？有啥用？

生产的是可执行文件，在 bin/ 目录下

> 5.改写函数为 C 语言时，integrate_delta_v_hartree 内部有调用其它的子程序，是不是也要一块重写？

是的，其他的函数也要一起翻过来

# 参考

[Fortran 内置函数参考](https://docs.oracle.com/cd/E19205-01/820-1202/aetjb/index.html)

[Mix C Fortran](https://wiki.ubuntu.org.cn/Mix_C_Fortran)：很好的网站，包含编译，链接等教程

[fortran 和 C 混合编程](https://www.cnblogs.com/snake553/p/6962386.html)

[混合编程教程](http://www.yolinux.com/TUTORIALS/LinuxTutorialMixingFortranAndC.html)

[MPI](https://www.mcs.anl.gov/research/projects/mpi/)：消息传递接口

[ScaLAPACK](http://www.netlib.org/scalapack/scalapack_home.html)：并行计算，可扩展的 LAPACK（依赖 BLAS） + MIMD（multiple instruction, multiple data）

[MKL](https://software.intel.com/content/www/us/en/develop/tools/math-kernel-library/choose-download.html)：Intel 开发的数学核心计算库

[线性代数运算库比较](https://www.zhihu.com/question/27872849)

[ifort](https://www.ch.cam.ac.uk/computing/software/intel-fortran-ifort)：intel 开发的 fortran 编译器

[Linux 命令大全](https://man.linuxde.net/apt-get-2)

[改写过程](https://gitee.com/shanghui_ict/fhi-aims_20191127_swgemm/commit/83cdd0393264b86c122dc745748fb5a1c1f8c2f0)

[30 分钟入门 shell 脚本](https://github.com/qinjx/30min_guides/blob/master/shell.md)

```
ifort -O3 -ip -module . -fp-model precise  -o hirshfeld_analysis.o -c hirshfeld_analysis.f90 
ifort: error #10106: Fatal error in /opt/intel/composer_xe_2015.0.090/bin/intel64/fortcom, terminated by kill signal
compilation aborted for calculate_fock_matrix_p0.f90 (code 1)
make: *** [calculate_fock_matrix_p0.o] Error 1
make: *** Waiting for unfinished jobs....
```

ifort -O3 -ip -module . -fp-model precise -c calculate_fock_matrix_p0.f90

```
scalapack_wrapper.f90(1): warning #5462: Global name too long, shortened from: scalapack_wrapper_mp_get_first_order_dm_sparse_matrix_dielectric_for_elsi_scalapack_$BLK.mpi_tasks_mp_mpi_argv_null_ to: _get_first_order_dm_sparse_matrix_dielectric_for_elsi_scalapack_$BLK.mpi_tasks_mp_mpi_argv_null_
! WPH:  The module is misnamed:  it is not a true ScaLAPACK wrapper, but rather it deals
^
```

```
scalapack_wrapper_mp_get_first_order_dm_sparse_matrix_dielectric_for_elsi_scalapack_$BLK.mpi_tasks_mp_mpi_argv_null_

_get_first_order_dm_sparse_matrix_dielectric_for_elsi_scalapack_$BLK.mpi_tasks_mp_mpi_argv_null_
```



```shell
ifort: error #10106: Fatal error in /opt/intel/composer_xe_2015.0.090/bin/intel64/fortcom, terminated by kill signal
compilation aborted for calculate_fock_matrix_p0.f90 (code 1)
make: *** [calculate_fock_matrix_p0.o] Error 1
make: *** Waiting for unfinished jobs....
```



```shell
ifort -O3 -ip -module . -fp-model precise  -o mbd-std/quadrature_grid.o -c mbd-std/quadrature_grid.f90 && touch -c ./quadrature_grid_module.mod
ifort: error #10106: Fatal error in /opt/intel/composer_xe_2015.0.090/bin/intel64/fortcom, terminated by kill signal
compilation aborted for forces_densmat.f90 (code 1)
make: *** [forces_densmat.o] Error 1
make: *** Waiting for unfinished jobs....
```

[参考](https://blog.csdn.net/tengh/article/details/7039966)

```
ld: ./esp_charges.o: file not recognized: file truncated
```
# KMC

seed 不变

region 改体系大小

set

- 0：空位，0.0134 表示占用比例
- 2：Cu

barrier Fe 0.56 能量壁垒

nbody eam xxx.dmk 势能模型，eam 最准确，ppair2 简单却不准确

run：蒙卡时间

sector： 直观看起来和通信次数成反比，即越大通信次数越少。但有自己物理含义(事件发生的平均值)

stats: 指定多少蒙卡时间打印一次

dump：输出文件，ovito软件可以处理



接口：app_vacancy.cpp/calcul_de()

[originpro 数据画图](https://www.originlab.com/Origin)

ovito

[deepmd快速安装](http://bbs.deepmd.org/bbs_en/forum.php?mod=viewthread&tid=76&extra=page%3D1)

[Loss Function](https://rohanvarma.me/Loss-Functions/) 

[常见损失函数](https://mp.weixin.qq.com/s?__biz=MzIwOTc2MTUyMg==&mid=2247499664&idx=3&sn=3aaaf703ab306ad02caabef30c4b6001&chksm=976c5a0da01bd31b3a9482d69e4cd097abebfc3f2d3440759c1b90e8f379284acd9c314f2981&mpshare=1&scene=23&srcid=1112EwB7GQEEruGgh8zndXLJ&sharer_sharetime=1605176911569&sharer_shareid=b2683982e761656e8b0970f4688b5004#rd)



```shell
atom->nlocal # 需要计算能量的原子数目（核心原子数）
atom->ntypes  # 有几种原子类型
atom->type[i] # 第i个原子的类型
atom->x  # 原子坐标(x,y,z) 
atom->tag_enable # true

#========= 不需要的变量 ============
atom->atom_style # 不用
atom->f # 力
atom->set_mass(atom_mass) # 质量、函数
    
comm->me # 通信统计相关
atom->eentropy # 熵
virial # 维里
eflag, vflag # lammps 特有
```

商老师，我能提个意见嘛？让陈欣写tensoralloy接口，提供给我们so库。理由如下：

**一、现状**：我投入大量的精力（32h），但是进展缓慢，性价比低。因为alloy本质是以继承子类形式来实现lammps固有接口，分离过程本质上是在看lamps源码然后进行分离重组，并不是大致过程明白就能分离重组正确的。

目前有两个方案可选：

| 方案                              | 完成时间   |
| --------------------------------- | ---------- |
| 1、我继续读源码，死磕拆分重组     | **不可控** |
| 2、**陈欣老师写接口，提供 so 库** | **2天**    |

**二、综合对比**：使用方案1，只要肯花时间是一定能完成任务的，但存在几个缺点：

- 从**项目**来说，KMC 仅仅把 tensoralloy 当成黑箱来调用，而我们重组实现对应功能，反而是将其作为**白盒**来实现，与项目本意相悖。
- 从**个人**来说，当成白盒费时费力，同时，该过程并不能给我带来能力上的提升，不仅浪费了我在优化程序上的时间，还增加了焦躁情绪。
- 从**合作角度**来说，陈欣对自己代码极为熟练，让他来重组实现接口，不仅大大节约时间，快速推进项目，而且能把我们时间花在刀刃上。**我们的目的仅仅是调用tensoralloy**。

**三、结论**：综上，我推荐使用方案2，虽然我已经投入一周的、

时间在方案1，但及时调整方案，反而可更好更快达到最终目的。以上是我个人深思熟虑后的结果，并不是我找借口想要推脱工作，而是为了项目更好更快的达到目的。有不妥之处咱们可再商量。

```
g++ -std=c++11 -I/home/hhshang2/dat01/chenxin/libtensorflow/1.15.3-conda/include/tensorflow/bazel-bin/tensorflow/include/ -c virtual_atom_approach.cpp                                             
libtensorflow/1.15.3/include/tensorflow/bazel-bin/tensorflow/include/tensorflow/core/platform/windows/integral_types.h

# makefile中写相对路径找不到？
gcc -O3 -g -Wall -Wextra -std=c++11 -I./jsoncpp/json/ -I/home/hhshang2/dat01/chenxin/libtensorflow/1.15.3-conda/include/tensorflow/bazel-bin/tensorflow/include/ -c virtual_atom_approach.cpp

```

- 编写 CMakeList.txt
- 创建 build 目录，`cd build`，`cmake ..`：会生成很多文件，其中包括 Makefile
- make 即可完成编译链接（`make VERBOSE=1`可输出详细信息，便于调试）



# LWPF热点分析

大量除法和平方根

动态负载均衡

- LDM：index_cc，multipole_c，index_ijk_max_cc
- mat直接算，不要先计算出来保存（只用了一次）：coord_mat，rest_mat（循环展开）

```c
far_distance_real_hartree_potential_single_atom_c_()
{
	// perf1: coord_c
    for (i_coord = 0; i_coord < 3; i_coord++)
    {
        // perf5: index_ijk_max_cc
        for (i_l = 1; i_l <= index_ijk_max_cc[(l_max) * 3 + i_coord]; i_l++)
            coord_c(i_l, i_coord) = dir[i_coord] * coord_c(i_l - 1, i_coord);
    }

    for (i_l = 0; i_l <= index_ijk_max_cc[(l_max) * 3]; i_l++)
    {
        for (i_l2 = 0; i_l2 <= index_ijk_max_cc[(l_max) * 3 + 1]; i_l2++)
            coord_mat(i_l, i_l2) = coord_c(i_l, 0) * coord_c(i_l2, 1);
    }

    for (i_l = 0; i_l <= index_ijk_max_cc[(l_max) * 3 + 2]; i_l++)
    {
        for (i_l2 = 0; i_l2 <= l_max; i_l2++)
            rest_mat(i_l, i_l2) = coord_c(i_l, 2) * Fp(i_l2, *p_i_center - 1);
    }

    // n_cc_lm_ijk(0:)
    for (n = 0; n < n_cc_lm_ijk[l_max]; n++)
    {
        // perf4: index_cc
        ii = index_cc(n, 2);
        jj = index_cc(n, 3);
        kk = index_cc(n, 4);
        // perf2: mat直接算 
        vector[n] = coord_mat(ii, jj) * rest_mat(kk, index_cc(n, 5));
    }

    dpot = 0.0;
    for (int i = 0; i < n_cc_lm_ijk[l_max]; i++)
    {
        // precise bug1: multipole is address of array
        // perf3: multipole_c 单维度->ldm
        dpot += vector[i] * multipole_c(i, center_to_atom[*p_current_center - 1] - 1);
    }

}
```

- 除法，sqrt太慢，考虑打表将其放入二维数组，在i_center外先打表（快速平方根算法考虑）
- center移入从核考虑？

```c
// L = max(2,0)
for (L = 2; L <= l_atom_max; L++)
{
    INDEX  = L*L+1;
    INDEX2 = INDEX + 2*L;
    MSIGN  = 1 - 2*MOD(L,2);

    YL1L1R = ylm_tab(INDEX-1);
    YL1L1I = - MSIGN * ylm_tab(INDEX-2*L+1);
    TEMP1 = -sqrt((double)(2*L+1)/(double)(2*L))*trigonom_tab(1);
    YLLR = TEMP1*(trigonom_tab(4)*YL1L1R - trigonom_tab(3)*YL1L1I);
    YLLI = TEMP1*(trigonom_tab(4)*YL1L1I + trigonom_tab(3)*YL1L1R);
    ylm_tab(INDEX2) = YLLR;
    ylm_tab(INDEX)  = MSIGN * YLLI;
    INDEX2 = INDEX2 - 1;
    INDEX  = INDEX  + 1;

    TEMP2 = sqrt((double)(2*L+1))*trigonom_tab(2);
    YLL1R = TEMP2*YL1L1R;
    YLL1I = TEMP2*YL1L1I;
    ylm_tab(INDEX2) = YLL1R;
    ylm_tab(INDEX)  = - MSIGN * YLL1I;
    INDEX2 = INDEX2 - 1;
    INDEX  = INDEX  + 1;

    I4L2 = INDEX - 4*L + 2;
    I2L  = INDEX - 2*L;
    I24L2 = INDEX2 - 4*L + 2;
    I22L  = INDEX2 - 2*L;
    D4LL1C = trigonom_tab(2)*sqrt((double)(4*L*L-1));
    D2L13  = -sqrt((double)(2*L+1)/(double)(2*L-3));

    for (M = L - 2; M >= 0; M--)
    {
        // 快速平方根倒数
        // 二维打表计算 sqrt（两种技术）F=7, center 外打表
        // center 移入从核？？
        TEMP1 = 1.00/sqrt((double)((L+M)*(L-M)));
        TEMP2 = D4LL1C*TEMP1;
        TEMP3 = D2L13*sqrt((double)((L+M-1)*(L-M-1)))*TEMP1;
        YLMR = TEMP2*ylm_tab(I22L) + TEMP3*ylm_tab(I24L2);
        YLMI = TEMP2*ylm_tab(I2L) + TEMP3*ylm_tab(I4L2);
        ylm_tab(INDEX2) = YLMR;
        ylm_tab(INDEX)  = YLMI;

        INDEX2 = INDEX2 - 1;
        INDEX  = INDEX  + 1;
        I24L2   = I24L2   - 1;
        I22L    = I22L    - 1;
        I4L2   = I4L2   + 1;
        I2L    = I2L    + 1;
    }
}
```

# 远程测试

1、登录天河，从路径 `/PARA/grid225/wuyangjun/performance_test_fhi-aims`下拷贝文件 `fhi-aims_sumup.tar` 到 U 盘，再拷贝入神威

2、解压缩命令：`tar -xvf 文件名`。执行 `make clean`。

3、**测试主核版本**

- 把可运行的 make.sys 拷贝一份到 src 目录下（我们之前lwpf3测试目录下的make.sys应该可以）。在 `FFLAGS` 变量上增加一个 `-cpp` 选项。
- `make -j 4 scalapack.mpi` 执行编译链接（若提示 `xxx.pl` 文件无权限，用 `chmod 777 *.pl` 授权。若还有错应该是编译器版本问题，先拍给我看看）
- 测试文件你应该知道，只要把提交脚本里的路径改为这个可执行文件路径就行

4、**测试从核版本**

- 把 `src/config.h` 中的 `#define DEV_TYPE 0` 改为 `#define DEV_TYPE 1`，从而切换为从核版本。
- `make.sh` 里，sumup.f90 的编译选项也要加上 `-cpp`
- 先 `make -j 4 scalapack.mpi` ，如果前面执行过就无需再执行。执行 `src` 目录下的 `make.sh ` 和 `link.sh`（其中 `make.sh` 可能有问题，基本是编译器版本或者配置问题，出错拍我看看）
- 结果比对：用 `vimdiff 文件1 文件2` 对从核输出结果和正确输出进行比对，搜索 `polar` 的第二个位置，看三维数组的结果相差多少，一般来说 `10^-6` 为正常。（这里给我看看）

# DMA优化

hartree_multipole_error是否可以删除？

- BUG TODO：没有用dma写回主核，但是好像结果不影响。可用变量副本来解决读写冲突问题。
- 他不影响最后的结果，但却是一个重要的中间变量，可以用**宏定义**或者**条件分支**暂时将其取消

```c
*p_hartree_multipole_error += pow(total_rho - rho_multipole(i_full_points), 2) * 										partition_tab(i_full_points);
```

n_my_batches_work 可在测试目录下的control.in 中的 point修改，后续进行参数性能调优，主要是负载均衡处理。明川在loop2中发现该值都是56/57，意味着有7/8个从核没用。

```
batch_size_limit 200
points_in_batch  100 
```





regular：静态编译时即可计算出内存地址 `A[i],A[i+4]`

irregular：动态运行才能知道内存地址，如 `A[B[i]]`

 k_points_max：Si_100_mini 作为基准



## DMA 优化开发约定

- `__thread_local` 尽量少用，尽可能不放数组；

- **统一 DMA 接口**：请使用 `src/dma_macros.h` 中的接口来交换从核和主核数据，如  `pe_get` ，`pe_put` 分别表示从主核读，写回主核，便于移植；

- **LDM 动态分配**：请使用统一栈式分配接口，具体使用方法如下：

```c
// 1.将 /home/export/online3/para030/wuyangjun/fhi-aims_cpe_dev_710/src/ldm_alloc.h 拷贝到自己的src目录下(已有请忽略该步骤)
// 2.在从核入口文件中包含头文件 ldm_alloc.h，如sum_up对于的从核文件就是sum_up_xxx_c.c
#include "ldm_alloc.h"
// 3.在从核入口函数中初始化一次即可，后面均可直接使用
ldm_alloc_init(tot_size); // 需要的ldm栈大小，以字节为单位
// 4.申请和释放，注意这是栈式分配，所以先申请的变量需后释放
int* p_ldm = (int*)ldm_alloc(alloc_size); // 这里申请alloc_size字节的空间，并转换为int*类型指针
ldm_pop_after(p_ldm); // 释放栈空间
```



因为他是开在栈上的，所以必须在函数内进行初始化，即申请数组空间。若在全局静态区开辟，则变成 LDM 静态区域了。

`ldm_alloc.h`

```c
ifndef LDM_ALLOC_H
#define LDM_ALLOC_H
#include <slave.h>
#include <assert.h>
__attribute__((weak)) __thread_local void *ldm_alloc_ptr, *ldm_alloc_base;
__attribute__((weak)) __thread_local int ldm_alloc_size;
#define ldm_alloc_init(tot_size) char ldm_alloc_buf[tot_size]; ldm_alloc_ptr = ldm_alloc_base = ldm_alloc_buf; ldm_alloc_size = tot_size;
#define UP_ALIGN_PTR(ptr) ((void*)(((long)ptr + 31) & ~31))

inline void *ldm_alloc(size_t size){
  void *ret = ldm_alloc_ptr;
  ldm_alloc_ptr = UP_ALIGN_PTR(ldm_alloc_ptr + size);
  assert(ldm_alloc_ptr - ldm_alloc_base < ldm_alloc_size);
  return ret;
}
void ldm_pop_after(void *ptr) {
  ldm_alloc_ptr = ptr;
}
#endif
```



sw5 上 link.sh 增加下列参数，可方便调试

```
-Wl,--whole-archive,-wrap,athread_init,-wrap,__expt_handler,-wrap,__real_athread_spawn /home/export/online3/swmore/release/lib/libspc.a -Wl,--no-whole-archive
```

link.sh增加以下库路径，都是用710编译的，把原来的链接删除

```
-L/home/export/online3/para030/dxh/local/lib  -lscalapack -llapack -lblas
```

```
# 轻量级线程库
/home/export/online3/swmore/opensource/qthread
```

[从核exception定位](http://bbs.nsccwx.cn/topic/108/%E5%A4%A7%E8%A7%84%E6%A8%A1%E5%B9%B6%E8%A1%8C%E4%B8%AD%E7%9A%84%E4%BB%8E%E6%A0%B8exception%E5%AE%9A%E4%BD%8D%E6%96%B9%E6%A1%88)

问题表征：

出现非法指令（signal=4），pc对应__acos，猜测是link中只链接了从核数学库-lm_slave，导致主从核都都现在加上主核数学库-lm

这个问题还是蛮诡异的。首先我link里有-lm_slave，而报错是说__acos非法指令，那既然没有-lm，acos是怎么链接成功的呢？难道m_slve里有这个函数？

段老师：链接器有bug

- libm.so 中确实包含 `__acos` 函数（`objdump -t /lib/libm.so | less` 查看） 



调试心得：

- 从昨晚到现在，最大感触是 spc.c 这个打印异常地址的重要性，这个链接上去，立马搞定许多问题。
- 所以别偷懒，工欲善其事，必先利其器
- 能否移植到 sw9 上？
- 先把信号挂载原理搞明白

## sw5上的调试神器

有些信号异常不给出 PC 值，导致问题定位困难。

- 先在主程序中挂载异常：`register_sighandler` 写在 `main.f90`（类似接管程序）
- rpcc.c 用主核编译，因为是用主核去读取从核状态
- 加入 link.sh

## 在sw5上编译sw9上可运行的aims

### 总体思路

1. 首先，改用 710 编译器（`module load sw/compiler/gcc710`），可用 `mpif90 -v` 来验证是否加载成功。因为 530 编译器不支持 `_MYID`，`malloc` 等，速度上。也比 `710` 慢了 8~10倍。 

2. 编译器影响三个库，分别为 `blas`，`scalapack`，`lapack`，都必须用 710 重新编译。 编译好记得在 `Makefile` 或者 `link.sh` 中进行修改。
3. 包含 `crts.h` 的头文件要删除，因为 `sw5` 上没该运行库。
4. 同理，`main.f90` 中的 `crts_init` 也没有，需要改成 `athread_init`。
5. 也没有从核锁，因此 H 中使用从核锁来解决读写冲突的方法不可行，于是将其转为单从核版本。

### 基准程序

**sw5，710编译器（mpif90,sw5gcc）**

> **sumup是64CPE，其余均为主核.** 

- /home/export/online3/para030/wuyangjun/fhi-aims_cpe_710

> **H是1CPE，rho是64CPE，sumup为主核**

- /home/export/online3/para030/wuyangjun/fhi-aims_CPE_O3/src

#### sw5-710编译运行步骤（仅sumup-64CPE）

- **获取项目源码**：将 `/home/export/online3/para030/wuyangjun/fhi-aims_cpe_710` 拷贝到自己目录下
- **使用最新编译器**：`module load sw/compiler/gcc710 `（**第一次进入shell编译时必须加载它**）
- **编译链接**

```shell
cd fhi-aims_cpe_710/src # 进入源码目录
make -j 4 scalapack.mpi # 该步骤链接会报错，忽略它
sh make.sh # 编译从核代码
sh link.sh # 链接主从核代码，没报错就是成功，会在../bin/目录下生成可执行文件
```

- **提交运行**

```shell
cd fhi-aims_cpe_710/test_sw5/n_100_mini/ # 进入测试目录
sh sub.sh # 提交脚本，output是输出文件，每次运行结果append到output后，可根据需要，每次提交前先删除output；out_ans 是标准输出结果，可用于对照（一般要求精度为10^-7）
```

- **结果验证**：打开 `output` 文件，搜索 `polar` 可快速定位如下部分，主要看 `xx yy zz` 这三个对应的值是否改变，不变就是对的。（**红框圈起来的必须值至少精确到小数点后面7位**）

![image-20201230123231663](C:\Users\86186\AppData\Roaming\Typora\typora-user-images\image-20201230123231663.png)

```shell
dP_dE (Bohr^3) at every cycle:--->
        (117.38992706720600,-3.53113571630931473E-014)  (-2.41051623106613988E-012,1.10429782993854329E-013)  (-2.16959783472248091E-012,1.39122305245730
 (-2.48157050464214990E-012,-1.32083922252047589E-013)         (117.38992731691920,2.70944933425449568E-015)  (-1.41320288804536176E-012,3.99078737669437
 (-2.26574314865501947E-012,-1.60078491697817760E-013) (-1.55553347980230683E-012,-4.06425338385186704E-013)        (117.38992731688820,-1.30795125050038)
DFPT polarizability (Bohr^3)        xx        yy        zz        xy        xz      yz
  | Polarizability:--->            117.390   117.390   117.390    -0.000   -0.000  -0.000

```

```shell
# 710编译器，为何一定加上 -sw3run swrun-5a?
bsub -b -m 1 -I  -cgsp 64 -q q_sw_expr -n 8  -share_size 4096 -host_stack 1024 -priv_size 32 -sw3run swrun-5a -o output ../../bin/aims.191127.scalapack.x
```



# 常用命令

```shell
# 神威：向JOBID的程序发送 30 号信号，调试常用，如死循环/不结束
bsignal -s 30 JOBID

# 从sw头文件找到对应函数的定义
grep -r "athread_spawn" /usr/sw-mpp/

# 可查询各类异常信号的代码
kill -l

# 查看具体的函数名
objdump/sw5objdump -t a.out

# 查看so依赖的动态链接库
ldd libm.so

# 将指令地址转换为代码行号
addr2line/sw5addr2line -Cfe 可执行文件名 指令地址

# 查文件名
find * | grep filename

# 查文件中的字符串
find * | xargs grep string
```

sw5

```shell
# 主核提交命令
# 加载 710 编译器：module load sw/compiler/gcc710
# 必须指定新的加载器：-sw3run swrun-5a
# 必须指定核组共享内存
bsub -share_size 4096 -q q_sw_share -n 6 -sw3run swrun-5a  -o out_host ../../bin/aims.191127.scalapack.mpi.x                           
```



# aims介绍

DFT介绍：https://baike.baidu.com/item/%E5%AF%86%E5%BA%A6%E6%B3%9B%E5%87%BD%E7%90%86%E8%AE%BA/4143952
DFPT是DFT基础上发展的目前最先进的算法，主要研究电子体系的性质，例如能量，势能，作用力等。应用广泛，如生物制药，化学分子结构，化学反应，催化反应等，固体物理用的也很多。
sumup主要做的就是在求解泊松方程（变体，很复杂），可以简单认为他是一个先进的泊松求解器。主要算法包含Multipole Expantion，Ewald summation这两种。两个i_batch分别对应倒空间（复数域）和实空间的求解。

这些也可以了解下：
DFT和DFPT都是以“第一性原理”为原理的方法，即不靠经验参数，只需几个基本的属性值就可以计算出诸多的性质，优势在于它模拟精度最高；
相对的有统计概率方法，例如最早的蒙特卡洛方法，现在很火的深度学习网络，这些需要依靠经验参数（所谓调参），他的优点在于算的快，精度可接受。
DFPT可以为概率算法提供优质的模型训练数据。

FHI-aims：全名为 Fritz Haber Institute ab initio molecular simulations

- FHI 为研究所名字
- ab initio：第一性原理（从头算）
- molecular simulations：分子模拟

# 项目申请

内容包含但不限于：背景及研究意义（说明项目当前研究背景、研究意义、主要解决的问题）、研究目标（根据基金项目的要求，提炼出研究目标）、研究方法（说明研究项目拟采用的技术路线与研究方法，突出其创新性和可落地性等优势）等。

（1）FHI-aims 为 DeepMD 提供优质的训练数据，并结合模型进行推理计算。

分子模拟在生物大分子药物研发，新材料研制等领域应用广泛，但传统的分子模拟计算方法由于其计算能力有限，已无法满足日益增长的大体系需求。FHI-aims 是基于第一性原理实现分子模拟的软件包，以其超高的模拟精度和良好的可扩展性得到广泛应用。但在超大分子体系下，计算速度依旧无法满足需求，因此，本项目将用 FHI-aims 为神经网络模型提供优质训练数据，然后再结合FHI-aims和训练好的模型完成推理计算。在创新实现方面，基于具有高效的分布式训练效率，数据预处理效率和执行效率的MindSpore 框架实现神经网络，可在昇腾处理器上将获得优异性能，同时 MindSpore 可大幅度提升能量的计算速度，从而大大提升 FHI-aims 整体的计算速度。MindSpore 支持的自动微分和自动并行使其能充分的利用昇腾处理器，可将软硬件协同优化性能提升到极致。基于源代码转换的自动微分，不仅支持便于神经网络模型构建和调试的控制流自动微分，而且容易对神经网络进行静态编译优化，以此获得获取更佳的训练和推理性能。基于算子切分的自动并行能在构建时自动选择开销最小的切分策略，实现自动分布式并行。FHI-aims 和 AI 的结合将把分子模拟的体系，精度和速度都推向一个新的高度。

（2）KMC 使用神经网络来推理计算能量。

动力学蒙特卡洛方法(KMC)能够有效地模拟原子体系的状态变迁，广泛应用于核辐照损伤，表面吸附/扩散/表面生长，统计物理等领域。但其本质是一个串行算法，严重限制了能计算的体系大小和时间尺度。本项目将基于具有高效的分布式训练效率，数据预处理效率和执行效率的 MindSpore 框架和昇腾处理器，训练能够高效计算原子体系能量的神经网络，以此将KMC可模拟的原子体系大小和时间尺度推向新的高度。在创新实现方面，MindSpore 支持的自动微分和自动并行使其能充分的利用昇腾处理器，可将软硬件协同优化性能提升到极致。基于源代码转换的自动微分，不仅支持便于神经网络模型构建和调试的控制流自动微分，而且容易对神经网络进行静态编译优化，以此获得获取更佳的训练和推理性能。基于算子切分的自动并行能在构建时自动选择开销最小的切分策略，实现自动分布式并行。



MindSpore 支持的自动微分和自动并行使其能充分的利用昇腾处理器，将软硬件协同优化性能提升到极致。基于源代码转换的自动微分，不仅支持便于图构建和调试的控制流自动微分，而且容易对神经网络进行静态编译优化，获得更佳性能。基于算子切分的自动并行能在构建时自动选择开销最小的切分策略，实现自动分布式并行。



https://github.com/deepmind/ferminet

https://github.com/deepqmc/deepqmc

```python
from tensoralloy.nn import EamAlloyNN
from tensoralloy.transformer import UniversalTransformer
elements = ['Cu', 'Fe']
rc = 6.5
nn = EamAlloyNN(elements, custom_potentials="zjw04")
clf = UniversalTransformer(elements, rc)
nn.attach_transformer(clf)
nn.export("CuFe.lmpb", export_partial_forces_model=True)
# 生成CuFe模型
from tensoralloy.nn import EamAlloyNN
from tensoralloy.transformer import UniversalTransformer

nn = EamAlloyNN(['Cu', 'Fe'], custom_potentials="zjw04", export_properties=['energy', 'forces'])
clf = UniversalTransformer(['Cu', 'Fe'], rcut=6.5)
nn.attach_transformer(clf)
nn.export("CuFe.lmpb", to_lammps=True)
```



1. 迁移tensoralloy训练端到 GPU 后端，工作量一人周
2. 铁铜性能模型（python前端，昇腾后端）工作量2人周
3. 昇腾后端铁铜模型训练，1人周
4. MindSpore C++ 推理接口（昇腾后端），工作量2人天
5. MindSpore 对接 KMC，工作量5人天

交付件：

- Tensoralloy GPU版本开源
- KMC 铁铜性能版（昇腾）



安装tensorflow1.13.1 

```
pip install https://download.tensorflow.google.cn/mac/cpu/tensorflow-1.13.1-py3-none-any.whl --upgrade
```



# SIMD

SIMD（Single Instruction Multiple Data）单指令多数据流。

SW39000

- 众核处理器主核：256 bit
- 众核处理器从核：512 bit

使用条件：

- 必须在内存中连续的数据可以取入向量寄存器进行向量运算
- 数据在内存中要求32字节对界（主核32B对齐，从核64B对齐），不对界的向量访存会引起异常，之后由操作系统模拟，降低性能





Fp数组相关：

sum_up_trans_private_variable_c.c 函数原型在这
hartree_potential_real_p0.f90：trans_Fp_c

# 调试

## 宏调试

为了快速追踪诱发问题的文件，函数，行数，除了传统的堆栈追踪以外，还有一种简单得方法，即宏调试。

即最小改动原则，在函数定义之后增加一句宏，保证之后每个调用函数的时候都验证一次条件，出现错误即可避免堆栈跟踪。

main.c

```c
#include <stdio.h>
#include <assert.h>
#include "my_assert.h"
int main()
{
        cal(10);
        return 0;
}
```

my_assert.h

```c
void cal(int a)
{
        printf("%d\n", a);
}
#define cal(a) \
        cal((a)); \
        assert((a)<8)
```



## Linux命令

统计行数，单词数、字节数

```shell
wc -l # 行数
wc -L # 最长行的长度
wc -w # 单词数
wc -c # 字节数
```



计算文件的校验和，文件占用的块数。在防止篡改文件很有用

```
sum 文件名
```



打印匹配给定模式的那一行

```
grep 匹配模式 输入字符串/文件名
-C NUM 打印匹配行的上下文前后各 NUM 行。相邻组会打印一行标记： `---` 
-B NUM 打印匹配行的上文 NUM 行
-A NUM 打印匹配行的下文 NUM 行
```



awk 强大的文本处理工具，可指定分隔符 
