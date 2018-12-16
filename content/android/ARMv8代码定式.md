+++
date = "2017-10-23T16:42:28+08:00"
draft = false
title = "ARMv8代码定式"
+++


本文案例默认编译自`android-ndk-r10d`, 并且编译选项的均为:

    APP_OPTIM := release  
    OPT_CFLAGS := -O2"

若为其它ndk环境将会在小节开头处有特殊说明.

# 一. 成员函数
## a. 参数传递


# 二. 全局变量引用
```
const char g_str0[] = "hello world 0~";
const char g_str1[] = "hello world 1~";
const char g_str2[] = "hello world 2~";
int main(int argc, char *argv[])
{
    printf(g_str0);
    printf(g_str1);
    printf(g_str2);
    return 0;
}
```
**objdump:**
```
adrp    x19, 11000      ; 这里的0x11000指向的是.data数据区
add     x19, x19, #0x10 ; .data+0x10, 定位到"hello world 0~"
mov     x0, x19
bl      590 <printf@plt>
add     x0, x19, #0x10  ; .data+0x20, 定位到"hello world 1~"
bl      590 <printf@plt>
add     x0, x19, #0x20  ; .data+0x30, 定位到"hello world 2~"
bl      590 <printf@plt>

```
**总结**

如果变量之间有相邻关系, 编译器会定位首个变量位置, 然后进行叠加定位.

上面的三个string长度为0xF,但3次累加偏移均为0x10. 原因是编译器布置数据的ALGIN值为0x10, 有利于更快访问内存.

# 三. 局部静态变量引用

下面的代码修改自`全局变量引用`,修改的部分只是将全局变量移入函数体并加入static修饰符
```
int main(int argc, char *argv[])
{
    static char g_str0[] = "hello world 0~";
    static char g_str1[] = "hello world 1~";
    static char g_str2[] = "hello world 2~";
    printf(g_str0);
    printf(g_str1);
    printf(g_str2);
    return 0;
}
```
**objdump:**
```
; 编译后的结果与`全局变量引用`中的汇编完全相同.
adrp    x19, 11000
add     x19, x19, #0x10
mov     x0, x19
bl      590 <printf@plt>
add     x0, x19, #0x10
bl      590 <printf@plt>
add     x0, x19, #0x20
bl      590 <printf@plt>

```
**总结**

局部静态变量与全局变量无本质区别, 二者数据都存放于.data段.


# 四. 常数合并
表达式中存在多个常量的情况
```
int n = 1 + 1;
```
**objdump:**
```
mov    w0, #0x2         ; 编译期间已得出结果.
```
**总结**

无论release(O2)还是debug对于简单表达式均会在编译期间计算完成.

# 五. 乘法
## a. 2的幂
```
int a = (int)argc * 4;
```
**ida:**
```
UBFM    W1, W0, #0x1E, #0x1D
```
**objdump:**
```
LSL    W1, W0, #2
```
**总结**

由于是乘2的幂, 所以可理解为左移. 左移不需要考虑符号位, 所以使用UBFM系列. ida与objdump解析差异请详见: `xBFM的别名转换与识别`小节.

## b. 非2的幂
### a). 乘数+1后为2的幂
```
int a = (int)argc * 15;
```
**ida:**
```
; W0 为 argc 
UBFM    W1, W0, #0x1C, #0x1B
SUB     W1, W1, W0

```
根据 `xBFM的别名转换与识别`可转换为:
```
LSL    W1, W0, #4    ;  a = argc * 16
SUB    W1, W1, W0   ;  a = a - argc
```
### b). 乘数-1后为2的幂
```
int a = (int)argc * 17;
```
**ida:**
```
ADD    W1, W0, W0,LSL#4 ; a = argc*16 + argc
```

### c). 乘数/2-1为2的幂
```
int a = (int)argc * 10;  
```
**ida:**
```
; W0 为 argc 
ADD     W1, W0, W0,LSL#2
UBFM    W1, W1, #0x1F, #0x1E
```
转换为:
```
ADD    W1, W0, W0,LSL#2  ; a = argc * 5 
LSL    W1, W1, #1        ; a = a * 2
```
### d). `乘数 = (2^x+1) * 2^(y-x)`  或  `乘数 = (2^x+1) * 2^(y-x) + 1`

    满足以下条件, m为乘数, y值为满足"大于并最接近m的 2的n次幂数"的n: 
    x的值为满足以下等式的值, 需要遍历尝试, 遍历范围为0到y:


