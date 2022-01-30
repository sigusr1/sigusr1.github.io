---
layout: post
title:  "cgroup中的cfs和rt调度简介"
date:   2021-05-12
categories:   操作系统
tags:  cgroup cfs rt 
author: coderhuo
mathjax: true
---

* content
{:toc}

本文主要以实验的形式简单介绍cgroup中cfs和rt调度的基本用法。  












# 背景 #
可以将一个或多个进程划到一个组，并限定内存、cpu等资源的使用比例，这就是cgroup的主要功能。

cfs和rt调度分别指公平调度和实时调度，二者最显著的区别可以用一个例子来说明：如果在一个单核系统上创建两个线程，二者都执行`while(1);`死循环，在cfs调度下，两个线程都会被执行，并且被执行次数基本相同（所以被称为公平调度算法）；**在rt调度下，第一个线程会永远占用cpu，第二个线程根本得不到执行**（做过嵌入式设备开发的同学可能会有感触，一不小心别的线程就会被饿死）。

# 如何开启rt调度 #

cgroup中的cpu子系统位于`/sys/fs/cgroup/cpu`,如果里面包含`cpu.rt_runtime_ms`文件，说明已经开启rt调度。

以`Ubuntu 20.04.1 LTS`为例说明如何开启：

-  



Cyber RT的模块加载机制主要分为两部分：
- 编译期进行模块注册
- 运行期加载模块并初始化

下面我们依次介绍下相关内容。

# 编译期工作 #

`common_component_example.h`中调用`CYBER_REGISTER_COMPONENT(CommonComponentSample)`注册component。  

将宏CYBER_REGISTER_COMPONENT逐步展开后的代码如下，其中：

- 模板参数Derived的值为CommonComponentSample，即本component对应的类名
- 模板参数Base的值为apollo::cyber::ComponentBase，是CommonComponentSample的间接基类
- UniqueID是一个唯一的整数，由编译期宏__COUNTER__实现

```
#define CLASS_LOADER_REGISTER_CLASS_INTERNAL(Derived, Base, UniqueID)     \
  namespace {                                                             \
  struct ProxyType##UniqueID {                                            \
    ProxyType##UniqueID() {                                               \
      apollo::cyber::class_loader::utility::RegisterClass<Derived, Base>( \
          #Derived, #Base);                                               \
    }                                                                     \
  };                                                                      \
  static ProxyType##UniqueID g_register_class_##UniqueID;                 \
  }
```

上面的宏定义了一个名为ProxyType##UniqueID的结构体，UniqueID保证了该结构体是全局唯一的，然后用这个结构体**定义了一个静态全局变量**(静态全局变量的妙用我们稍后再做介绍)。


# 运行期工作 #

运行期主要负责加载动态库`libcommon_component_example.so`，然后创建类`CommonComponentSample`的实例对象并初始化。

可执行程序mainboard通过dag文件加载对应的component，入口为`cyber/mainboard/mainboard.cc`中的`main`函数。类ModuleController负责加载动态库并初始化各component的实例。

函数`ModuleController::LoadModule(const std::string& path)`先将protocolbuffers文件`common.dag`序列化为DagConfig对象，然后调用`ModuleController::LoadModule(const DagConfig& dag_config)`，该函数主要部分解释如下：

```
bool ModuleController::LoadModule(const DagConfig& dag_config) {
  const std::string work_root = common::WorkRoot();

  for (auto module_config : dag_config.module_config()) {
    ...
    // 1. 加载动态库libcommon_component_example.so
    class_loader_manager_.LoadLibrary(load_path);

    // 2. 根据配置信息初始化各个component实例
    for (auto& component : module_config.components()) {
      const std::string& class_name = component.class_name();
      std::shared_ptr<ComponentBase> base =
          class_loader_manager_.CreateClassObj<ComponentBase>(class_name);
      if (base == nullptr || !base->Initialize(component.config())) {
        return false;
      }
      component_list_.emplace_back(std::move(base));
    }
    ... 
  }
  
  return true;
}

```

