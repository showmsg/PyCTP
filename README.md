# PyCTP

**使用SWIG封装的上期CTP API**

参考：https://blog.csdn.net/pjjing/article/details/77338423

**编译环境：**

* **Windows**：Windows 10 1909 18363.720；
* **Python**：Python 3.7.6 (tags/v3.7.6:43364a7ae0, Dec 19 2019, 00:42:30) [MSC v.1916 64 bit (AMD64)]；
* **SWIG**：SWIG Version 4.0.1；
* **Visual Studio**：Microsoft Visual Studio Community 2019 16.5.2；

## 准备工作

 - 从CTP官网上下载CTP API点击下载。目前非穿透式版本如6.3.11已经不能使用了，现在用的穿透式版本为：v6.3.15_20190220 20:39:53，二代行情版本为：v6.3.15_P2_20190403 14:44:41，这两个都可以使用，64位的API文件包清单如下：
```
error.dtd
error.xml
ThostFtdcMdApi.h
ThostFtdcTraderApi.h
ThostFtdcUserApiDataType.h
ThostFtdcUserApiStruct.h
thostmduserapi_se.dll
thostmduserapi_se.lib
thosttraderapi_se.dll
thosttraderapi_se.lib
```
 - 安装Swig软件，本文中所用的Swig是swigwin-4.0.1版本，点击下载。
 - 安装python，如果使用32位Python那么对应的CTP版本也应该是32位的，本文使用的版本为Python 3.7.6 (tags/v3.7.6:43364a7ae0, Dec 19 2019, 00:42:30) [MSC v.1916 64 bit (AMD64)]
 - 安装Visual Studio，本文使用的版本为：Microsoft Visual Studio Community 2019 16.5.2；其他版本理论上来说也可以

## 通过Swig得到python接口文件
~~在刚刚下载得到的API文件夹内，新建文件PyCTP.i，内容如下：~~

请不要使用下面的代码，乱码的问题不好解决

```c++
%module(directors="1") PyCTP
 
%{
 
#include "ThostFtdcMdApi.h"
#include "ThostFtdcTraderApi.h"
%}
 
 
%feature("director") CThostFtdcMdSpi;
 
%feature("director") CThostFtdcTraderSpi;
 
%ignore THOST_FTDC_VTC_BankBankToFuture;
 
%ignore THOST_FTDC_VTC_BankFutureToBank;
 
%ignore THOST_FTDC_VTC_FutureBankToFuture;
 
%ignore THOST_FTDC_VTC_FutureFutureToBank;
 
%ignore THOST_FTDC_FTC_BankLaunchBankToBroker;
 
%ignore THOST_FTDC_FTC_BrokerLaunchBankToBroker;
 
%ignore THOST_FTDC_FTC_BankLaunchBrokerToBank;
 
%ignore THOST_FTDC_FTC_BrokerLaunchBrokerToBank;
 
 
%typemap(in) char *[] {
 
  /* Check if is a list */
  if (PyList_Check($input)) {
    int size = PyList_Size($input);
    int i = 0;
    $1 = (char **) malloc((size+1)*sizeof(char *));
    for (i = 0; i < size; i++) {
      PyObject *o = PyList_GetItem($input, i);
      if (PyString_Check(o)) {
        $1[i] = PyString_AsString(PyList_GetItem($input, i));
      } else {
        free($1);
        PyErr_SetString(PyExc_TypeError, "list must contain strings");
        SWIG_fail;
      }
    }
    $1[i] = 0;
  } else {
    PyErr_SetString(PyExc_TypeError, "not a list");
    SWIG_fail;
  }
}
 
// This cleans up the char ** array we malloc'd before the function call
%typemap(freearg) char ** {
 
  free((char *) $1);
}
 
%include "ThostFtdcUserApiDataType.h"
 
%include "ThostFtdcUserApiStruct.h"
 
%include "ThostFtdcMdApi.h"
 
%include "ThostFtdcTraderApi.h"
```
如果不把GBK转为UTF-8，会出现下面这样的乱码问题：

![此处输入图片的描述][1]

别看着这个字符串好像只需要解码什么的就可以得到源码，在python下处理这个特殊的字符串还是挺麻烦的，这个和普通的bytes str等的转换不同，所以我们最好使用下面的这版本代码进行封装就可以在C++底层自动把编码转为UTF-8

![此处输入图片的描述][2]

