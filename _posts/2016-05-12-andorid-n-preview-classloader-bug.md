---
layout: post
title: Android N Preview ClassLoader Bug
tagline: already fixed in master branch
tags : [Android, ClassLoader]
---


## 重现
Android N 使用了Openjdk的运行库来替换原有的Dalvik运行库（Dalvik运行库是由Apache harmony项目发展而来）。N上也实现了利用classloader进行Native隔离的功能。
这也带来了兼容性问题，比如下面的code在N以前的版本可以正常运行，但是在N无法运行

    final ClassLoader cl = new ClassLoader(LoginActivity.class.getClassLoader()) {};
    DexFile dex = DexFile.loadDex(new File(b, "a.jar").getAbsolutePath(), new File(b,   
                                   "a.dex").getAbsolutePath(), 0);
    Class<?> clz = dex.loadClass("a.Hello", cl);
    Method m = clz.getDeclaredMethod("run", String.class);
    m.setAccessible(true);
    Log.d("test","result is "+m.invoke(null, lib));

a.jar中的代码：

    package a;
    
    public class Hello {
        static String run(String lib) {
            System.load(lib);
            return hi();
        }
    
        static native String hi();
    }

日志：

    JNI DETECTED ERROR IN APPLICATION: JNI IsSameObject called with pending exception java.lang.NullPointerException: 
       at java.lang.Runtime.nativeLoad(String, ClassLoader, String) (Runtime.java:-2)
       at java.lang.Runtime.doLoad(...) (Runtime.java:1060)
       at java.lang.Runtime.load0(...) (Runtime.java:895)
       at java.lang.System.load(java.lang.String) (System.java:1603)
       at ja.Hello.run(java.lang.String) (Hello.java:6)







## 分析

从上面的日志上可以明显观察到nativeLoad函数在调用IsSameObject时没有处理一个空指针异常，那么哪里可能会抛出空指针呢？
分析了nativeLoad函数的3个输入参数：
第一个参数是so的文件路径，不会是null，排除
第二个参数是类加载器，一定不会是null，排除
第三个参数，是LD_PRELOAD的值，由于上面的类加载器是ClassLoader，而不是BaseDexClassLoader类的子类，所以这个值一定是null，看样子就是这货了！

    private String doLoad(String name, ClassLoader loader) {
        ...
        String librarySearchPath = null;
        if (loader != null && loader instanceof BaseDexClassLoader) {
            BaseDexClassLoader dexClassLoader = (BaseDexClassLoader) loader;
            librarySearchPath = dexClassLoader.getLdLibraryPath();
        }
        ...
        synchronized (this) {
            return nativeLoad(name, loader, librarySearchPath);
        }
    }

跟着native调用


    Runtime_nativeLoad // libcore/ojluni/src/main/native/Runtime.c
    JVM_NativeLoad     // art/runtime/openjdkjvm/OpenjdkJvm.cc
    LoadNativeLibrary  // art/runtime/java_vm_ext.cc
    OpenNativeLibrary  // system/core/libnativeloader/native_loader.cpp
    Create             // system/core/libnativeloader/native_loader.cpp

    android_namespace_t* Create(JNIEnv* env,
                              jobject class_loader,
                              bool is_shared,
                              jstring java_library_path, // NULL
                              jstring java_permitted_path) {
        ScopedUtfChars library_path(env, java_library_path);
        ...
    }

看来这个ScopedUtfChars对象的创建是空指针的原因， YES！就是这货


    // libnativehelper/include/nativehelper/ScopedUtfChars.h
    ScopedUtfChars(JNIEnv* env, jstring s) : env_(env), string_(s) {
        if (s == NULL) {
            utf_chars_ = NULL;
            jniThrowNullPointerException(env, NULL);
        } else {
            utf_chars_ = env->GetStringUTFChars(s, NULL);
        }
    }


## 结论
所以可以确定这个是虚拟机的bug，产生的原因是

  1. Android N系统
  1. 类加载器没有继承自BaseDexClassLoader类或子类
  1. 或者getLdLibraryPath()返回空指针
  1. 另外这个用来加载so的类加载器必须加载过类，否则出错


    bool JavaVMExt::LoadNativeLibrary(...) {
        ...
        void* class_loader_allocator = nullptr;
        {
            ScopedObjectAccess soa(env);
            // As the incoming class loader is reachable/alive during the call of this function,
            // it's okay to decode it without worrying about unexpectedly marking it alive.
            mirror::ClassLoader* loader = soa.Decode<mirror::ClassLoader*>(class_loader);
            class_loader_allocator =
                Runtime::Current()->GetClassLinker()->GetAllocatorForClassLoader(loader);
            CHECK(class_loader_allocator != nullptr);
        }
        ...
    }


## 修复方案

方案1. 避免使用ClassLoader, 而使用PathClassLoader

    class CL extends ClassLoader {
        public CL(ClassLoader parent) {
            super(parent);
        }
    }

    class CL extends PathClassLoader {
        public CL(ClassLoader parent) {
            super(".", parent);
        }
    }


方案2. 在新的PathClassLoader中进行加载
首先，创建一个独立的dex，包含下面的代码

    public void load0(String lib) {
        System.load(lib);
    }
然后，代码中使用这个dex创建出PathClassLoader, 反射调用load0函数进行加载

方案3 直接调用nativeLoad函数

    void load_after_android_N(String lib, ClassLoader cl) {
      Method m = Runtime.class.getDeclaredMethod("nativeLoad",     
                   String.class, ClassLoader.class, String.class);
      m.setAccessible(true);
      String result = (String) m.invoke(null, lib, cl,   
              getApplicationInfo().nativeLibraryDir); // !!!!!!!!!!!!!!
      if (result != null) {
        throw new UnsatisfiedLinkError(result);
      }
    }

方案4 直接忽略这样的bug
AOSP的master分支上已经修复了，所以在未来的正式版N中应该不会有bug.

    // ...