**m=(2x+1)∗2(y−x)**

**或**

**m=(2x+1)∗2(y−x)+1**

```
int a = (int)argc * 1041; 
```
**ida解析:**
```
; W0 = argc = m = 1041
; x = 10 = 6+4
ADD    W1, W0, W0,LSL#6  ; 65 = (2^6 + 1)
ADD    W1, W0, W1,LSL#4  ; 1041= 65 * 2^4 + 1
```
## GCC优化乘法总结
该节的乘法优化针对变量*常量的情况.  因为`常量*常量`会在编译器计算, `变量*变量`会直接使用MUL指令.

GCC针对乘法优化方式的采用优先级为: 

`2的n次幂` > `乘数-1=2^n`/`乘数+1=2^n` > `乘数=(2^x+1)*(2^(y-x))`/`乘数=(2^x+1)*(2^(y-x))+1` > `不适合所有优化方式,使用MUL指令`

# 六. 除法
## a. 2的幂
```
int a = (int)argc / 4;
```
**ida解析:**
```
SBFM    W1, W2, #2, #0x1F
```
**objdump解析:**
```
asr     w1, w2, #2
```
**总结**

由于除数是2的幂, 所以可理解为右移. 若被除数是有符号数需要考虑符号位, 所以使用SBFM. 详见`xBFM的别名转换与识别`小节.

# 七. xBFM的别名转换与识别
Unsigned Bitfield Move / Signed Bitfield Move

32bit: `XBFM <Wd>, <Wn>, #<immr>, #<imms>`

64bit: `XBFM <Xd>, <Xn>, #<immr>, #<imms>`

    举例来说, 在arm指令手册中, LSL、LSR、UBFIZ、UBFX、UXTB、UXTH都属于UBFM的别名, 也就是说指令编码是完全相同的. 具体解析方式根据反汇编工具的具体策略而视. 如上:《IDA》中直接将所有情况解析为能概括所有情况的UBFM. 但是在《objdump》中会根据情况将其解析为最符合使用意图的别名. LSL或LSR实际上属于UBFM的子集. 换言之LSL或LSR可以被UBFM概括.


## a. LSL、LSR/ASR
识别特征(这里直接通过操作数的特征来转为易读的汇编形式(属于土方子), 正规的方式在指令手册有详细描述): 
```
if (is_word) // is Wx
{
  if (immr-imms == 1){
    type = TP_LSL;
    printf ("LSL %s, %s, #%d", $Wd, $Wn, 32-$immr);
  }else if (immr <= 0x1F && imms == 0x1F){
    type =  (is_SBFM? TP_ASR : TP_LSR);  // LSR or ASR
    printf ("%cSR %s, %s, #%d", (is_SBFM?'A':'L'), $Wd, $wn, $immr);
  }else{
    type = TP_OTHER;
  }
}
else if (is_Xword)
{
  // ...
  if ((imms == 0x1F && immr > 0x1F) || immr-imms == 1){
    /*
      LSL (<=32bit number) 左移位数小于等于32, 产生特征为: (imms == 0x1F && immr > 0x1F)
      LSL (>32bit number)  左移位数大于32, 产生特诊为: (immr-imms == 1)
     */
    type = TP_LSL;
    printf("LSL %s, %s, #%d", $Xd, $Xn, 64-$immr);
  }else if (immr <= 0x3F && imms == 0x3F ){
    type = (is_SBFM? TP_ASR : TP_LSR);
    printf("%cSR %s, %s, $%d", (is_SBFM?'A':'L'), $Xd, $Xn, $immr);
  }else{
    type = TP_OTHER;
  }
}
```
举例1, 有如下IDA汇编:
```
UBFM    W1, W0, #0x1E, #0x1D
```
可利用转化规则`LSL  W1, W0, #($32-immr)`, 得到如下LSL结果:
```
LSL    W1, W0, #2
```
举例2, 有如下IDA汇编:
```
SBFM    W1, W0, #2, #0x1F
```
可以利用转化规则`ASR  W1, W0, $immr` (因为是Signed Bitfield Move, 所以转为ASR), 得到如下ASR结果:
```
ASR    W1, W0, #2
```
举例3, 有如下IDA汇编:
```
UBFM    X1, X0, #0x21, #0x20
```
可利用转化规则`LSL  X1, X0, #($64-immr)`, 得到如下LSL结果:
```
LSL    X1, X0, #31
```
举例4, 有如下IDA汇编:
```
SBFM    X1, X0, #0x32, #0x3F
```
可利用转化规则`ASL  X1, X0, $immr`, 得到如下ASR (因为是SBFM, 所以采用ASR)结果:
```
ASR    X1, X0, #50
```