让CTP的GBK字符在返回的时候自动转化为UTF-8，那么使用下面的代码：
```c++
%module(directors="1") PyCTP
 
%{
 
#include "ThostFtdcMdApi.h"
#include "ThostFtdcTraderApi.h"
 
#include <codecvt>
#include <locale>
#include <vector>
#include <string>
using namespace std;
#ifdef _MSC_VER
const static locale g_loc("zh-CN");
#else    
const static locale g_loc("zh_CN.GB18030");
#endif
 
%}
 
 
%feature("director") CThostFtdcMdSpi;
 
%feature("director") CThostFtdcTraderSpi;
 
%ignore THOST_FTDC_VTC_BankBankToFuture;
 
%ignore THOST_FTDC_VTC_BankFutureToBank;
 
%ignore THOST_FTDC_VTC_FutureBankToFuture;
 
%ignore THOST_FTDC_VTC_FutureFutureToBank;
 
%ignore THOST_FTDC_FTC_BankLaunchBankToBroker;
 
%ignore THOST_FTDC_FTC_BrokerLaunchBankToBroker;
 
%ignore THOST_FTDC_FTC_BankLaunchBrokerToBank;
 
%ignore THOST_FTDC_FTC_BrokerLaunchBrokerToBank;
 
 
%typemap(out) char[ANY], char[] {
 
    const std::string &gb2312($1);
    std::vector<wchar_t> wstr(gb2312.size());
    wchar_t* wstrEnd = nullptr;
    const char* gbEnd = nullptr;
    mbstate_t state = {};
    int res = use_facet<codecvt<wchar_t, char, mbstate_t> >
        (g_loc).in(state,
            gb2312.data(), gb2312.data() + gb2312.size(), gbEnd,
            wstr.data(), wstr.data() + wstr.size(), wstrEnd);

    if (codecvt_base::ok == res)
    {
        wstring_convert<codecvt_utf8<wchar_t>> cutf8;
        std::string result = cutf8.to_bytes(wstring(wstr.data(), wstrEnd));
        resultobj = SWIG_FromCharPtrAndSize(result.c_str(), result.size());
    }
    else
    {
        std::string result;
        resultobj = SWIG_FromCharPtrAndSize(result.c_str(), result.size());
    }
}
 
%typemap(in) char *[] {
 
  /* Check if is a list */
  if (PyList_Check($input)) {
    int size = PyList_Size($input);
    int i = 0;
    $1 = (char **) malloc((size+1)*sizeof(char *));
    for (i = 0; i < size; i++) {
      PyObject *o = PyList_GetItem($input, i);
      if (PyString_Check(o)) {
        $1[i] = PyString_AsString(PyList_GetItem($input, i));
      } else {
        free($1);
        PyErr_SetString(PyExc_TypeError, "list must contain strings");
        SWIG_fail;
      }
    }
    $1[i] = 0;
  } else {
    PyErr_SetString(PyExc_TypeError, "not a list");
    SWIG_fail;
  }
}
 
// This cleans up the char ** array we malloc'd before the function call
%typemap(freearg) char ** {
 
  free((char *) $1);
}
 
%include "ThostFtdcUserApiDataType.h"
 
%include "ThostFtdcUserApiStruct.h"
 
%include "ThostFtdcMdApi.h"
 
%include "ThostFtdcTraderApi.h"
```
参考：https://blog.csdn.net/pjjing/article/details/103441333

说下这个接口文件大概的意思，具体的配置方法可以查看SWIG官方文档：

http://www.swig.org/Doc4.0/Python.html

http://www.swig.org/Doc4.0/SWIG.html#SWIG

    %module(directors="1") PyCTP
    
这句的意思是说，声明模组名称为PyCTP，directors="1" 表示生成的Python类可以像C++类一样被继承，相当于打开了这个功能开关。

    %{...%}

里的代码引入了所需的头文件和一些常量C++声明。

    %feature("director") CThostFtdcMdSpi;

声明让C++类CThostFtdcMdSpi开启director功能可以被Python继承。这里为啥不声明Api类是可继承的，我看了下大概是因为，Api类没有继承的必要，因为功能和调用都是提前订好的，且如果声明Api在封装的时候会再多两个警告，且Api类的生成方式有点特殊，不是直接__init__形式生成的，而是有一个类函数来创建，这就导致了如果在这声明继承Api在过后的使用中可能会导致不可预料的问题。

    %ignore ...;

忽略一些类或常量不封装，不知道为啥作者要忽略掉这些变量，不过这些变量也用不到。

    %typemap(out) char[ANY], char[]

从C++类型到Python类型的自定义转换，这里比如对编码进行自定义转化，对Python list参数进行识别直接使用的typemap函数，和最下面的一个清理内存的typemap函数。

    %include ...;

导入头文件

### 使用SWIG生成wrap文件
在当前文件夹下打开命令行，输入：

    swig -threads -c++ -python PyCTP.i

可能会出警告，我这是这样的：

```
λ swig -threads -c++ -python PyCTP.i
ThostFtdcMdApi.h(30) : Warning 514: Director base class CThostFtdcMdSpi has no virtual destructor.
ThostFtdcTraderApi.h(30) : Warning 514: Director base class CThostFtdcTraderSpi has no virtual destructor.

```

