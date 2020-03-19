# dex 文件
能够被DVM识别，并且加载执行的文件格式


## 堆区
 作用： 所有通过new 创建的对象的内存都在堆中分配
 特点： 是虚拟机中最大的一块内存，是GC要回收的一部分

## ClassLoader
### java 中的classloader
![Alt text](https://github.com/hoyouly/BlogResource/raw/master/imges/java_classloader.png)

类加载流程
![Alt text](https://github.com/hoyouly/BlogResource/raw/master/imges/class_Loader_loding.png)
### Android 中的classloader
#### 种类
* BootClassLoader 加载Android FW层的字节码文件
* PathClassLoader APPClassLoader 加载已经安装到系统中的APk中class 文件
* DexClassLoader  custom classloader 作用一样，用来加载指定目录字节码文件
* BaseDexClassLoader  是PathClassLoader 和DexClassLoader的父类

app 中至少需要BootClassLoader 和PathClassLoader才能运行


特点
* 双亲代理模型  

```java
ClassLoader.java
protected Class<?> loadClass(String name, boolean resolve)throws ClassNotFoundException{
             // First, check if the class has already been loaded
             Class<?> c = findLoadedClass(name);
             if (c == null) {
                 try {
                     if (parent != null) {
                         c = parent.loadClass(name, false);
3                    } else {
                         c = findBootstrapClassOrNull(name);
                     }
                 } catch (ClassNotFoundException e) {
                     // ClassNotFoundException thrown if class not found
                     // from the non-null parent class loader
                 }

                 if (c == null) {
                     // If still not found, then invoke findClass in order
                     // to find the class.
                     c = findClass(name);
                 }
             }
             return c;
     }
```
![Alt text](https://github.com/hoyouly/BlogResource/raw/master/imges/loadclass.png)
如图，当前classloader是否加载过当前类，如果加载过，就直接返回，如果没有加载过，则看父类有没有加载过，如果加载过，则直接返回，如果都没加载过，则子classloader调用 findClass()真正加载类
findClass()在classLoader中是一个空实现，所以需要到具体的子类中去处理，我们知道，PathClassLoader和DexClassLoader都是继承BaseDexClassloader,所以可以先在BaseDexClassloader中去查找
```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
         List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
         Class c = pathList.findClass(name, suppressedExceptions);
         if (c == null) {
             ClassNotFoundException cnfe = new ClassNotFoundException(
                     "Didn't find class \"" + name + "\" on path: " + pathList);
             for (Throwable t : suppressedExceptions) {
                 cnfe.addSuppressed(t);
             }
             throw cnfe;
         }
         return c;
     }
```
这里面关键的就是执行pathList.findClass()方法，

* 类加载的共享功能
FW层的类一旦加载过，就会缓存到内存总，其他classloader再次加载的时候，就直接得到
* 类加载的隔离功能
不同路径的classloader

## 热修复
动态更新，
不需要任何感知，就可以完成bug的修复和小功能的更新，
只是一个亡羊补牢的手段，不到万不得已不会使用
几种热修复技术的对比
![Alt text](https://github.com/hoyouly/BlogResource/raw/master/imges/hot_fix_comp.png)


技术选型
我们的需求是什么，需求是衡量一些的标准
能满足需求的情况下，学习成本最低，使用简单，调试简单，维护简单
学习成本一样的情况下，优先选择大公司方案