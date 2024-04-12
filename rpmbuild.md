---
title: rpmbuild概要 
date: 2024-03-30 09:46:11
tags: [rpm,rpmbuild,spec]
categories: rpm
---

简要记录如何编写spec文件来打rpm包。

# 安装 (1)

以CentOS-7为例：

```
yum install -y rpmdevtools.noarch rpmlint.noarch
rpmdev-setuptree
```

# Spec文件的结构 (2)

首先，假设要打多个rpm包，例如

- `lua-5.3.6-1.el7.x86_64.rpm`
- `lua-devel-5.3.6-1.el7.x86_64.rpm`
- `lua-static-5.3.6-1.el7.x86_64.rpm`

第一个叫main package。打单个rpm包(main package)是打多个包的特例，不比细说。

Spec文件主要包含：

- 元信息：
    - `Name`, `Version`, `Release`, `License`, `URL`, `Source0`, `Source1`, ..., `Patch0`, `Patch1`, ..., `BuildRequires`
    - `Summary`, `Group`, `Provides`, `Requires`
    - 每个package的元信息写在`%package ${package-name}`的下面，main package除外，可以认为默认是main的元信息。
    - 上面我故意把元信息分层2部分(`Name`开头的一行和`Summary`开头的一行)。通常，只有main包含第1部分，其它package不须重复，因为一般情况下多个package是一个工程编译构建出来的，源代码(Source)、Patch、编译依赖(BuildRequires)是共同的。
- 编译前准备(`%prep`段)：一般是解压并进入源代码目录(`%setup`命令)，安装patch(`%patch0`命令)，也可以添加一些自定义操作(用bash写即可)；
- 编译(`build`段)：前一步已经进入源代码目录，现在开始编译。一般linux程序的编译步骤是`./configure && make`(用bash写即可)。Rpmbuild工具也提供了一个`%configure`命令，应该是调用`./configure`。
- 安装到buildroot(`%install`段)：
    - 通常是`make install DESTDIR=%{buildroot}`
    - 注意，这不是`rpm -hiv`，也不是在**目标机器**上执行。从命令`make install`也可以看出来，是在**编译机器**上是把编译产生的binaries, headers, libraries, docs, scripts等文件拷贝到buildroot(通常是/root/rpmbuild/BUILDROOT/{name}-{release}.{arch})目录。
- 定义每个package包含哪些文件(`%files`段)：例如main应该包含binaries和一些libraries, scripts及docs；devel package应该包含headers和libraries等等。注意：**上一步install拷贝的文件要被全覆盖**，否则报错。
- 安装前脚本(`%pre`段)：在**目标机器**上执行。执行`rpm -hiv`时拷贝文件之前的动作。
- 安装后脚本(`%post`段)：在**目标机器**上执行。执行`rpm -hiv`时拷贝文件之后的动作。例如，安装可执行包之后把binaries的目录添加到`PATH`；安装devel package之后执行`ldconfig`建立libraries索引；
- 删除前脚本(`preun`段)：在**目标机器**上执行。执行`rpm -e`时删除文件之前的动作。
- 删除后脚本(`%postun`段)：在**目标机器**上执行。执行`rpm -e`时删除文件之后的动作。通常是`%post`段的反操作，例如把binaries的目录从`PATH`删除；删除devel package之后再次执行`ldconfig`更新索引；
- 清理(`%clean`段): 通常是`rm -rf ${buildroot}`.
- 修改日志(`%changelog`段)：每个release的更新记录；

Spec skeleton:

```spec
#%package foo declaration omitted
Name:           foo
Version:        1.0
Release:        1%{?dist}
Summary:        Stupid foo
Group:          Development
License:        MIT
URL:            http://www.foo.org/
Source0:        foo-%{version}.tar.gz
Patch0:         foo-%{version}-bugfix.patch
BuildRequires:  bar-devel buz-devel
#package foo provides foo-1.0
Provides:       foo = 1.0

#should be "%description foo", but package name ("foo") omitted
%description
Foo is a super stupid foo project.

%prep
%setup -q
%patch0 -p1

%build
./configure --prefix=/usr/local
make -j30

%install
rm -rf %{buildroot}
make install DESTDIR=%{buildroot}

#should be "%files foo", but package name ("foo") omitted
%files
/usr/local/bin/*
/usr/local/libexec/*
/usr/local/share/*

#should be "%pre foo", but package name ("foo") omitted
%pre

#should be "%post foo", but package name ("foo") omitted
%post

#should be "%preun foo", but package name ("foo") omitted
%preun

#should be "%postun foo", but package name ("foo") omitted
%postun

#package declaration
%package devel
Summary:        Development files for %{name}
Group:          System Environment/Libraries
Requires:       %{name} = %{version}-%{release}
Requires:       pkgconfig

%description devel
This package contains development files for %{name}.

%files devel
/usr/local/include/*
/usr/local/lib/lib*.so

%pre devel

%post devel
ldconfig

%preun devel

%postun devel
ldconfig

#package declaration 
%package another
Summary:  ...
Group:    ...
Requires: ...

%description another

%files another

%pre another

%post another

%preun another

%postun another

%clean
rm -rf %{buildroot}

%changelog
```

# 案例 (3)