暂时忽略警告，之后我测试了下，暂时没有发现错误。

## 通过C++得到python可调用的pyd动态库
使用VS新建一个C++工程：
![此处输入图片的描述][3]

选择空项目：
![此处输入图片的描述][4]

项目名称填：**_PyCTP**
注意前面一定要有这个下划线，不能乱取，要和前面接口文件声明的一样
![此处输入图片的描述][5]

点击创建，然后把项目配置切换到Releas x64，切换到你要封装的环境上：
![此处输入图片的描述][6]
到：
![此处输入图片的描述][7]

向工程文件夹中添加需要编译的文件，我这边是把刚才SWIG生成的文件夹下的全部文件先复制到工程文件目录下，我的目录是这样的：
```
error.dtd
error.xml
ThostFtdcMdApi.h
ThostFtdcTraderApi.h
ThostFtdcUserApiDataType.h
ThostFtdcUserApiStruct.h
thostmduserapi_se.dll
thostmduserapi_se.lib
thosttraderapi_se.dll
thosttraderapi_se.lib
PyCTP.i
PyCTP.py
PyCTP_wrap.cxx
PyCTP_wrap.h
```
后面四个就是SWIG生成的源文件等，其实VS编译不需要全部的文件的，我全部复制只是为了方便。

然后 **右键左侧解决方案-添加-添加-现有项**，选择对应的头文件、源文件、资源文件添加到工程，我添加完毕后长这样：
![此处输入图片的描述][8]
然后还需要设置下面的几个参数：

**1）工程建64位dll类型**
![此处输入图片的描述][9]

**2）运行库选多线程(/MT)**
![此处输入图片的描述][10]

**3）将python目录下include文件夹的路径添加至C++附加包含目录**

我的设置在：**工程 – 属性 – 配置属性 – c/c++ – 常规 – 附加包含目录**
![此处输入图片的描述][11]

**4）将python目录下libs下的python37.lib添加至工程附加依赖项中**

我的设置在：工程 – 属性 – 配置属性 – 连接器 – 输入 – 附加依赖项
![此处输入图片的描述][12]

## 编译生成dll
点击菜单上的 **生成 – 生成解决方案F7**

可能会报一些类型警告，如果没有报错的话我们就编译好我们需要的dll文件了。

在_PyCTP\x64\Release下可以找到编译好的 **_PyCTP.dll**，把这个文件改名为：**_PyCTP.pyd**

然后复制SWIG生成的 **PyCTP.py** ，和CTP自带的 **thostmduserapi_se.dll** 与 **thosttraderapi_se.dll**

把这四个文件放到同一个目录，这四个文件就是我们封装好的CTP包了，直接

    import PyCTP

就可以使用CTP的类和一些常量了，DEMO懒得写了，你们去看原文，结合C++ Demo看看，调用和使用和C++ 版保持一致。

## 常见问题
1 invalid conversion from ‘const char**’ to ‘char**’ [-fpermissive]
解决方法：将i文件中，if (iconv(cd, (const char **)in, &inlen, &out, &outlen) != static_cast<size_t>(-1)) 中的(const char **)删掉

2 ‘strcpy’: This function or variable may be unsafe. Consider using strcpy_s instead.
解决方法：项目属性->配置属性->C/C+±>预处理器->预处理器定义中添加：_CRT_SECURE_NO_WARNINGS

3 fatal error LNK1112: 模块计算机类型“X86”与目标计算机类型“x64”冲突
解决方法：选择编译64位库时，vs中需要如下设置：解决方案属性->配置属性->将平台选为X64->配置管理器->选择X64平台 右键项目清理即可

4 对象或库文件“EDLib.lib”是使用比创建其他对象所用编译器旧的编译器创建的；请重新生成旧的对象和库
解决方法：项目属性->配置属性->高级->全程序优化改为无全程序优化

5 节数超过对象文件格式限制: 请使用 /bigobj 进行编译
解决方法：项目属性->配置属性->C/C+±>命令行->其它选项中键入/bigobj

6 无法解析的外部符号 __imp_sprintf_s
解决方法：项目属性->配置属性->链接器->附加依赖项中添加legacy_stdio_definitions.lib

其他问题请自己谷歌，一般是没有太大问题的。

**声明：仅是个人爱好编译，对此API引起的你的任何损失不负责任。**


  [1]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200408155104.png
  [2]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200409090143.png
  [3]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200407155924.png
  [4]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200407160049.png
  [5]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200407160145.png
  [6]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200407160450.png
  [7]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200407160548.png
  [8]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200407161145.png
  [9]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200407164447.png
  [10]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200407164558.png
  [11]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200407165819.png
  [12]: https://raw.githubusercontent.com/EVA-JianJun/GitPigBed/master/blog_files/img/SWIG_20200407170031.png