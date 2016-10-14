### 静态编译ffmpeg

这两天折腾了一下完全静态编译ffmpeg，期望通过这种方式实现编译出来的ffmpeg能够在各个linux发现版上运行。进而可以延伸到编译出来完全静态的依赖ffmpeg库的应用程序，从而简化产品在各个linux发行版的产品编译和部署区分，减少开发和维护成本。

经过尝试，最终确实编译出来完全静态的ffmpeg，但是也发现了隐患，具体在"一些问题"中描述。所以，我对完全静态编译持保留意见，需要根据实际的使用情况来定。下面的文字，就是我整理的本次尝试的要点了。


#### 1. 静态编译

首先，查看一下之前编译出来的ffmpeg的动态库链接情况。可以看出它链接了许多动态库，这些库有些是系统自带的，有的需要自己安装。

```bash
[root@root tmp]# ldd ffmpeg
    linux-vdso.so.1 =>  (0x00007fff133fe000)
    libm.so.6 => /lib64/libm.so.6 (0x00007fb094f81000)
    libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fb094d65000)
    librt.so.1 => /lib64/librt.so.1 (0x00007fb094b5c000)
    libdl.so.2 => /lib64/libdl.so.2 (0x00007fb094958000)
    libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007fb094650000)
    libnuma.so.1 => /lib64/libnuma.so.1 (0x00007fb094444000)
    liblzma.so.5 => /lib64/liblzma.so.5 (0x00007fb09421f000)
    libbz2.so.1 => /lib64/libbz2.so.1 (0x00007fb09400f000)
    libz.so.1 => /lib64/libz.so.1 (0x00007fb093df8000)
    libc.so.6 => /lib64/libc.so.6 (0x00007fb093a36000)
    /lib64/ld-linux-x86-64.so.2 (0x00007fb09528c000)
    libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007fb093820000)

```

而完全静态的ffmepg是这样的：

```bash
[root@root tmp]# ldd ffmpeg
    not a dynamic executable
```

下面就是静态编译的要点，具体可以参考文档结尾附的编译脚本内容。


1.1 安装标准静态库

安装linux c标准库: yum install glibc-static

安装stdc++静态库:  yum install libstdc++-static


1.2 编译第三方静态库

ffmpeg通常会使用一些第三方库，例如: x264, x265, libfdk-aac等。这些第三方库通常都是可以通过configure配置（--enable-static=yes --enable-shared=no）控制产生静态库，不创建动态库。对于一些没有configure的库，其实也是有编译出静态库的方法，注意发现就OK。该过程的编译脚本，可以参考文后附的编译脚本内容。



1.3 编译静态ffmpeg

具体的configure配置，可以参考附的编译脚本内容。


#### 2. 一些问题

在编译过程中遇到的大部分问题都是链接库出错，对于这种问题，基本就是安装或编译需要的静态库，偶尔也可能是库的顺序问题。

不过，在尝试编译支持nvenc的时候，出现了**warning: Using 'dlopen' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking**警示信息，因为已经编译出静态文件，所以当时并没有在意。不过在后来的测试中，发现在使用nvenc环境进行nvenc编解码操作的的时候，程序会段错误。原来，像"dlopen"这样的函数，是依赖运行环境的，需要使用运行环境的动态库。故，完全静态编译存在这样的隐患，就是有些函数依赖运行环境的动态库(其实这些库本身也是为了兼容性，不同平台提供相同的接口)，所以，标准库除非要求，否则尽量使用动态库吧。


#### 3. 附静态ffmpeg下载

ffmpeg: http://pan.baidu.com/s/1kVfOY7H

ffprobe: http://pan.baidu.com/s/1boS8cJt


