---
title: "rpmbuild and PyInstaller"
date:  2015-08-19 20:12:00
description: "Avoiding “Cannot open self” errors when using PyInstaller binaries packaged with rpmbuild"
keywords: rpmbuild, PyInstaller, cannot open self, debug, rpm
---

I recently ran into an issue using `rpmbuild` to package PyInstaller binaries. Running a packaged executable yielded the following:

~~~
Cannot open self /path/to/executable or /path/to/executable.pkg
~~~

While this particular output could be caused by a few things, after using `debug=True` in the [PyInstaller spec file](http://pythonhosted.org/PyInstaller/#using-spec-files) to make a debug binary, I tracked the error down to the [pyi_arch_check_cookie](https://github.com/pyinstaller/pyinstaller/blob/v2.1/bootloader/common/pyi_archive.c#L174) function.

What's happening here is that PyInstaller is doing a consistency check of sorts on the package internals. I didn't make the connection to `rpmbuild`, which made the troubleshooting session a little longer than it needed to be.

As part of the default RPM building process any binary files are stripped of [debug symbols](https://en.wikipedia.org/wiki/Debug_symbol). This is to say that any variable or function names that remain in binaries are removed and placed into a separate package. The result is smaller binaries with the option of loading the symbols at a later time for troubleshooting. While this would be desirable for a typical package, it breaks the integrity checks that PyInstaller is performing.

The solution is to disable stripping for the rpmbuild operation, by adding some definitions to the rpm spec file.[^1][^2]

~~~
%define __os_install_post %{nil}
%define debug_package %{nil}
~~~

Rebuild your package and give it another go.

[^1]: The second line may not be strictly necessary. It prevents the creation of a [debuginfo package](https://fedoraproject.org/wiki/Packaging:Debuginfo).

[^2]: I could probably also have [used %global here](http://www.rpm.org/wiki/PackagerDocs/Macros#BuiltinMacros) instead of define.