```
%bcond_without local_install

#package declaration omitted:
#%package lua

Name:           lua
Version:        5.3.6
Release:        1%{?dist}
Summary:        Powerful light-weight programming language
Group:          Development/Languages
License:        MIT
URL:            http://www.lua.org/
Source0:        http://www.lua.org/ftp/lua-%{version}.tar.gz
Patch0:         lua-5.3.6-makefile.patch
BuildRoot:      %{_tmppath}/%{name}-%{version}-%{release}-root-%(%{__id_u} -n)
BuildRequires:  readline-devel ncurses-devel
#package lua provides lua-5.3 and lua(abi)-5.3
Provides:       lua = 5.3
Provides:       lua(abi) = 5.3

#short for "%description lua", main package name omitted
%description
Lua is a powerful light-weight programming language.

#The %prep section specifies how to prepare the build environment, this usually involves:
#    - expansion of compressed archives of the source code;
#    - application of patches
#    - potentially, parsing of information provided in the source code for use in a later portion of the SPEC.
#Here we simply use the built-in macro %setup -q.

#Macro %setup can be used to build the package with source code tarballs, tipically
#
#          cd /root/rpmbuild/BUILD
#          rm -rf lua-5.3.6
#          /usr/bin/gzip -dc /root/rpmbuild/SOURCES/lua-5.3.6.tar.gz
#          /usr/bin/tar -xf -
#          STATUS=0
#          '[' 0 -ne 0 ']'
#          cd lua-5.3.6
#          /usr/bin/chmod -Rf a+rX,u+w,g-w,o-w .
#
#It
#    - ensures that we are working in the right directory;
#    - removes residues of previous builds;
#    - unpacks the source tarball;
#    - and sets up some default privileges.
#There are multiple options to adjust the behavior of the %setup macro.
#    -q: limits verbosity of %setup macro
#    -n: In some cases, the directory from expanded tarball has a different name than expected %{name}-%{version}, this
#        can lead to an error of the %setup macro. The name of a directory has to be specified by "-n {directory_name}".
#    -c: used if the source code tarball does not contain any subdirectories, thus after unpacking, files from an archive
#        fill the current directory; the -c option creates the directory and steps into the archive expansion;
#    ...

%prep
%setup -q
%patch0 -p1

%build
make linux

#install to %{buildroot} (normally /root/rpmbuild/BUILDROOT/)
%install
rm -rf %{buildroot} 
%if 0%{with local_install}
make install DESTDIR=%{buildroot} INSTALL_TOP=/usr/local
%else
make install DESTDIR=%{buildroot} INSTALL_TOP=/usr
%endif

#now we have
#    /root/rpmbuild/BUILD/lua-5.3.6/
#        ├── doc
#        ├── Makefile
#        ├── README
#        └── src
#
#    /root/rpmbuild/BUILDROOT/lua-5.3.6-1.el7.x86_64/
#    └── usr (or usr/local, depending on local_install)
#        ├── bin
#        │   ├── lua
#        │   └── luac
#        ├── include
#        │   ├── lauxlib.h
#        │   ├── luaconf.h
#        │   ├── lua.h
#        │   ├── lua.hpp
#        │   └── lualib.h
#        ├── lib64
#        │   ├── liblua-5.3.a
#        │   ├── liblua-5.3.so
#        │   ├── liblua.so -> liblua-5.3.so
#        │   └── lua
#        ├── man
#        │   └── man1
#        └── share
#            ├── doc
#            └── lua
#
# "%files" and "%files devel" directives include them in different packages respectively.


#these macros are defined in /usr/lib/rpm/platform/x86_64-linux/macros;
#                 package install location                     from where of BUILDROOT
#   %{_bindir}        = /usr/bin                     /root/rpmbuild/BUILDROOT/lua-5.3.6-1.el7.x86_64/usr/bin
#   %{_libdir}        = /usr/lib64                   /root/rpmbuild/BUILDROOT/lua-5.3.6-1.el7.x86_64/usr/lib64
#   %{_mandir}        = /usr/man                     /root/rpmbuild/BUILDROOT/lua-5.3.6-1.el7.x86_64/usr/man
#   %{_datarootdir}   = /usr/share                   /root/rpmbuild/BUILDROOT/lua-5.3.6-1.el7.x86_64/usr/share
#   %{_datadir}       = /usr/share                   /root/rpmbuild/BUILDROOT/lua-5.3.6-1.el7.x86_64/usr/share
#   %{_includedir}    = /usr/include                 /root/rpmbuild/BUILDROOT/lua-5.3.6-1.el7.x86_64/usr/include
#   %{_defaultdocdir} = /usr/share/doc               /root/rpmbuild/BUILDROOT/lua-5.3.6-1.el7.x86_64/usr/share/doc
#
#without local_install, we install it to /usr, the default values are fine;
#with local_install, we install it to /usr/local, thus we need to set the values;

%if 0%{with local_install}
%define _bindir /usr/local/bin
%define _libdir /usr/local/lib64
%define _mandir /usr/local/share/man
%define _datarootdir /usr/local/share
%define _datadir /usr/local/share
%define _includedir /usr/local/include
%define _defaultdocdir /usr/local/share/doc
%endif

#short for "%files lua"
%files
#set default attributes for files and directories:
#  %defattr(<file mode>, <user>, <group>, <dir mode>)
#      dash(-):  do not need to be set
#set attributes for a specific file:
#  %attr(<mode>, <user>, <group>) file
%defattr(-,root,root,-)

#RPM keeps track of documentation files in its database, so that a user can easily find information about an installed
#package. In addition, RPM can create a package-specific documentation directory during installation and copy documentation
#into it. The %doc directive flags the filename(s) that follow, as being documentation.
#The file README and dir "doc/" exist in /root/rpmbuild/BUILD/lua-5.3.6/, we include them "as documentation" files in the
#package file. When the package is installed, RPM creates a dir /usr/share/doc/lua-5.3.6/ and copies those files there.
#(why /usr/share/doc? see rpm --showrc | grep _defaultdocdir).
%doc README doc/*.html doc/*.css doc/*.gif doc/*.png
%{_bindir}/lua*
%attr(755,root,root) %{_libdir}/liblua-*.so
%{_mandir}/man1/lua*.1*
%dir %{_libdir}/lua
%dir %{_libdir}/lua/5.3
%dir %{_datadir}/lua
%dir %{_datadir}/lua/5.3

#package devel ......
%package devel
Summary:        Development files for %{name}
Group:          System Environment/Libraries
Requires:       %{name} = %{version}-%{release}
Requires:       pkgconfig

%description devel
This package contains development files for %{name}.

%files devel
%defattr(-,root,root,-)
%{_includedir}/l*.h
%{_includedir}/l*.hpp
%{_libdir}/liblua.so
%{_libdir}/pkgconfig/*.pc

#package static ......
%package static
Summary:        Static library for %{name}
Group:          System Environment/Libraries
Requires:       %{name} = %{version}-%{release}

%description static
This package contains the static version of liblua for %{name}.

%files static
%defattr(-,root,root,-)
%{_libdir}/*.a

%clean
rm -rf %{buildroot} 

%changelog
```
