# Debian从源码编译deb包简单介绍

下面以示例程序简单说明如何从源码编译为deb包，详细信息请参考 [Debian 新维护人员手册](https://www.debian.org/doc/manuals/maint-guide/index.zh-cn.html)


--------------
### 1. 环境准备  
Fedora和CentOS等基于rpm的操作系统，可以使用 `yum install dpkg-dev fakeroot dh-make` 安装所需的命令  
Ubuntu和Deepin等基于dpkg的操作系统，可以使用 `apt-get install devscripts dh-make` 安装所需的命令

源码所在的目录，要求使用 `<package>-<version>` 这样的格式，例如下面的 `hello-0.1`

源码的目录内容如下：

    /tmp/hello-0.1$ ls  
    hello.c  Makefile

Makefile文件内容

    /tmp/hello-0.1$ cat Makefile      
    SRCS = $(wildcard *.c)
    OBJS = $(SRCS:.c = .o)
    CC = gcc
    CCFLAGS = -g -Wall -O0
    
    hello : $(OBJS)
	    $(CC) $^ -o $@ 
        
    clean:
	    rm -f hello
        
    .PHONY:clean
    
 hello.c文件内容  
 
    /tmp/hello-0.1$ cat hello.c
    #include <stdio.h>
    
    int main(void)
    {
	    printf("Hello, World!\n");
	    return 0;
    }

----------------
### 2. 生成源码压缩包和debian目录  
输入下面的命令，并且分别选择 **s** 和 **y** 两个选项
    
    
    dh_make --createorig
    
   得到这样的结果
   
   
    /tmp/hello-0.1$ find .
    .
    ./debian
    ./debian/hello.cron.d.ex
    ./debian/hello.doc-base.EX
    ./debian/hello.default.ex
    ./debian/hello-docs.docs
    ./debian/control
    ./debian/copyright
    ./debian/compat
    ./debian/changelog
    ./debian/README.source
    ./debian/rules
    ./debian/postinst.ex
    ./debian/init.d.ex
    ./debian/postrm.ex
    ./debian/preinst.ex
    ./debian/manpage.xml.ex
    ./debian/watch.ex
    ./debian/README.Debian
    ./debian/menu.ex
    ./debian/prerm.ex
    ./debian/manpage.1.ex
    ./debian/manpage.sgml.ex
    ./debian/source
    ./debian/source/format
    ./hello.c
    ./Makefile
    
-------------------- 
### 3. 处理debian目录  
先 `cd debian` ，然后删除不需要的文件，执行


    rm *.{ex,EX,docs} && rm README*
    
之后debian目录大概是这样的

    /tmp/hello-0.1/debian$ ls
    changelog  compat  control  copyright  rules  source
    
剩下的文件中 changelog，control和rules文件是需要我们修改的，其他的文件可以暂时无视  
所有文件的具体作用可以参考 [Debian 新维护人员手册](https://www.debian.org/doc/manuals/maint-guide/index.zh-cn.html)  
 
先修改 changelog 文件，基本内容可以修改成这样
 
    /tmp/hello-0.1/debian$ cat changelog 
    hello (0.1-1) unstable; urgency=medium
    
    * Initial release.

    -- deepin <deepin@deepin>  Fri, 14 Oct 2016 13:55:12 +0800
    
文件第1行需要注意的是的是deb包的名字(`hello`)和版本号(`0.1-1`)，必须和目录的名字(`hello-0.1`)相符合  
第5行的维护者姓名和邮箱可以随意修改，时间使用默认生成的即可

然后修改 control 文件，删除不需要新信息后，基本内容可以修改成这样

    /tmp/hello-0.1/debian$ cat control 
    Source: hello
    Section: devel
    Priority: optional
    Maintainer: deepin <deepin@deepin>
    Build-Depends: debhelper (>=9)
    Standards-Version: 3.9.8
    
    Package: hello
    Architecture: any
    Depends: ${shlibs:Depends}, ${misc:Depends}
    Description: Hello
     hello world
     
最后是 rules 文件，rules 文件实际上是一个makefile  
rules文件的主要内容基本上默认的Makefile已经完成，所以这里删除注释，简单调用即可  
    
    #!/usr/bin/make -f
    #export DH_VERBOSE = 1
    
    %:
        dh $@

为了告诉dpkg这个源码包所生成的二进制文件安装到哪里，需要添加一个文件 install  
这个文件特地用来说明哪些文件会被放到deb包中的哪个位置  
这个事情也可以在 rules 文件中完成，和单独的 install 文件功能一致  
这个 hello 程序的 install 文件可以写成这样

    /tmp/hello-0.1/debian$ cat install 
    hello usr/bin


简单说明一下 install 文件的格式  
`hello` 是指从 `hello-0.1` 这个目录开始计算的 **相对路径**所对应的文件路径  
如果 `hello` 这个二进制文件是存放在 `hellon-0.1/build/` 目录下，这个 hello 就需要写成  `build/hello`  

`usr/bin` 是同一行前面的 `hello` 文件所放到的deb包内的路径(也就是安装到系统内的路径)  
这个路径也是有操作系统的根路径 `/` 所对应的 **相对路径**，`usr/bin` 表示会被安装到系统的 `/usr/bin/` 目录下  

总结起来，这一行 `hello usr/bin` 的意思是把源码目录下的文件 `hello` 放到deb包的 `usr/bin/` 目录下，并且在安装这个deb包的时候安装到系统的 `/usr/bin/` 目录下

还可以多写一行，保留这里编译的源码文件 `hello.c`，这样的install文件内容可以是这样

    /tmp/hello-0.1/debian$ cat install 
    hello usr/bin
    hello.c usr/share/hello
    
这样生成的deb包不仅包括二进制文件，也包括生成二进制文件的源码  

上面这种源码文件和二进制文件放在同一个deb一般是不需要的，这里只是举例说明，源码应该放到另外一个叫 `hello-dev` 的deb包中

完成修改之后debian目录应该是这样的  
    
    /tmp/hello-0.1/debian$ ls
    changelog  compat  control  copyright  install  rules  source


---------------------------
### 4. 编译源码生成deb

使用dpkg的编译生成包的命令 `dpkg-buildpackage` 即可，例如

    dpkg-buildpackage -rfakeroot -sa -us -uc
    
`dpkg-buildpackage` 命令的具体参数可以使用 `man dpkg-buildpackage` 查看

生成的deb包，`hello_0.1-1_amd64.deb`，会放到 `hello-0.1` 的同级目录，可以查看其具体内容

    /tmp/hello-0.1$ dpkg-deb -c ../hello_0.1-1_amd64.deb 
    drwxr-xr-x root/root         0 2016-10-14 14:57 ./
    drwxr-xr-x root/root         0 2016-10-14 14:57 ./usr/
    drwxr-xr-x root/root         0 2016-10-14 14:57 ./usr/bin/
    -rwxr-xr-x root/root      4320 2016-10-14 14:57 ./usr/bin/hello
    drwxr-xr-x root/root         0 2016-10-14 14:57 ./usr/share/
    drwxr-xr-x root/root         0 2016-10-14 14:57 ./usr/share/doc/
    drwxr-xr-x root/root         0 2016-10-14 14:57 ./usr/share/doc/hello/
    -rw-r--r-- root/root       128 2016-10-14 14:24 ./usr/share/doc/hello/changelog.Debian.gz
    -rw-r--r-- root/root      1676 2016-10-14 14:01 ./usr/share/doc/hello/copyright
    drwxr-xr-x root/root         0 2016-10-14 14:57 ./usr/share/hello/
    -rw-r--r-- root/root        78 2016-10-14 11:07 ./usr/share/hello/hello.c
    
    
可以看到deb包的内容与之前的install文件所预期的目录和文件相同