接下来分两部分讲解上面的1和2两个步骤。

## 动态库加载 ##

动态库的加载入口为`ClassLoaderManager::LoadLibrary(const std::string& library_path)`， 最终在下面的函数实现具体功能：

```
bool LoadLibrary(const std::string& library_path, ClassLoader* loader) {
  ...
  SharedLibraryPtr shared_library = nullptr;
  static std::recursive_mutex loader_mutex;
  {
    std::lock_guard<std::recursive_mutex> lck(loader_mutex);

    // 设置动态库加载前的上下文信息，后面会用到
    SetCurActiveClassLoader(loader);
    SetCurLoadingLibraryName(library_path);

    // 加载动态库
    shared_library = SharedLibraryPtr(new SharedLibrary(library_path));

    SetCurLoadingLibraryName("");
    SetCurActiveClassLoader(nullptr);
  }
  ...
}
```

SharedLibrary对象构造的时候通过`SharedLibrary::Load(const std::string& path, int flags)`调用`dlopen`加载了`libcommon_component_example.so`。

还记得前面提到的全局静态变量`g_register_class_##UniqueID`吗？当`libcommon_component_example.so`被加载的时候(`dlopen`返回前)，它是要被初始化的，也就是它的构造函数会被调用。


我们看下这个构造函数主要做了些什么。

```
// 模板参数Derived的值为CommonComponentSample
// 模板参数Base的值为apollo::cyber::ComponentBase，是CommonComponentSample的间接基类
template <typename Derived, typename Base>
void RegisterClass(const std::string& class_name,
                   const std::string& base_class_name) {

  //创建CommonComponentSample的类工厂
  utility::AbstractClassFactory<Base>* new_class_factory_obj =
      new utility::ClassFactory<Derived, Base>(class_name, base_class_name);

  /*
    设置类工厂的上下文信息：
    在so加载前，LoadLibrary中已通过SetCurActiveClassLoader和SetCurLoadingLibraryName设置了上下文信息，
    下面通过GetCurActiveClassLoader和GetCurLoadingLibraryName获取相关信息，
    然后和对象new_class_factory_obj做绑定。
  */
  new_class_factory_obj->AddOwnedClassLoader(GetCurActiveClassLoader());
  new_class_factory_obj->SetRelativeLibraryPath(GetCurLoadingLibraryName());

  GetClassFactoryMapMapMutex().lock();
  ClassClassFactoryMap& factory_map =
      GetClassFactoryMapByBaseClass(typeid(Base).name());

  /* 
    下面的代码将类工厂new_class_factory_obj加到factory_map中，
    map的key为类的名字CommonComponentSample, value为工厂对象new_class_factory_obj。
    
    factory_map又维护在另一个map中，该map作为静态变量由函数GetClassFactoryMapMap维护，
    它的key是Base的类名，即上面的typeid(Base).name()。
  */
  factory_map[class_name] = new_class_factory_obj;
  GetClassFactoryMapMapMutex().unlock();
}
```

从上面的代码可以看出，如果要获取CommonComponentSample对应的类工厂，
- 首先要以CommonComponentSample的基类apollo::cyber::ComponentBase的名字作为key从函数GetClassFactoryMapByBaseClass获取一个map
- 然后以类名CommonComponentSample作为key从这个map中获取对应的类工厂。

上面提到的两个map是嵌套的，定义如下：

```
//map的key为字符串CommonComponentSample，value为类工厂对象
using ClassClassFactoryMap =
    std::map<std::string, utility::AbstractClassFactoryBase*>;

//map的key为CommonComponentSample的基类apollo::cyber::ComponentBase的名字，value为上面的map
using BaseToClassFactoryMapMap = std::map<std::string, ClassClassFactoryMap>;
```


当函数LoadLibrary返回时，CommonComponentSample对应的类工厂已经维护在上述的二维map结构中。至此，动态库已加载完毕，并**形成了class_name、ClassLoader和ClassFactory的绑定关系**。


## Component 初始化 ##

