# scripy安装提示错误：Microsoft Visual C++ 14.0 is required
出现这个问题时，如果你按照提示去下载Microsoft Visual C++，也是解决不了的，因为是版本不匹配问题。主要解决方式有2种：
- 下载对应python版本的twisted即可。（https://www.lfd.uci.edu/~gohlke/pythonlibs/#twisted）
- 下载visualcppbuildtools_full.exe编译工具包（vc_redist.x86.exe之类的都不行）