# 八.类
本节以如下代码为例展开分析：
```
#include <jni.h>
#include <string>
#include <stdio.h>
#include <android/log.h>

class SuperA{
public:
   int value_a;

   SuperA(){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperA::SuperA()");
   }
   virtual ~SuperA(){
       this->value_a = 0;
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperA::~SuperA()");
   }
   virtual int fun1
           (int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperA::fun1()");
       return a + 1;
   }
   virtual int fun2(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperA::fun2()");
       this->value_a = a + 1;
       return a + 1;
   }
   virtual int testA(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperA::testA()");
       this->value_a = a + 2;
       return a + 2;
   }
};

class SuperB{
public:
   int value_b;

   SuperB(){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperB::SuperB()");
   }
   virtual ~SuperB(){
       this->value_b = 0;
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperB::~SuperB()");
   }
   virtual int fun1(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperB::fun1()");
       return a + 1;
   }
   virtual int fun2(int a){
        __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperB::fun2()");
       return a + 2;
   }
   virtual int testB1(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperB::testB1()");
       return a + 1;
   }
   virtual int testB2(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperB::testB2()");
       this->value_b = a+1;
       return a + 1;
   }
};

class SuperC{
public:
   int value_c;

   SuperC(){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperC::SuperC()");
   }
   virtual ~SuperC(){
       this->value_c = 0;
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperC::~SuperC()");
   }
   virtual int fun1(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperC::fun1()");
       return a + 1;
   }
   virtual int fun2(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperC::fun2()");
       return a + 2;
   }
   virtual int testC1(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperC::testC1()");
       this->value_c = a+1;
       return a + 1;
   }
   virtual int testC2(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "SuperC::testC2()");
       this->testC3(123);
       return a + 1;
   }
   virtual int testC3(int a) = 0;
};


class Sub: public SuperA, public SuperB, public SuperC{
public:
   int value_sub;

   Sub(){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "Sub::Sub()");
   }
   virtual ~Sub(){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "Sub::~Sub()");
   }
   virtual int fun1(int a){
        __android_log_print(ANDROID_LOG_DEBUG, "c++", "Sub::fun1()");
       return a + 1 + 1;
   }
   virtual int fun2(int a){
        __android_log_print(ANDROID_LOG_DEBUG, "c++", "Sub::fun2()");
       this->value_sub = a + 2 + 2;
       return a + 2 + 2;
   }
   virtual int fun3(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "Sub::fun3()");
       SuperC::testC1(1);
       return a + 3;
   }
   virtual int fun4(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "Sub::fun4()");
       return a + 23;
   }
   virtual int testC3(int a){
       __android_log_print(ANDROID_LOG_DEBUG, "c++", "Sub::testC3()");
       return a + 23;
   }
};

extern "C"
JNIEXPORT jstring JNICALL
Java_cplusplus_1test_example_com_myapplication_MainActivity_stringFromJNI(
       JNIEnv *env,
       jobject /* this */) {
   std::string hello = "Hello from C++";

   Sub *sub = new Sub();
   sub->fun1((int)env->GetVersion());
   sub->fun2((int)env->GetVersion());
   sub->fun3((int)env->GetVersion());
   sub->fun4((int)env->GetVersion());
   sub->testA((int)env->GetVersion());
   sub->testB1((int)env->GetVersion());
   sub->testC1((int)env->GetVersion());

   delete sub;

   return env->NewStringUTF(hello.c_str());
}
```
**以上代码中4个class的关系如下所示，`SuperA,..B,..C`被`Sub`继承, 并且所有class均有`虚析构`:**

