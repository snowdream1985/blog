+++
date = "2018-04-11T10:06:24+08:00"
draft = false
title = "改造linker, 实现无痕加载so模块"
+++

    最近研究了一些防内存检测方面的姿势, 异常so模块检测在对抗中占了挺重要的部分.
    由于`Bionic/linker`完全是一个用户态的实现, 所以就想着对linker的代码做一番改造, 实现一个自己的so loader, 目的是隐藏so模块,躲过检测.

目前遍历进程内已加载模块的方法大概有两个:

1. 遍历soinfo->next(本身为一个环形链表).
2. 读取/proc/pid/maps.
 
 
涉及的核心逻辑主要在: `linker.cpp`与`linker_phdr.cpp`两个文件中, `linker_phdr.cpp`负责elf内存的映射与读取, `linker.cpp`主要负责对数据进行展开和初始化.
 
**linker里面涉及的引用比较多,弄了很久才剥离出来,最后修修补补,提取出了以下几个必须的文件:**
```
    elf_machdep.h 
    diy_exec_elf.h 
    diy_linker.cpp 
    diy_linker.h 
    diy_linker_phdr.cpp 
    diy_linker_phdr.h 
```
在linker中是不允许调用`malloc`,`free`等函数的. 这是因为linker作为第一个被加载的模块(早于libc.so),所以可用资源方面比较严苛, 从代码中的一段注释可以得知原因:
```
/* >>> IMPORTANT NOTE - READ ME BEFORE MODIFYING <<<
 *
 * Do NOT use malloc() and friends or pthread_*() code here.
 * Don't use printf() either; it's caused mysterious memory
 * corruption in the past.
 * The linker runs before we bring up libc and it's easiest
 * to make sure it does not depend on any complex libc features
 *
 * open issues / todo:
 *
 * - are we doing everything we should for ARM_COPY relocations?
 * - cleaner error reporting
 * - after linking, set as much stuff as possible to READONLY
 *   and NOEXEC
 */
```
虽然上面列举了linker中的诸多限制, 但是这仅仅是对于原生linker而言. 而我们自己改造的linker在使用时周围的资源已经是非常丰富了, libc等各种库基本都已经加载完毕. 所以可以放开手脚的干一番.  废话不多, 下面开始干活.

# 第一步: 改造各种输出宏定义.
linker中有很多不同种类的调试信息输出宏, 我并不想一次性删除, 因为这些输出对后面的调错还有很大帮助, 毕竟linker里的逻辑还是比较复杂的. 我全部都用__android_log_print():
```
// linker.h

#define DL_ERR(x...) __android_log_print(ANDROID_LOG_DEBUG, "linker_ERR", x);

#define DL_WARN(fmt, x...) __android_log_print(ANDROID_LOG_DEBUG, "linker_WARN", fmt, x);

#define DL_TRACE(fmt, x...) __android_log_print(ANDROID_LOG_DEBUG, "linker_TRACE", fmt, x);

#define TRACE_TYPE(x, y...) __android_log_print(ANDROID_LOG_DEBUG, "linker_TRACE_TYPE", y);

#define DEBUG(x...) __android_log_print(ANDROID_LOG_DEBUG, "linker_DEBUG", x);

#define INFO(x...) __android_log_print(ANDROID_LOG_DEBUG, "linker_INFO", x);

#define LOOKUP 1  // 某个宏...作用忘了

#define __libc_format_buffer(b, s, f, p...) sprintf(b, f, p);     // 这里用sprintf代替, 否则就要把libc里面的一大堆文件剥离出来, 不建议入坑.
```
# 第二步: 替换DT_NEED的依赖加载
在static bool soinfo_link_image(soinfo* si)函数的后半段，有一处代码是遍历DT_NEED并加载依赖库的代码，这里将代码直接替换成dlopen(), 调用原生linker来加载依赖库.
```
// linker.cpp
  if (dynamic != NULL) {
    for (Elf32_Dyn* d = dynamic; d->d_tag != DT_NULL; ++d) {
      if (d->d_tag == DT_NEEDED) {
        const char* library_name = strtab + d->d_un.d_val;
        TRACE("\"%s\": calling constructors in DT_NEEDED \"%s\"", name, library_name);
        find_loaded_library(library_name)->CallConstructors();
      }
    }
  }
```
修改后为:
```
    for (Elf32_Dyn* d = si->dynamic; d->d_tag != DT_NULL; ++d) {
        if (d->d_tag == DT_NEEDED) {
            const char* library_name = si->strtab + d->d_un.d_val;
            DEBUG("%s needs %s", si->name, library_name);
            soinfo* lsi = (soinfo*)dlopen(library_name, 0);       // 加载其他依赖的so, 这里直接调用原生linker
            if (lsi == NULL) {
                strlcpy(tmp_err_buf, linker_get_error_buffer(), sizeof(tmp_err_buf));
                DL_ERR("could not load library \"%s\" needed by \"%s\"; caused by %s",
                       library_name, si->name, tmp_err_buf);
                return false;
            }
            *pneeded++ = lsi;
        }
    }
```
# 第三步: 去除soinfo链.
该链作为原生linker中的一条很关键的结构, 每个加载的soinfo都会被串到这条链上来. 去除soinfo链后我们加载的模块就无法被通过遍历soinfo链表检测到.
先删除掉以下变量定义, 编译一把使引用部位报错, 然后一点点清理.
```
// linker.cpp
static struct soinfo_pool_t* gSoInfoPools = NULL;
static soinfo* gSoInfoFreeList = NULL;
static soinfo* solist = &libdl_info;
static soinfo* sonext = &libdl_info;
static soinfo* somain; /* main process, always the one after libdl_info */
```
一处关键的引用在函数`soinfo_alloc`中, 这里用于分配新的`soinfo`空间:
```C
static soinfo* soinfo_alloc(const char* name) {
  if (strlen(name) >= SOINFO_NAME_LEN) {
    DL_ERR("library name \"%s\" too long", name);
    return NULL;
  }

  if (!ensure_free_list_non_empty()) {
    DL_ERR("out of memory when loading \"%s\"", name);
    return NULL;
  }

  // Take the head element off the free list.
  soinfo* si = gSoInfoFreeList;
  gSoInfoFreeList = gSoInfoFreeList->next;

  // Initialize the new element.
  memset(si, 0, sizeof(soinfo));
  strlcpy(si->name, name, sizeof(si->name));
  sonext->next = si;
  sonext = si;

  TRACE("name %s: allocated soinfo @ %p", name, si);
  return si;
}
```
由于我们剥离了全局的soinfo链, 所以这里清理掉相关代码, 直接用`malloc`代替, 修改后的`soinfo_alloc`如下:
```C
static soinfo* soinfo_alloc(const char* name) {
  if (strlen(name) >= SOINFO_NAME_LEN) {
    DL_ERR("library name \"%s\" too long", name);
    return NULL;
  }

  if (!ensure_free_list_non_empty()) {
    DL_ERR("out of memory when loading \"%s\"", name);
    return NULL;
  }

  // Initialize the new element. 
  // 以下是我们修改后的代码, 我这里直接用了new ....嘿嘿. 
  soinfo *si = new soinfo();
  memset(si, 0, sizeof(soinfo));
  strlcpy(si->name, name, sizeof(si->name));
  DL_TRACE("name %s: allocated soinfo @ %p", name, si);
  return si;
}
```

