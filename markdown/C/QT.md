### 1. 识别中文代码
```c++
// 识别中文代码
QString Class::knowChinese(const QByteArray &ba)
{
    QTextCodec::ConverterState state;
    QTextCodec *codec = QTextCodec::codecForName("UTF-8");
    QString text = codec->toUnicode( ba.constData(), ba.size(), &state);
    if (state.invalidChars > 0)
    {
        text = QTextCodec::codecForName("GBK")->toUnicode(ba);
    }
    else
    {
        text = ba;
    }

    return text;
}
```


### 2. QT5编译oci

#### 2.1 Qt Creator打开工程

D:\ProgramFiles\Qt\5.15.2\Src\qtbase\src\plugins\sqldrivers\oci\oci.pro，因为是用于Visual Stdio 2019，编译器选msvc2019_64。

#### 2.2 修改文件

```
// oci.pro
#QMAKE_USE += oci
QMAKE_LFLAGS += oci.lib

INCLUDEPATH += D:\ProgramFiles\Oracle\product\11.2.0\dbhome_1\OCI\include
LIBS += D:\ProgramFiles\Oracle\product\11.2.0\dbhome_1\OCI\lib\MSVC\oci.lib
LIBPATH += D:\ProgramFiles\Oracle\product\11.2.0\dbhome_1\OCI\lib\MSVC

// qsqldriverbase.pri
# For QMAKE_USE in the parent projects.
# include($$shadowed($$PWD)/qtsqldrivers-config.pri)

// qsql_oci.cpp，使用oracle 11会报错
OCIBindByPos2  ----->  OCIBindByPos
bindColumn.lengths,  ----->  (ub2*)bindColumn.lengths,
```

#### 2.3 编译

只编译不运行，会生成静态库和动态库，生成文件在D盘根目录，也可能在其它目录。

