+++
date = "2018-04-26T11:07:43+08:00"
draft = false
title = "王者荣耀荣耀C#脚本的提取"
+++

王者荣耀采用流行的Unity3d框架开发，主要游戏逻辑使用C#编写， 但目前尚未确认是否有使用到Lua。

## 一、关于Unity3D
Unity3D是一款专用于3D游戏开发的跨平台框架，支持Lua、C#、JS脚本语言开发游戏逻辑。但目前使用较为广泛的还是C#，`王者荣耀使用的正是C#`。

## 二、Unity3D with C#手游的通用修改方案
Unity3D只是提供3D框架，其对C#脚本的支持能力完全依赖开源项：

mono (https://github.com/mono/mono, `在apk的lib下面有一个名为libmono.so就是使用了mono项目的特征`)。

![img](/android/王者荣耀CSharp脚本的提取/1.png "img")

Unity3D编译好后，最终的C#文件保存在`×.apk/assets/bin/Data/Managed/中的Assembly-CSharp.dll` 和 `Assembly-Csharp-firstpass.dll` 两个文件，其都是编译好的C#文件。

![img](/android/王者荣耀CSharp脚本的提取/2.png "img")

对Unity3D游戏逻辑的修改最后就是针对以上这两个dll文件，`这两个dll文件是windows上的PE格式`

![img](/android/王者荣耀CSharp脚本的提取/3.png "img")

正常无保护的情况下可以很明显的看出以上两个文件的特征，使用一些C#的反编译工具(如:`.Net Reflector、ILSpy`)可以很容易的得到源码逻辑：

![img](/android/王者荣耀CSharp脚本的提取/4.png "img")

最后利用如.NetReflector工具对 dll文件中的游戏逻辑做修改，并替换dll文件后打包。便完成了游戏的修改。

## 三、对王者荣耀进行分析
**1. 在王者荣耀中，这两个文件(`Assembly-CSharp.dll` 和 `Assembly-Csharp-firstpass.dll`)均被加密保护，已经无法看到标准的PE头。**

![img](/android/王者荣耀CSharp脚本的提取/5.png "img")

**2. C#解析引擎libmono.so也被加壳混淆。**

![img](/android/王者荣耀CSharp脚本的提取/6.png "img")

**3. 游戏存在反调试保护，无法用断点断到关键函数，会自动退出。**

根据阅读mono开源项目得知其的加载assembly文件的方法名为：

mono_image_open_from_data_with_name(`char *data, guint32 data_len`, gboolean need_copy, MonoImageOpenStatus *status, gboolean refonly, const char *name)

并且该函数在libmono.so中对外导出

* 第1个参数data, 指向的是Assembly-*.dll的完整文件数据。

* 第2个参数是data_len，是数据长度。

首先查看libmono.so中的`mono_image_open_from_data_with_name`导出函数，发现已被加密混淆。怀疑TX可能对mono做了特殊的定制(如果是的话会比较棘手)


**4. 通过进程注入对libmono.so脱壳还原代码后发现并无特殊定制：**

说明调用到`mono_image_open_from_data_with_name`后参数中的一定是明文dll数据。

![img](/android/王者荣耀CSharp脚本的提取/7.png "img")

既然没有定制libmono.so，所以直接采用hook方式进行(可使用开源项目adbi、或自己开发)hook这个函数(`mono_image_open_from_data_with_name`)
但由于Assembly-*.dll文件在游戏启动时就被加载，所以需要考虑hook时机：

1.使用debug模式启动王者荣耀，使其处于等待调试状态...

2.先hook住libc.so中的__openat函数，当发现有Assembly-*.dll文件被打开时，再对libmono.so中的`mono_image_open_from_data_with_name`挂钩，这样可以保证hook不错过时机，并且外壳代码已经还原完毕。

3.最后将`mono_image_open_from_data_with_name` 的数据写出文件。

**5. DUMP成功后的脚本，使用反编译工具可以看到游戏逻辑，并可以修改：**

![img](/android/王者荣耀CSharp脚本的提取/8.png "img")

本次分析验证到此。

## 四、基于此开发辅助工具的一点思路

定位到游戏关键逻辑后（如：地图全亮）。使用C#反编译修改工具(如:`.Net Reflector、ILSpy`)对脚本进行修改。最后利用hook将修改后的脚本替换到内存中去，让游戏使用新的脚本。即可达成修改游戏逻辑的目的。