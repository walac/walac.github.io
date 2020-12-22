---
title: 'NUIPreview: a Visual Studio extension to build Tizen.NUI graphical interfaces'
category: tizen
comments: true
tags: [tizen, C#]
---

As the last project of 2020, I and some other co-workers were assigned the
task of creating a prototype for a graphical user interface preview Visual
Studio extension for the
[NUI](https://docs.tizen.org/application/dotnet/guides/nui/overview/)
graphics toolkit. Notice the extension is for
[Visual Studio](https://visualstudio.microsoft.com/), not
[Visual Studio Code](https://code.visualstudio.com/).

The idea is the programmer create the user interface in a
[XAML](https://docs.microsoft.com/en-us/dotnet/desktop/wpf/fundamentals/xaml?view=netdesktop-5.0#:~:text=is under construction.-,What is XAML,NET Core app.)
file and the plugin displays the preview of the user interface. `NUI` already
has [some support](https://docs.tizen.org/application/dotnet/guides/nui/xaml/xaml-support-for-tizen-nui/)
for the `XAML` standard.

For parsing the `XAML` file, [@lauromoura](https://github.com/lauromoura)
found the [OmniXaml](https://github.com/OmniGUI/OmniXAML) library. Although not
maintained anymore, the library is pretty complete. Our initial idea was to make
the window show the result integrated with Visual Studio, but as NUI creates its
own Window and message loop, we had to take another approach. The basic
architecture is as follows: when the user click in the menu to show the preview,
we start a thread in which we create the NUI Window and run the message loop. We also
start a timer that periodically polls the active document; if it is a XAML file
and we can parse it successfully, we update the UI.

One particular detail is that Visual Studio is a 32 bits executable, so all the
[DALi](https://docs.tizen.org/application/native/guides/ui/dali/) libraries and
their dependencies must be compiled in 32 bits. As `DALi` is contantly evolving,
we created a branch in our own fork to make sure we can keep the plugin development
without being surprised by bugs introduced by new commits.

# Building DALi for 32 bits Windows

First, you have to make sure all [vcpkg](https://github.com/Microsoft/vcpkg) dependencies
are installed with their 32 bits variant.
That's the default in vcpkg last time I checked, but you can also add the `:x86-windows` suffix
for each package name.

That said, download our fork of the `DALi` libraries:

```sh
$ git clone -b vsix git://github.com/expertisesolutions/dali-core
$ git clone -b vsix git://github.com/expertisesolutions/dali-adaptor
$ git clone -b vsix git://github.com/expertisesolutions/dali-toolkit
$ git clone -b vsix git://github.com/expertisesolutions/dali-csharp-binder
$ git clone -b vsix git://github.com/expertisesolutions/tizenfx-stub
```

You also need the
[windows-dependencies](https://github.com/dalihub/windows-dependencies/)
repository. To build the libraries, you can use the following batch script:

```bat
set base_dir=c:\work
set common_args=-g "Visual Studio 16 2019" -A Win32 -DCMAKE_INSTALL_PREFIX=%base_dir%\dali-env \
-DCMAKE_TOOLCHAIN_FILE=%base_dir%\vcpkg\scripts\buildsystems\vcpkg.cmake -DCMAKE_BUILD_TYPE=Debug -DENABLE_DEBUG=ON -Wno-dev

REM window-dependencies
cd %base_dir%\windows-dependencies\build\
rd /Q /S build
mkdir build
cd build
cmake %common_args% .. || goto :error
cmake --build . --target install --config debug || goto :error

REM dali-core
cd %base_dir%\dali-core\build\tizen
rd /Q /S build-windows
mkdir build-windows
cd build-windows
cmake %common_args% -DENABLE_PKG_CONFIGURE=OFF -DENABLE_LINK_TEST=OFF -DINSTALL_CMAKE_MODULES=ON .. || goto :error
cmake --build . --target install --config debug || goto :error

REM dali-adaptor
cd %base_dir%\dali-adaptor\build\tizen
rd /Q /S build-windows
mkdir build-windows
cd build-windows
cmake %common_args% -DENABLE_PKG_CONFIGURE=OFF -DENABLE_LINK_TEST=OFF -DINSTALL_CMAKE_MODULES=ON -DPROFILE_LCASE=windows .. || goto :error
cmake --build . --target install --config debug || goto :error

REM dali-toolkit
cd %base_dir%\dali-toolkit\build\tizen
rd /Q /S build-windows
mkdir build-windows
cd build-windows
cmake %common_args% -DENABLE_PKG_CONFIGURE=OFF -DENABLE_LINK_TEST=OFF -DINSTALL_CMAKE_MODULES=ON -DUSE_DEFAULT_RESOURCE_DIR=OFF .. || goto :error
cmake --build . --target install -j1 --config debug || goto :error

REM dali-csharp-binder
cd %base_dir%\dali-csharp-binder\build\tizen
rd /Q /S build-windows
mkdir build-windows
cd build-windows
cmake %common_args% .. || goto :error
cmake --build . --target install --verbose --config debug || goto :error

tizenfx-stub
cd %base_dir%\tizenfx-stub
rd /Q /S build-windows
mkdir build-windows
cd build-windows
cmake %common_args% .. || goto :error
cmake --build . --target install --verbose --config debug || goto :error

:error
cd %base_dir%
```

Notice it assumes you have the vcpkg and the repositories under `C:\work`.
You can tweak the script for your own directory structure.

You also need to build [angle](https://opensource.google/projects/angle).
To build it you can follow the
[official documentation](https://chromium.googlesource.com/angle/angle/+/refs/heads/master/doc/DevSetup.md),
but before running the `autoninja` command to compile it, you have to
run `gn args <output directory>` and set the following variables:

```
angle_enable_d3d11 = false
target_cpu = "x86"
```

To make redistribution of the plugin easier, we ship the
[NUIPreview repository](https://github.com/expertisesolutions/NUIPreview)
with all
[DLL dependencies](https://github.com/expertisesolutions/NUIPreview/tree/master/NUIPreview/deps), so
you don't need to worry about compiling them.

# Running the plugin

Here is a demo of how the plugin behaves:

<iframe width="640" height="360" src="https://www.youtube.com/embed/67ccJSP7Vi8" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Of course, this is just a prototype. You check the source code in the
[repository](https://github.com/expertisesolutions/NUIPreview).
