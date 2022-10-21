+++
title = "Wrangling CRT libraries on Windows"
date = 2022-10-21
+++

Sometimes when working on my [toybox](/@/projects/toybox.md) game engine project I find myself needing to contribute to upstream. In this case I wound up implementing mingw support for [KTX-Software](https://github.com/KhronosGroup/KTX-Software). It was a bit of a hassle but it was necessary to get my whole engine compiling with `gcc` on Windows.

I had hoped that the work had been done but lo and behold it never is. Turns out with the latest version of `gcc` (12.2.0) came build breakages in the KTX-Software CI. Of course I was pinged about this since I was the one who cared. If you want to see a record of my descent into madness trying to fix this I can direct you [to the github issue](https://github.com/KhronosGroup/KTX-Software/issues/641).

The source of the issue was a failure to compile this workaround for a bug in `msvcrt`, the older C runtime library for Windows:
```
     // Need to flush so that fstat will return the current size. 
     // Can ignore return value. The only error that can happen is to tell you 
     // it was a NOP because the file is read only. 
 #if (defined(_MSC_VER) && _MSC_VER < 1900) || defined(__MINGW32__) 
     // Bug in VS2013 msvcrt. fflush on FILE open for READ changes file offset 
     // to 4096. 
     if (str->data.file->_flag & _IOWRT) 
 #endif 
```

The build error is specifically that `_flag` doesn't exist and `_IOWRT` is not defined. When we look at the mingw headers to see where `_IOWRT` is defined you'll see that it's only defined if `_UCRT` is defined. Which was the first clue that we were suddenly using the `ucrt`, Microsoft's newer "Universal" C runtime library.

A good tool for wrangling this was `MSYS2` as it allows you to install a mingw-w64 toolchain for both the older `msvcrt` (mingw-w64-x86_64-toolchain) and for the newer `ucrt` (mingw-w64-x86_64-ucrt-toolchain). Compiling with a `gcc` from the older toolchain worked just fine, it was only when building with the newer toolchain would this fail to compile.

So just skip the `msvcrt` runtime workaround when `_UCRT` is defined. Good enough, right? Not quite, the tests fail when built with `ucrt`. KTX-Software builds for a lot of platforms so it would be strange if there was a bug specifically in mingw-w64 just when building against the `ucrt`. It's possible but `ucrt` has been the default for `msvc` for a while (since VS2013) and the Visual Studio based builds run their tests just fine. So we're probably just missing a runtime dependency.

Sure enough when using a tool like [Dependencies](https://github.com/lucasg/Dependencies) to examine the DLLs required by the test executables it was clear that we were missing `libstdc++-6.dll`. Copying this dll from my mingw-w64 toolchain next to the test executables was all it took for the tests to pass. 

So how do we fix this without copying dlls around, manually or even automatically. I really don't want to have to write a custom CMake target to copy a dll around. How are downstream dependencies supposed to know they need that dll unless it comes in from a CMake target; sounds like a hard problem to solve. Instead I found [this Stack Overflow question](https://stackoverflow.com/questions/6404636/libstdc-6-dll-not-found) in my searches and sure enough making sure that the test executables were linked with `-static-libgcc` and `-static-libstdc++` was all it took to gete a working build. 

You can find the whole fix [over in this PR](https://github.com/KhronosGroup/KTX-Software/pull/642).