# 第四步: 从/proc/pid/maps中隐藏map信息
很多的对抗行为都发生在**/proc/pid/**目录下, 对于模块检测来说`maps`一定是非常重要的检测点. 所以下面我们修改代码, 不让模块信息出现在`maps`中.
首先理清原理: 之所以能在maps文件中看到模块信息, 是因为linker在加载so时使用`mmap(xx, xx, xx, xx, fd_, xx)`映射了so文件的fd(文件描述符). 这个操作会被内核记录下来体现在`maps`中:
```C
// linker_phdr.cpp

// Map all loadable segments in process' address space.
// This assumes you already called phdr_table_reserve_memory to
// reserve the address space range for the library.
// TODO: assert assumption.
bool ElfReader::LoadSegments() {
...
    if (file_length != 0) {
      void* seg_addr = mmap((void*)seg_page_start,
                            file_length,
                            PFLAGS_TO_PROT(phdr->p_flags),
                            MAP_FIXED|MAP_PRIVATE,
                            fd_,    // 就是这个参数....
                            file_page_start);
      if (seg_addr == MAP_FAILED) {
        DL_ERR("couldn't map \"%s\" segment %d: %s", name_, i, strerror(errno));
        return false;
      }
    }
...
}
```

下面我们必须改变直接映射fd的方式, 并且还不影响原有功能, 思路为: 先mmap一块无fd内存, 然后使用read读出so指定偏移内容填充到mmap的内存处, 最后将mmap的内存属性设置为segment的属性. 代码如下:
```C
bool ElfReader::LoadSegments() {
    if (file_length != 0) {
       // 修改后: 
       void* seg_addr = mmap((void*)seg_page_start,
                              file_length,
                              PROT_WRITE|PROT_READ,
                              MAP_FIXED|MAP_PRIVATE| MAP_ANONYMOUS,
                              0,  // 不直接映射文件...
                              0);
      if (seg_addr == MAP_FAILED) {
         DL_ERR("couldn't mmap1 \"%s\" segment %d: %s", name_, i, strerror(errno));
         return false;
      }
      if(lseek( fd_ , file_page_start, SEEK_SET ) == -1L ) { // 移动到当前segment处
          DL_ERR("couldn't lseek1 \"%s\" segment %d: %s", name_, i, strerror(errno));
          return -1;
      }
      if(-1 == read(fd_, seg_addr, file_length)) { // 读出内容到mmap出的缓存区
          DL_ERR("couldn't read \"%s\" segment %d: %s", name_, i, strerror(errno));
          return -1;
      }
      if( -1 == mprotect(seg_addr, file_length, PFLAGS_TO_PROT(phdr->p_flags))) { // 根据文件内容设置内存属性.
          DL_ERR("couldn't mprotect \"%s\" segment %d: %s", name_, i, strerror(errno));
          return -1;
      }
      DL_INFO("LoadSegments succeed:%s!", name_);
    }
}
```

## 具体思路如上, `nexus 5`+ `4.4.4`下测试通过.  对linker了解不深, 如果有偏差和错误还请大家指正. 附上代码:~
https://bbs.pediy.com/thread-216119.htm