![img](1.png "img")


## a. 构造函数
从以下汇编代码来看，这个构造函数大概做了两件事：

1. 首先调用3个父类的构造函数构造好父类空间.

2. 将3个父类的虚表替换为Sub自身的虚表.

```
.text:0000000000008D14 ; __int64 __fastcall Sub::Sub(Sub *__hidden this)
.text:0000000000008D14                 WEAK _ZN3SubC2Ev
.text:0000000000008D14 _ZN3SubC2Ev                             ; CODE XREF: Sub::Sub(void)+Cj
.text:0000000000008D14                                         ; DATA XREF: .got:_ZN3SubC2Ev_ptro
.text:0000000000008D14
.text:0000000000008D14 var_20          = -0x20
.text:0000000000008D14 var_10          = -0x10
.text:0000000000008D14 var_s0          =  0
.text:0000000000008D14
.text:0000000000008D14                 STP             X22, X21, [SP,#-0x10+var_20]!
.text:0000000000008D18                 STP             X20, X19, [SP,#0x20+var_10]
.text:0000000000008D1C                 STP             X29, X30, [SP,#0x20+var_s0]
.text:0000000000008D20                 ADD             X29, SP, #0x20
# ------------------------------------------------- inline SuperA::SuperA() - start ---------------------------------------------
.text:0000000000008D24                 ADRP            X21, #_ZTV6SuperA_ptr@PAGE
.text:0000000000008D28                 LDR             X21, [X21,#_ZTV6SuperA_ptr@PAGEOFF]
.text:0000000000008D2C                 ADRP            X20, #aC@PAGE ; "c++"
.text:0000000000008D30                 ADD             X20, X20, #aC@PAGEOFF ; "c++"
.text:0000000000008D34                 ADRP            X2, #aSuperaSupera@PAGE ; "SuperA::SuperA()"
.text:0000000000008D38                 MOV             X19, X0
.text:0000000000008D3C                 ADD             X8, X21, #0x10    # +0x10是因为要跳过虚表头部的0x8(全0)+0x8(typeinfo ptr)
.text:0000000000008D40                 ADD             X2, X2, #aSuperaSupera@PAGEOFF ; "SuperA::SuperA()"
.text:0000000000008D44                 MOV             W0, #3
.text:0000000000008D48                 MOV             X1, X20
.text:0000000000008D4C                 STR             X8, [X19]    # 1. 在sub obj内部offset:0x00的位置作为SuperA的对象区域
.text:0000000000008D50                 BL              .__android_log_print
# ------------------------------------------------- inline SuperA::SuperA() - end ---------------------------------------------
# ------------------------------------------------- inline SuperB::SuperB() - start ---------------------------------------------
.text:0000000000008D54                 ADRP            X22, #_ZTV6SuperB_ptr@PAGE
.text:0000000000008D58                 LDR             X22, [X22,#_ZTV6SuperB_ptr@PAGEOFF]
.text:0000000000008D5C                 ADD             X8, X22, #0x10
.text:0000000000008D60                 STR             X8, [X19,#0x10]    # 2. 在sub obj内部offset:0x10的位置作为SuperA的对象区域
.text:0000000000008D64                 ADRP            X2, #aSuperbSuperb@PAGE ; "SuperB::SuperB()"
.text:0000000000008D68                 ADD             X2, X2, #aSuperbSuperb@PAGEOFF ; "SuperB::SuperB()"
.text:0000000000008D6C                 MOV             W0, #3
.text:0000000000008D70                 MOV             X1, X20
.text:0000000000008D74                 BL              .__android_log_print
# ------------------------------------------------- inline SuperB::SuperB() - end ---------------------------------------------
# ------------------------------------------------- inline SuperC::SuperC() - start ---------------------------------------------
.text:0000000000008D78                 ADRP            X20, #_ZTV6SuperC_ptr@PAGE
.text:0000000000008D7C                 LDR             X20, [X20,#_ZTV6SuperC_ptr@PAGEOFF]
.text:0000000000008D80                 ADD             X8, X20, #0x10
.text:0000000000008D84                 STR             X8, [X19,#0x20]    # 3. 在sub obj内部offset:0x20的位置作为SuperA的对象区域
.text:0000000000008D88                 ADRP            X1, #aC@PAGE ; "c++"
.text:0000000000008D8C                 ADRP            X2, #aSupercSuperc@PAGE ; "SuperC::SuperC()"
.text:0000000000008D90                 ADD             X1, X1, #aC@PAGEOFF ; "c++"
.text:0000000000008D94                 ADD             X2, X2, #aSupercSuperc@PAGEOFF ; "SuperC::SuperC()"
.text:0000000000008D98                 MOV             W0, #3
.text:0000000000008D9C                 BL              .__android_log_print
# ------------------------------------------------- inline SuperC::SuperC() - end ---------------------------------------------
.text:0000000000008DA0                 ADRP            X8, #_ZTV3Sub_ptr@PAGE
.text:0000000000008DA4                 LDR             X8, [X8,#_ZTV3Sub_ptr@PAGEOFF]
.text:0000000000008DA8                 ADD             X9, X8, #0x10    # 1. 取出Sub的第1块虚表， 用于覆盖SuperA.
.text:0000000000008DAC                 ADD             X10, X8, #0x60    # 2. 取出Sub的第2块虚表， 用于覆盖SuperB.
.text:0000000000008DB0                 ADD             X8, X8, #0xA0    # 3. 取出Sub的第3块虚表， 用于覆盖SuperC.
.text:0000000000008DB4                 STR             X9, [X19]    # 1. 用Sub的第1块虚表, 覆盖掉SuperA的虚表.
.text:0000000000008DB8                 STR             X10, [X19,#0x10]   # 2. 用Sub的第2块虚表, 覆盖掉SuperB的虚表.
.text:0000000000008DBC                 STR             X8, [X19,#0x20]    # 3. 用Sub的第3块虚表, 覆盖掉SuperC的虚表.
.text:0000000000008DC0                 ADRP            X1, #aC@PAGE ; "c++"
.text:0000000000008DC4                 ADRP            X2, #aSubSub@PAGE ; "Sub::Sub()"
.text:0000000000008DC8                 ADD             X1, X1, #aC@PAGEOFF ; "c++"
.text:0000000000008DCC                 ADD             X2, X2, #aSubSub@PAGEOFF ; "Sub::Sub()"
.text:0000000000008DD0                 MOV             W0, #3
.text:0000000000008DD4                 BL              .__android_log_print
.text:0000000000008DD8                 LDP             X29, X30, [SP,#0x20+var_s0]
.text:0000000000008DDC                 LDP             X20, X19, [SP,#0x20+var_10]
.text:0000000000008DE0                 LDP             X22, X21, [SP+0x20+var_20],#0x30
.text:0000000000008DE4                 RET
.text:0000000000008DE4 ; End of function Sub::Sub(void)
```
以上三个父类的大小之所以为0x10，是因为:0x8(VT) + 0x4(this->value_*) + 0x4(align), 最后class Sub的内存布局为：