#### 4. 附静态ffmpeg编译脚本
```shell
#!/bin/bash
set -e

current_dir=$(cd ../; pwd -P)
build_dir="${current_dir}/_build"
release_dir="${current_dir}/_release"
echo "start to build the tools for transcode system(current_dir: ${current_dir}, build_dir: ${build_dir}, release_dir: ${release_dir})..."

mkdir -p ${build_dir}
mkdir -p ${release_dir}

cp -rf yasm-1.3.0.tar.gz fdk-aac-0.1.4.tar.gz faac-1.28.tar.bz2 lame-3.98.4.tar.gz opencore-amr-0.1.2.tar.gz x264-snapshot-20140803-2245.tar.bz2 x265_1.9.tar.gz ${build_dir}
cp -rf libxml2-2.9.4.tar.gz libpng-1.6.24.tar.xz freetype-2.7.tar.bz2 fribidi-0.19.7.tar.bz2 fontconfig-2.12.1.tar.bz2 libass-0.13.2.tar.gz ffmpeg-3.0.1.tar.bz2 ${build_dir}
cp -rf zlib-1.2.8.tar.gz bzip2-1.0.6.tar.gz libevent-2.0.22-stable.tar.gz xz-5.2.2.tar.gz numactl_2.0.8.orig.tar.gz ${build_dir}

export PKG_CONFIG_PATH=${PKG_CONFIG_PATH}:${release_dir}/lib/pkgconfig
export PATH=${PATH}:${release_dir}/bin

# libaacplus not support in ffmpeg3.0.1



# yasm 
pushd ${build_dir}
if ! [ -e "yasm" ]
then
    echo "########## yasm begin ##########"
    if ! [ -e "yasm-1.3.0.tar.gz" ]
    then
        # download yasm
        echo "########## to download yasm ##########"
        wget http://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz
    fi

    tar xf yasm-1.3.0.tar.gz
    pushd yasm-1.3.0
    ./configure --prefix=${release_dir}
    make
    make install
    popd
    touch yasm
    echo "########## yasm ok ##########"
else
    echo "########## yasm has been installed ##########"
fi
popd

# libevent
pushd ${build_dir}
if ! [ -e "libevent" ]
then
    echo "########## libevent begin ##########"

    tar xf libevent-2.0.22-stable.tar.gz
    pushd libevent-2.0.22-stable
    ./configure --prefix=${release_dir}  --enable-static=yes --enable-shared=no
    make
    make install
    popd
    touch libevent
    echo "########## libevent ok ##########"
else
    echo "########## libevent has been installed ##########"
fi
popd

# libz
pushd ${build_dir}
if ! [ -e "zlib" ]
then
    echo "########## zlib begin ##########"
    # wget http://zlib.net/zlib-1.2.8.tar.gz  

    tar xf zlib-1.2.8.tar.gz
    pushd zlib-1.2.8
    ./configure --prefix=${release_dir} --static
    make
    make install
    popd
    touch zlib
    echo "########## zlib ok ##########"
else
    echo "########## zlib has been installed ##########"
fi
popd


# libbz2
pushd ${build_dir}
if ! [ -e "libbz2" ]
then
    echo "########## libbz2 begin ##########"
    # wget http://www.bzip.org/1.0.6/bzip2-1.0.6.tar.gz

    tar xf bzip2-1.0.6.tar.gz
    pushd bzip2-1.0.6
    make PREFIX=${release_dir}
    make PREFIX=${release_dir} install
    popd
    touch libbz2
    echo "########## libbz2 ok ##########"
else
    echo "########## libbz2 has been installed ##########"
fi
popd


# fdk-aac
pushd ${build_dir}
if ! [ -e "fdk-aac" ]
then
    echo "########## fdk-aac begin ##########"
    tar xf fdk-aac-0.1.4.tar.gz
    pushd fdk-aac-0.1.4
    ./autogen.sh 
    ./configure --prefix=${release_dir}
    make
    make install
    popd
    touch fdk-aac
    echo "########## fdk-aac ok ##########"
else
    echo "########## fdk-aac has been installed ##########"
fi
popd



# libfaac
pushd ${build_dir}
if ! [ -e "faac" ]
then
    echo "########## libfaac begin ##########"
    tar xf faac-1.28.tar.bz2
    pushd faac-1.28
    ./configure --prefix=${release_dir} --enable-static --without-mp4v2
    make
    make install
    popd
    touch faac
    echo "########## libfaac ok ##########"
else
    echo "########## libfaac has been installed ##########"
fi
popd


# libmp3lame
pushd ${build_dir} 
if ! [ -e "mp3lame" ]
then
    echo "########## libmp3lame begin ##########"
    tar xf lame-3.98.4.tar.gz
    pushd lame-3.98.4
    ./configure --prefix=${release_dir} --enable-static
    make
    make install
    popd
    touch mp3lame
    echo "########## libmp3lame ok ##########"
else
    echo "########## libmp3lame has been installed ##########"
fi
popd

# libopencore_amrnb
pushd ${build_dir} 
if ! [ -e "opencore_amrnb" ]
then
    echo "########## libopencore_amrnb begin ##########"
    tar xf opencore-amr-0.1.2.tar.gz
    pushd opencore-amr-0.1.2
    ./configure --prefix=${release_dir} --enable-static
    make
    make install
    popd
    touch opencore_amrnb
    echo "########## libopencore_amrnb ok ##########"
else
    echo "########## libopencore_amrnb has been installed ##########"
fi
popd


# libx264
pushd ${build_dir} 
if ! [ -e "x264" ]
then
    echo "########## libx264 begin ##########"
    tar xf x264-snapshot-20140803-2245.tar.bz2
    pushd x264-snapshot-20140803-2245
    ./configure --prefix=${release_dir} --enable-static --disable-opencl 

    sed -i -e 's/-s //' -e 's/-s$//' config.mak
    make
    make install
    popd
    touch x264
    echo "########## libx264 ok ##########"
else
    echo "########## libx264 has been installed ##########"
fi
popd


# libnuma (for x265)
pushd ${build_dir}
    if ! [ -e "numa" ]
    then

        if ! [ -e "numactl_2.0.8.orig.tar.gz" ]
        then
            # download yasm
            echo "########## to download numa ##########"
            wget http://numactl.sourcearchive.com/downloads/2.0.8/numactl_2.0.8.orig.tar.gz
        fi

        tar xf numactl_2.0.8.orig.tar.gz
        pushd numactl-2.0.8
        make PREFIX=${release_dir}
        make PREFIX=${release_dir} install
        rm -rf ${release_dir}/lib64/libnuma.so*
        popd

        touch numa
        echo "########## numa ok ##########"
    else
        echo "########## numa has been built ##########"
    fi
popd

# libx265
pushd ${build_dir} 
if ! [ -e "x265" ]
then
    echo "########## libx265 begin ##########"
    # download page: https://bitbucket.org/multicoreware/x265/downloads
    tar xf x265_1.9.tar.gz
    pushd x265_1.9
    cmake ./source -DCMAKE_INSTALL_PREFIX=${release_dir} -DBUILD_SHARED_LIBS=OFF
    make
    make install
    popd
    touch x265
    echo "########## libx265 ok ##########"
else
    echo "########## libx265 has been installed ##########"
fi
popd


# lzma (requried by ffmpeg drawtext)
pushd ${build_dir} 
if ! [ -e "lzma" ]
then
    echo "########## lzma begin ##########"

    tar xf xz-5.2.2.tar.gz
    pushd xz-5.2.2

    ./configure --prefix=${release_dir} --enable-static=yes --enable-shared=no

    make
    make install
    popd
    touch lzma
    echo "########## lzma ok ##########"
else
    echo "########## lzma has been installed ##########"
fi
popd


# libpng (requried by freetype)
pushd ${build_dir} 
if ! [ -e "libpng" ]
then
    echo "########## libpng begin ##########"
    echo "remove all so to force the ffmpeg to build in static"
    rm -f ${release_dir}/lib/*.so*


    tar xf libpng-1.6.24.tar.xz
    pushd libpng-1.6.24
    ./configure --prefix=${release_dir} --enable-static=yes --enable-shared=no
    make
    make install
    popd
    touch libpng
    echo "########## libpng ok ##########"
else
    echo "########## libpng has been installed ##########"
fi
popd

# libxml2 (requried by fontconfig)
pushd ${build_dir} 
if ! [ -e "libxml2" ]
then
    echo "########## libxml2 begin ##########"
    echo "remove all so to force the ffmpeg to build in static"
    rm -f ${release_dir}/lib/*.so*

    tar xf libxml2-2.9.4.tar.gz
    pushd libxml2-2.9.4
    ./configure --prefix=${release_dir} --enable-static=yes --enable-shared=no
    make
    make install
    popd
    touch libxml2
    echo "########## libxml2 ok ##########"
else
    echo "########## libxml2 has been installed ##########"
fi
popd

# freetype (requried by libass)
pushd ${build_dir} 
if ! [ -e "freetype" ]
then
    echo "########## freetype begin ##########"
    echo "remove all so to force the ffmpeg to build in static"
    rm -f ${release_dir}/lib/*.so*


    # wget http://downloads.sourceforge.net/freetype/freetype-2.7.tar.bz2
    tar xf freetype-2.7.tar.bz2
    pushd freetype-2.7
    ./configure --prefix=${release_dir} --enable-static=yes --enable-shared=no
    make
    make install
    popd
    touch freetype
    echo "########## freetype ok ##########"
else
    echo "########## freetype has been installed ##########"
fi
popd

# fribidi (requried by libass)
pushd ${build_dir} 
if ! [ -e "fribidi" ]
then
    echo "########## fribidi begin ##########"
    echo "remove all so to force the ffmpeg to build in static"
    rm -f ${release_dir}/lib/*.so*

    tar xf fribidi-0.19.7.tar.bz2
    pushd fribidi-0.19.7
    ./configure --prefix=${release_dir} --enable-static=yes --enable-shared=no
    make
    make install
    popd
    touch fribidi
    echo "########## fribidi ok ##########"
else
    echo "########## fribidi has been installed ##########"
fi
popd

# fontconfig (requried by libass)
pushd ${build_dir} 
if ! [ -e "fontconfig" ]
then
    echo "########## fontconfig begin ##########"
    echo "remove all so to force the ffmpeg to build in static"
    rm -f ${release_dir}/lib/*.so*

    tar xf fontconfig-2.12.1.tar.bz2
    pushd fontconfig-2.12.1
    ./configure --prefix=${release_dir} --enable-static=yes --enable-shared=no --enable-libxml2
    make
    make install
    popd
    touch fontconfig
    echo "########## fontconfig ok ##########"
else
    echo "########## fontconfig has been installed ##########"
fi
popd

# libass
pushd ${build_dir} 
if ! [ -e "libass" ]
then
    echo "########## libass begin ##########"
    echo "remove all so to force the ffmpeg to build in static"
    rm -f ${release_dir}/lib/*.so*


    tar xf libass-0.13.2.tar.gz
    pushd libass-0.13.2
    ./configure --prefix=${release_dir} --enable-static=yes --enable-shared=no
    make
    make install
    popd
    touch libass
    echo "########## libass ok ##########"
else
    echo "########## libass has been installed ##########"
fi
popd



# ffmpeg
pushd ${build_dir} 
if ! [ -e "ffmpeg3.0.1" ]
then
    echo "########## ffmepg begin ##########"
    set -x

    if ! [ -d "ffmpeg-3.0.1" ]
    then
        tar xf ffmpeg-3.0.1.tar.bz2
    fi
    
    echo "remove all so to force the ffmpeg to build in static"
    rm -f ${release_dir}/lib/*.so*


    pushd ffmpeg-3.0.1

    export ffmpeg_exported_release_dir=${release_dir}
    echo ${ffmpeg_exported_release_dir}/include
    echo ${ffmpeg_exported_release_dir}/lib


./configure --prefix=${release_dir} --cc=$CC \
--extra-cflags="-I${release_dir}/include -I${release_dir}/include/hiredis" \
--extra-ldflags="-L${release_dir}/lib -L${release_dir}/lib64 -ldl -lm -lpthread -lrt -lstdc++ -static" \
--pkg-config-flags="--static" \
--enable-gpl --enable-static --enable-nonfree --enable-version3 --disable-ffplay --disable-ffserver \
--enable-postproc \
--enable-demuxer=oss \
--disable-vaapi --disable-indev=alsa --disable-outdev=alsa \
--enable-libopencore-amrnb --enable-libmp3lame --enable-libx264 --enable-libx265 --enable-libfaac --enable-libfdk-aac \
--enable-libass --enable-libfreetype --enable-libfontconfig --enable-libfribidi \
--extra-libs=-lhiredis --extra-libs=-lnuma --extra-libs=-levent 
#--extra-libs=-lstdc++ --extra-libs=-lc



    echo "ffmpeg3.0.1 begin make"
    make
    make install
    popd
    #touch ffmpeg3.0.1
    echo "########## ffmpeg3.0.1 ok ##########"
else
    echo "########## ffmpeg3.0.1 has been installed ##########"
fi
popd



```