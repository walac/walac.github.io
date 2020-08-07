---
title: 'Building DALi library for Windows'
category: dali
comments: true
tags: [windows, dali]
---

Since I left [Mozilla](https://mozilla.org) I came back to my old days as a
C/C++ developer. One of the projects I am working on is porting the
[Tizen.NUI](https://docs.tizen.org/application/dotnet/guides/nui/overview/) API
to Windows. [Tizen.NUI](https://docs.tizen.org/application/dotnet/guides/nui/overview/)
itself is written in C#, so it doesn't require any major effort to run on Windows,
but it is heavily based on
[DALi](ttps://docs.tizen.org/application/native/guides/ui/dali/), and there
where the job lies. [DALi](ttps://docs.tizen.org/application/native/guides/ui/dali/)
kind of works on Windows already, but I caught several bugs affecting 64 bits. You
can refer to the pull requests to get details on what was wrong.

[DALi](ttps://docs.tizen.org/application/native/guides/ui/dali/),
doesn't belong to a single repo, actually. On top of
[dali-core](https://github.com/dalihub/dali-core/) we have
[dali-adaptor](https://github.com/dalihub/dali-adaptor) and
[dali-toolkit](https://github.com/dalihub/dali-toolkit), not to mention the sample
[applications](https://github.com/dalihub/dali-demos/). This post is a step by step
guide to build [DALi](https://docs.tizen.org/application/native/guides/ui/dali/)
on Windows.

Dependencies
============

To build and install [DALi](ttps://docs.tizen.org/application/native/guides/ui/dali/)
you will need:

* [Visual Studio 2019](https://visualstudio.microsoft.com/vs/community/) (Community Edition is fine)
* [git](https://gitforwindows.org/)
* [vcpkg](https://github.com/Microsoft/vcpkg/)

Building Windows dependencies
=============================

After the dependencies installed, you need the
[windows-dependencies](https://github.com/dalihub/windows-dependencies) repository.
Run the following commands in a subfolder that will host the DALi repos:

```
C:\DALi>git clone https://github.com/dalihub/windows-dependencies
C:\DALi>set VCPKG_FOLDER=C:\DALi
C:\DALi>set DALI_ENV_FOLDER=C:\DALi\dali-env
C:\DALi>cd windows-dependencies\build
C:\DALi\windows-dependencies\build>mkdir build
C:\DALi\windows-dependencies\build>cd build
C:\DALi\windows-dependencies\build\build>cmake -g Ninja . -DCMAKE_TOOLCHAIN_FILE=%VCPKG_FOLDER%/vcpkg/scripts/buildsystems/vcpkg.cmake -DCMAKE_INSTALL_PREFIX=%DALI_ENV_FOLDER% ..
C:\DALi\windows-dependencies\build\build>cmake --build . --target install
```

`DALI_ENV_FOLDER` is the output directory where the artifacts generated are installed.

Building the DALi libraries
===========================

Building the DALi libraries ([dali-core](https://github.com/dalihub/dali-core/),
[dali-adaptor](https://github.com/dalihub/dali-adaptor/) and
[dali-toolkit](https://github.com/dalihub/dali-toolkit/)) is straightforward and
the `README` of each library describes the steps in more detail. Here
I am just giving you the sequence of commands to build them. If you want to know
more details, you can refer to each library documentation.

dali-core
---------

```
C:\DALi>git clone https://github.com/dalihub/dali-core/
C:\DALi>cd dali-core\build\tizen
C:\DALi\dali-core\build\tizen>mkdir build
C:\DALi\dali-core\build\tizen>cd build
C:\DALi\dali-core\build\tizen\build>cmake -g Ninja . -DCMAKE_TOOLCHAIN_FILE=%VCPKG_FOLDER%/vcpkg/scripts/buildsystems/vcpkg.cmake -DENABLE_PKG_CONFIGURE=OFF -DENABLE_LINK_TEST=OFF -DCMAKE_INSTALL_PREFIX=%DALI_ENV_FOLDER% -DINSTALL_CMAKE_MODULES=ON -Wno-dev ..
C:\DALi\dali-core\build\tizen\build>cmake --build . --target install
```

dali-adaptor
------------

```
C:\DALi>%VCPKG_FOLDER%\vcpkg\vcpkg.exe install pthreads:x64-windows
C:\DALi>git clone https://github.com/dalihub/dali-adaptor/
C:\DALi>cd dali-adaptor\build\tizen
C:\DALi\dali-adaptor\build\tizen>mkdir build
C:\DALi\dali-adaptor\build\tizen>cd build
C:\DALi\dali-adaptor\build\tizen\build>cmake -g Ninja . -DCMAKE_TOOLCHAIN_FILE=%VCPKG_FOLDER%/vcpkg/scripts/buildsystems/vcpkg.cmake -DENABLE_PKG_CONFIGURE=OFF -DENABLE_LINK_TEST=OFF -DCMAKE_INSTALL_PREFIX=%DALI_ENV_FOLDER% -DINSTALL_CMAKE_MODULES=ON -DPROFILE_LCASE=windows -Wno-dev ..
C:\DALi\dali-adaptor\build\tizen\build>cmake --build . --target install
```

dali-toolkit
------------

```
C:\DALi>git clone https://github.com/dalihub/dali-toolkit/
C:\DALi>cd dali-toolkit\build\tizen
C:\DALi\dali-toolkit\build\tizen>mkdir build
C:\DALi\dali-toolkit\build\tizen>cd build
C:\DALi\dali-toolkit\build\tizen\build>cmake -g Ninja . -DCMAKE_TOOLCHAIN_FILE=%VCPKG_FOLDER%/vcpkg/scripts/buildsystems/vcpkg.cmake -DENABLE_PKG_CONFIGURE=OFF -DENABLE_LINK_TEST=OFF -DCMAKE_INSTALL_PREFIX=%DALI_ENV_FOLDER% -DINSTALL_CMAKE_MODULES=ON -DUSE_DEFAULT_RESOURCE_DIR=ON -Wno-dev ..
C:\DALi\dali-toolkit\build\tizen\build>cmake --build . --target install
```

dali-demo
---------

[dali-demo](https://github.com/dalihub/dali-demo/) contains a series of sample applications
that you can try to test if your installation is ok. The procedure to build and install
these apps are the same as for the libraries:

```
C:\DALi>git clone https://github.com/dalihub/dali-demo/
C:\DALi>cd dali-demo\build\tizen
C:\DALi\dali-demo\build\tizen>mkdir build
C:\DALi\dali-demo\build\tizen>cd build
C:\DALi\dali-demo\build\tizen>cmake -g Ninja . -DCMAKE_TOOLCHAIN_FILE=%VCPKG_FOLDER%/vcpkg/scripts/buildsystems/vcpkg.cmake -DENABLE_PKG_CONFIGURE=OFF -DINTERNATIONALIZATION=OFF -DCMAKE_INSTALL_PREFIX=%DALI_ENV_FOLDER% -Wno-dev ..
C:\DALi\dali-demo\build\tizen\build>cmake --build . --target install
```

Running
=======

If everything was ok, your binaries should reside inside inside the
`DALI_ENV_FOLDER` directory.

[DALi](ttps://docs.tizen.org/application/native/guides/ui/dali/)
depends on [libANGLE](https://chromium.googlesource.com/angle/angle),
an OpenGL ES library that translates OpenGL calls to one of several supported
backends. On Windows, the default backend is Direct3D 11. I was having
problems with the D3D11 backend on my machine when trying to run the `DALi`
sample applications. What I did was build `libANGLE` disabling the D3D11
backend, falling back to Direct X 9. The
[build instructions](https://chromium.googlesource.com/angle/angle/+/refs/heads/master/doc/DevSetup.md)
tell how to disable the D3D11 backend.

With all set, you can run a sample application:

```
C:\DALi\dali-env\bin>hello-world.example.exe
```

On the day I write this post, I have some patches merged in the
[DALi](ttps://docs.tizen.org/application/native/guides/ui/dali/)
repos and others are pending to review. If you are experiencing crashes
you may want to give a look at the open pull requests and apply
some of them.
