+++
date = "2016-06-18T08:36:15+08:00"
draft = false
title = "Java混淆/反混淆/加固 总结"
+++

## 混淆手段

### 定义超长类名
可以造成超长文件名, 造成常规反编译工具失败(路径太长).
例子: oooooooooooooo

### 找茬
OoOoOoooOOooo
__$$_$$__$_$$_

### 使用Java关键字
java对关键字检查操作在编译时, 所以在编译后修改没有问题.

### Unicode字符
...

### CJK(China Japan korea)字符
...

### 其他少见字符
...

### 盲文点字符
...

### 识别类名
通过源码中保留的Log.d(tag, "xx"); 通常程序员会将tag写为所在class名.

### 识别Enum
构建Enum的第二个参数是成员名.

### ACC_BRIDGE
标识的函数会调用另外一个, 两个函数同名.

### Getter/Setter
成员与函数同名

### 根据日志
....

### 字符串加密
...

### Asset加密
加密文件, 在打开文件函数处注入解密代码.  达到无缝加密. 分析并解密

### AndroidManifest.xml中namespace:name被清除.
该文件中同时包含ResourceId(Android特有)和namespace/name(标准xml特有)两套访问方式, 而Android中大部分使用resourceId, 该ResourceId对应了文本资源中的namaspace/name字符串。

### 函数合并
合并函数调用

### 利用Java函数重载
在java中, 函数以 "函数名+参数列表" 为sig(函数名相同, 参数不同可构成重载). 而在smali(bytecodes)中, 函数则以 "函数名+参数列表+返回类型" 为sig.
因此直接在编译后的smali(bytecodes)上动手, 将具有相同"函数列表"的函数改为同名. 这样在smali中可以正常执行, 但还原为java后, 会因为有多个相同"函数名+参数列表"而产生冲突.

## 反混淆手段
### 借助JEB简化混淆的命名
  ...

### 借助Proguard简化混淆的命名
如果混淆字符长期且复杂, 可以先用Proguard混淆一把, 简化复杂度.

通过Proguard keep class *(不混淆任何东西)来输出mapping表文件.(目的是为了修改mapping表中的对应名称来达到整体修改工程中名称的目的)

通过分析得到类的有意义名称, 然后将其修改至mapping文件, 最后利用Proguard来帮助批量修改回代码.

## 加固

### "dalvik"bug
  利用"dalvik"bug使其可以运行, 但不能够被反编译.

### dex超长数组
  定义超长数组可以使不少工具反编译失败(耗尽其内存), 实际运行中要在入口将其释放, 以免内存太大无法运行.

### elf清除或畸形化section
  清除或填充为畸形, 可使IDA和gikdbg.art等发生崩溃或无法正常解析. 可参见笔记: IDA - 打开畸形ELF时的报错信息含义