回到上文的ModuleController::LoadModule函数，下面的代码根据类名初始化Component实例：

```
// 2. 根据配置信息初始化module中各个component实例
for (auto& component : module_config.components()) {
  const std::string& class_name = component.class_name();

  // class_name即CommonComponentSample，该component创建完成后，放在component_list_中统一维护
  std::shared_ptr<ComponentBase> base =
      class_loader_manager_.CreateClassObj<ComponentBase>(class_name);
  if (base == nullptr || !base->Initialize(component.config())) {
    return false;
  }
  component_list_.emplace_back(std::move(base));
}
```

函数`ClassLoaderManager::CreateClassObj`首先根据类名找到对应的ClassLoader对象，然后使用ClassLoader对象创建component实例


```
template <typename Base>
std::shared_ptr<Base> ClassLoaderManager::CreateClassObj(
    const std::string& class_name) {

  //获取所有的ClassLoader对象
  std::vector<ClassLoader*> class_loaders = GetAllValidClassLoaders();

  //根据类名查找对应的ClassLoader对象，然后用该对象创建component实例
  for (auto class_loader : class_loaders) {
    if (class_loader->IsClassValid<Base>(class_name)) {
      return (class_loader->CreateClassObj<Base>(class_name));
    }
  }

  AERROR << "Invalid class name: " << class_name;
  return std::shared_ptr<Base>();
}
```


接下来我们看下ClassLoader是如何创建component实例的。

```
template <typename Base>
std::shared_ptr<Base> ClassLoader::CreateClassObj(
    const std::string& class_name) {
  ...
  Base* class_object = utility::CreateClassObj<Base>(class_name, this);
  ...
}
```
utility::CreateClassObj中首先根据类名查找对应的类工厂，然后由该工厂创建component实例对象。

```
template <typename Base>
Base* CreateClassObj(const std::string& class_name, ClassLoader* loader) {
  GetClassFactoryMapMapMutex().lock();

  //根据基类名称查找由<class_name, utility::AbstractClassFactoryBase*>组成的map
  ClassClassFactoryMap& factoryMap =
      GetClassFactoryMapByBaseClass(typeid(Base).name());

  //查找class_name对应的类工厂，即CommonComponentSample对应的类工厂
  AbstractClassFactory<Base>* factory = nullptr;
  if (factoryMap.find(class_name) != factoryMap.end()) {
    factory = dynamic_cast<utility::AbstractClassFactory<Base>*>(
        factoryMap[class_name]);
  }
  GetClassFactoryMapMapMutex().unlock();

  //由类工厂创建CommonComponentSample实例对象
  Base* classobj = nullptr;
  if (factory && factory->IsOwnedBy(loader)) {
    classobj = factory->CreateObj();
  }

  return classobj;
}
```

补充说明：类工厂的定义如下，在前面提到的RegisterClass函数中实例化，模板参数ClassObject的值为CommonComponentSample。所以，下面的CreateObj函数中直接new了一个CommonComponentSample对象。

```
template <typename ClassObject, typename Base>
class ClassFactory : public AbstractClassFactory<Base> {
 public:
  ClassFactory(const std::string& class_name,
               const std::string& base_class_name)
      : AbstractClassFactory<Base>(class_name, base_class_name) {}

  Base* CreateObj() const { return new ClassObject; }
};

```

至此，CommonComponentSample已完成实例化。


再回到前面的ModuleController::LoadModule函数，它调用ComponentBase::Initialize(虚函数）初始化component。

```
// class_name即CommonComponentSample
std::shared_ptr<ComponentBase> base =
    class_loader_manager_.CreateClassObj<ComponentBase>(class_name);
if (base == nullptr || !base->Initialize(component.config())) {
  return false;
}
```

由于如下的继承关系： `CommonComponentSample --> 模板类Component --> ComponentBase`。Initialize函数最终执行的是`模板类Component`中的`Initialize`函数，该函数又调用了`bool CommonComponentSample::Init()`，至此完成CommonComponentSample的初始化。