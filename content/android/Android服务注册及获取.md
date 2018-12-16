+++
date = "2017-05-14T18:32:05+08:00"
draft = false
title = "Android服务注册及获取"
+++

> 本文以Android 5.1为例展开，在7.0上细节会略有差异，但基本一致：

### Service大致分为两种
**1. 系统服务SystemService**

// 以下展示的框架为标准的Android SystemService的调用结构。如：Context.LOCATION_SERVICE. 
[services.jar部分]

* 服务提供方通过ServiceManager.addService()注册服务

* 服务调用方通过ServiceManager.getService()获取服务，为IBinder类型. (通常供framework.jar使用，framework.jar内部会再包装Wrapper类，最后供上层APP使用.)

[framework.jar部分]

* 使用ContextImpl.registerService() (在7.0上为SystemServiceRegistry), 将通过ServiceManager.getService()的到的IBinder对象注册到framework中.

* 在APP中使用context.getSystemService(LOCATION_SERVICE)

**2. 普通APP的Binder服务**
     服务提供方在内部最终通过什么注册服务?  (TODO: 分析如应用中的服务是如何注册的)
     服务调用放通过bindService获得服务，为IBinder类型

以上两种方法对于IBinder都一样，使用asInterface将IBinder转为可方便调用的Client端接口：
```
    public static IPowerManager asInterface(IBinder arg2) {
        IPowerManager v1 = null;
        if(arg2 == null) {
            return v1;
        }

        IInterface v0 = arg2.queryLocalInterface("android.os.IPowerManager");
        if(v0 != null && ((v0 instanceof IPowerManager))) {
            return ((IPowerManager)v0);  // 如果该Service在本进程内，则直接返回。
        }

        return new Proxy(arg2);  // Service不在本进程内，使用Proxy进行代理访问(内部通过mRemote.transact()访问远端服务)。
    }
```

### 1个标准Android SystemService的完整实现

**首先在services.jar中添加服务**

在service.jar可以通过调用SystemService.publishBinderService()或ServiceManager.addService将1个Service加入Binder队列.
```java
SystemManager.addService(LOCATION_SERVICE, new LocationManagerService(context));
// or
SystemService.publishBinderService(POWER_SERVICE, new BinderService(this, null));
```

**最后在framework中使用ContextImpl.registerService()将包装Binder后的Wrapper对象加入队列，之后便可以在APP开发中直接使用context.getSystemService(XXXX_SERVICE)获取Manager(Wrapper)进行服务操作:**
```java
// xref: /frameworks/base/core/java/android/app/ContextImpl.java
 static {
      // 将对应Service的Manager(Wrapper)加入管理器中， 以后便可以直接在APP开发中使用context.getSystemService(XXXX_SERVICE)来获取指定服务的Manager(Wrapper).
    /////////////////////////////////////// LOCATION_SERVICE //////////////////////////////////////
    registerService(LOCATION_SERVICE, new ServiceFetcher() {
            public Object createService(ContextImpl ctx) {
                IBinder b = ServiceManager.getService(LOCATION_SERVICE);
                return new LocationManager(ctx, ILocationManager.Stub.asInterface(b)); 
            }});
    // ...
}
```

### 其它
**addService / getService大致实现**
```
// pos: framework.jar
package android.os;
public final class ServiceManager {
    // ...

    public static void addService(String arg3, IBinder arg4) {
        try {
            ServiceManager.getIServiceManager().addService(arg3, arg4, false);
        }
        catch(RemoteException v0) {
            Log.e("ServiceManager", "error in addService", ((Throwable)v0));
        }
    }

    public static IBinder getService(String arg5) {
        IBinder v4 = null;
        try {
            Object v1 = ServiceManager.sCache.get(arg5);
            if(v1 != null) {
                return ((IBinder)v1);
            }

            return ServiceManager.getIServiceManager().getService(arg5);
        }
        catch(RemoteException v0) {
            Log.e("ServiceManager", "error in getService", ((Throwable)v0));
            return v4;
        }
    }
    // ...
}
```
**publishBinderService的实现**
```java
// pos: services.jar
package com.android.server;
public abstract class SystemService {
    // ...
    protected final void publishBinderService(String arg1, IBinder arg2, boolean arg3) {
        ServiceManager.addService(arg1, arg2, arg3);  // 最终会调入framework.jar中的android.os.ServiceManager
    }
    // ...
}
```