## b.内存布局

![img](2.png "img")

根据以上图片得出，内存的分布为先父类后子类。以上的`Sub::value_sub`被分配到了最尾部.

## c.虚表
以上代码中共有4个类，但一共只有三个虚表。这是因为子类Sub的虚表与首个父类SuperA进行了合并。以下是三个虚表的分布和指向情况：
```
# ============================== [off: 0x00] SuperA & Sub =================================
.data.rel.ro:000000000003EF40 off_3EF40       DCQ _ZN3SubD2Ev         ; Sub::~Sub()    # 虚析构函数1
.data.rel.ro:000000000003EF48                 DCQ _ZN3SubD0Ev         ; Sub::~Sub()    # 虚析构函数2 (该析构比`析构函数1`多了释放内存的操作，专供delete释放动态new出来的对象)
.data.rel.ro:000000000003EF50                 DCQ _ZN3Sub4fun1Ei      ; Sub::fun1(int)    # 这里指向Sub::fun1(int)
.data.rel.ro:000000000003EF58                 DCQ _ZN3Sub4fun2Ei      ; Sub::fun2(int)    # 这里指向Sub::fun2(int)
.data.rel.ro:000000000003EF60                 DCQ _ZN6SuperA5testAEi  ; SuperA::testA(int)    # 这里指向SuperA::testA(int)
.data.rel.ro:000000000003EF68                 DCQ _ZN3Sub4fun3Ei      ; Sub::fun3(int)    # 这里指向Sub::fun3(int)
.data.rel.ro:000000000003EF70                 DCQ _ZN3Sub4fun4Ei      ; Sub::fun4(int)    # 这里指向Sub::fun4(int)
.data.rel.ro:000000000003EF78                 DCQ _ZN3Sub6testC3Ei    ; Sub::testC3(int)    # 这里指向Sub::testC3(int)

# ============================== [off: 0x10] SuperB =================================
.data.rel.ro:000000000003EF90 off_3EF90       DCQ _ZThn16_N3SubD1Ev   ; `non-virtual thunk to'Sub::~Sub()    # 这里指向了虚析构函数1: Sub::~Sub(), 因为声明了虚析构
.data.rel.ro:000000000003EF98                 DCQ _ZThn16_N3SubD0Ev   ; `non-virtual thunk to'Sub::~Sub()    # 这里指向了虚析构函数1: Sub::~Sub(), 因为声明了虚析构
.data.rel.ro:000000000003EFA0                 DCQ _ZThn16_N3Sub4fun1Ei ; `non-virtual thunk to'Sub::fun1(int)    # 这里指向了Sub::fun1(), 因为SuperB::fun1已被Sub:fun1覆盖
.data.rel.ro:000000000003EFA8                 DCQ _ZThn16_N3Sub4fun2Ei ; `non-virtual thunk to'Sub::fun2(int)     # 这里指向了Sub::fun2(), 因为SuperB::fun2已被Sub:fun2覆盖
.data.rel.ro:000000000003EFB0                 DCQ _ZN6SuperB6testB1Ei ; SuperB::testB1(int)     # 这里指向了SuperB::testB1(), 未被覆盖
.data.rel.ro:000000000003EFB8                 DCQ _ZN6SuperB6testB2Ei ; SuperB::testB2(int)    # 这里指向了SuperB::testB2(), 未被覆盖

# ============================== [off: 0x20] SuperC =================================
.data.rel.ro:000000000003EFD0 off_3EFD0       DCQ _ZThn32_N3SubD1Ev   ; `non-virtual thunk to'Sub::~Sub()    # 这里指向了虚析构函数1: Sub::~Sub(), 同SuperB
.data.rel.ro:000000000003EFD8                 DCQ _ZThn32_N3SubD0Ev   ; `non-virtual thunk to'Sub::~Sub()    # 这里指向了虚析构函数1: Sub::~Sub(), 同SuperB
.data.rel.ro:000000000003EFE0                 DCQ _ZThn32_N3Sub4fun1Ei ; `non-virtual thunk to'Sub::fun1(int)    # 这里指向了Sub::fun1(), 因为SuperC::fun1已被Sub:fun1覆盖
.data.rel.ro:000000000003EFE8                 DCQ _ZThn32_N3Sub4fun2Ei ; `non-virtual thunk to'Sub::fun2(int)     # 这里指向了Sub::fun2(), 因为SuperC::fun2已被Sub:fun2覆盖
.data.rel.ro:000000000003EFF0                 DCQ _ZN6SuperC6testC1Ei ; SuperC::testC1(int)     # 这里指向了SuperC::testC1(), 未被覆盖
.data.rel.ro:000000000003EFF8                 DCQ _ZN6SuperC6testC2Ei ; SuperC::testC2(int)    # 这里指向了SuperC::testC2(), 未被覆盖
.data.rel.ro:000000000003F000                 DCQ _ZThn32_N3Sub6testC3Ei ; `non-virtual thunk to'Sub::testC3(int)    # 这里指向了Sub::testC3(), 因为SuperC::testC3已被Sub::testC3覆盖实现
```
以上虚表中，类型为`non-virtual thunk`的为中转函数(在实际生产环境中IDA不一定能直接识别出`non-virtual thunk`)，表明该位置的函数已被覆盖，通过该中转函数将调用跳转到覆盖后的新函数中，举例：

以上`SuperB::fun2`与`SuperC::fun2`均被`Sub::fun2`覆盖， 所以该虚表项直接被替换为中转函数地址，将调用转到`Sub::fun2()`中以实现多态。该中转函数如下：
```
.text:00000000000097E4 ; __int64 __fastcall `non-virtual thunk to'Sub::fun2(Sub *__hidden this, int)
.text:00000000000097E4                 WEAK _ZThn16_N3Sub4fun2Ei
.text:00000000000097E4 _ZThn16_N3Sub4fun2Ei                    ; DATA XREF: .data.rel.ro:000000000003EFA8o
.text:00000000000097E4
.text:00000000000097E4 var_C           = -0xC
.text:00000000000097E4 var_8           = -8
.text:00000000000097E4
.text:00000000000097E4 ; FUNCTION CHUNK AT .plt:0000000000008840 SIZE 00000010 BYTES
.text:00000000000097E4
.text:00000000000097E4                 SUB             SP, SP, #0x10
.text:00000000000097E8                 STR             X0, [SP,#0x10+var_8]
.text:00000000000097EC                 STR             W1, [SP,#0x10+var_C]
.text:00000000000097F0                 LDR             X0, [SP,#0x10+var_8]
.text:00000000000097F4                 SUBS            X0, X0, #0x10 ; this    # 将this减去0x10后指针回到Sub头(因为此函数被调用前，this会被加上0x10)
.text:00000000000097F8                 ADD             SP, SP, #0x10
.text:00000000000097FC                 B               ._ZN3Sub4fun2Ei ; Sub::fun2(int)  # 转到Sub::fun2()
.text:00000000000097FC ; End of function `non-virtual thunk to'Sub::fun2(int)
.text:00000000000097FC
```
以上中转函数中的this-0x*也可作为识别`non-virtual thunk`的一种特征.

## d.虚函数调用
**a). 首先分析，调用的目标为：`父类`中未被`子类`覆盖的虚函数，如： 调用`sub->testB1()`**

C++代码：
```
sub->testB1((int)env->GetVersion());
```
对应汇编代码如下：
```
.text:0000000000008B74                 LDR             X8, [X21,#0x10]!   # 1. X21为this, this+0x10定位到SuperB的头部
.text:0000000000008B78                 LDR             X9, [X19]
.text:0000000000008B7C                 LDR             X22, [X8,#0x20]    # 2. X8为SuperB头部的虚表(构造时已被替换为Sub的)，[X8+0x20]即可得到SuperB::testB1的地址(对照上节`c.虚表`中的虚表内容即可证明)。
.text:0000000000008B80                 LDR             X8, [X9,#0x20]
.text:0000000000008B84                 MOV             X0, X19
.text:0000000000008B88                 BLR             X8    # 调用env->GetVersion()
.text:0000000000008B8C                 MOV             W1, W0
.text:0000000000008B90                 MOV             X0, X21
.text:0000000000008B94                 BLR             X22    # 3.调用SuperB::testB1()
```
**总结:**

调用没有被`子类`覆盖的方法时，依旧会依赖`父类`的`虚表`(本例中使用SuperB的虚表进行调用).

**b). 接下来分析，调用的目标为：`父类`中已被`子类`覆盖的虚函数，如：调用`sub->fun2()`**

C++代码：
```
sub->fun2((int)env->GetVersion());
```
对应汇编代码如下：
```
.text:0000000000008AE0                 LDR             X8, [X20]    # 1. X20为this, 这里直接取出Sub/SuperA的虚表(因为Sub与SuperA的虚表混合为一体，并且该虚表已被Sub替换)
.text:0000000000008AE4                 LDR             X9, [X19]
.text:0000000000008AE8                 LDR             X21, [X8,#0x18]    # 2. 取出虚表中index为3的项， 里面存放这Sub::fun2的指针(对照上节`c.虚表`中的虚表内容即可证明)。
.text:0000000000008AEC                 LDR             X8, [X9,#0x20]
.text:0000000000008AF0                 MOV             X0, X19
.text:0000000000008AF4                 BLR             X8  # 调用env->GetVersion()
.text:0000000000008AF8                 MOV             W1, W0
.text:0000000000008AFC                 MOV             X0, X20
.text:0000000000008B00                 BLR             X21    # 3. 调用Sub::fun2()
```
**总结:**

调用被`子类`覆盖的方法时，会直接使用`子类`的`虚表`.