---
layout: post
title: Qmeu的QOM框架分析
---

## 面向对象编程概念

QOM是Qemu Object Model的缩写，它是qemu提供的一个面向对象编程的框架。面向对象编程通过将程序拆解成一个个彼此相关的对象，每个对象有自己的属性、状态和方法，拥有共同特征的对象被抽象成一个类，而又根据抽象的程度，可以把类进一步划分出等级，子类继承父类等等。

从而，可以将一个大型且复杂的程序，以对象为粒度，拆分成不同的模块，梳理出不同的层级，使得整个系统解耦，易于更新迭代和维护，保证了代码的复用性，可读性，可伸缩性，开发效率性等。另外，在使用面向对象思想编程时，前人已经把他们踩过的坑和获得的经验总结成很多优秀的设计模式，我们可以学习和借鉴。

面向对象编程可以分为两种思路：基于类的面向对象编程和基于原型的面向对象编程，这里分析的是基于类的面向对象编程，QOM也是基于该思想构建的。

### 面向对象编程的关键术语：

1. 类(class)：是对一类事物的抽象，类只是一个抽象概念，并不对应真实存在的事物。它定义了一类事物所共有的特征和行为。比如，当把人划分作一个类时，人的这些特征（名字，年龄，性别等）可以被抽象为特定数据格式的变量，这些变量表示了人的属性和状态等；另外，人的这些行为（说话，行走，睡觉等）可以抽象为方法函数。当然，在抽象一个类的时候，一般只会视实际需求来添加必要的属性和方法。
2. 对象(object)：是一个类的实例，它是一个实实在在存在的事物。
3. 类变量(class variables)：只属于类，从程序角度看，它在内存中只存在一份，对象中仅仅包含对类的变量的引用。比如，属性“最大身高”，它表示了对属于人类的所有对象的身高限制，所以放在类的变量就很合适，而属性“名字”就无法作为类的变量，因为不同对象的有不同的名字。
4. 实例变量(instance variables)：属于每个对象自己的数据，每个对象的变量在内存中都会有自己的副本，其中包含了每个对象自己的特定属性或者状态等，比如不同的人会有不同的名字。
5. 类方法(class methods)：只属于类，对象中仅仅包含了对类的方法的引用，对象不能修改它，或者说归属于一个类的所有对象调用了同一个方法实体。
6. 实例方法(instance methods)：属于每个对象自己的行为，也因此每个对象可以调用不同的方法，一般在程序中，对象的方法定义为一个函数指针，那么我们可以在对象的构造函数中初始化对象的方法，从而不同的对象指向不同的方法实体。
7. 构造函数(constructor)：当创建一个实例时所调用的函数被称为构造函数，构造函数完成对实例的创建和初始化，比如申请实例所占用的内存，初始化实例的变量和方法。

### 面向对象编程的机制

1. 封装(encapsulation)：将对象的一些变量和行为封装成方法暴露出去，而这些变量和方法则作为对象的私有数据，其它对象不可访问和修改，他们只可以调用封装后的方法，这样可以在方法的实现中增加检查，限制外部对私有数据的直接读取和篡改，从而提高了程序的安全性；另一方面，通过封装隐藏了对象的内部细节，实现了解耦，当我们需要修改对象的内部实现时，无需修改外部代码，有益于多人合作开发一个大型项目。
2. 抽象(abstraction)：将复杂的现实问题抽象成不同具体程度的类，它可以为具体问题找到最恰当的类定义，并且可以在最恰当的继承级别解释问题。比如，“旺财”在大多时候都被当做一条狗，但是如果想要让它做牧羊犬做的事，完全可以调用牧羊犬的方法。如果狗这个类还有动物的父类，那么完全可以视“旺财”为一个动物。
3. 继承(inheritance)：将一批事物抽象成不同层级的类，子类继承父类的属性和方法，并且也可以包含他们自己的。比如“旺财”是一只牧羊犬，可以把“旺财”视作牧羊犬类，而牧羊犬是犬类的一种，犬类是动物类的一种，因此可以根据具体化的程度抽象出动物类、犬类、牧羊犬类，牧羊犬类继承于犬类，犬类继承于动物类。当还需要定义一个猫类的时候，猫类也可以继承于动物类，达到动物类被复用的目的，从而提高了代码复用性，并且代码也具有相似于现实世界的层次结构，增加了代码的可读性。另外，一个类也可以从多个父类继承，称之为多重继承。
4. 多态(Polymorphism)：是指当子类继承父类时，子类的对象可以视具体需要，替换父类方法的实现或者变量的值。前面我们提到过，一般对象的方法是一个函数指针，因此，可以通过改变函数指针指向的函数来替换方法的具体实现。

## QOM分析

前面分析了面向对象编程的基本思想，目前很多编程语言都有对面向对象编程的支持，如c++，python，rust等，他们在语言特性中添加了面向对象编程的原语，比如，c++直接提供了类定义的关键字class。而编写qemu所使用的c语言，其本身并没有提供面向对象编程的语言特性，因此，qemu创建了一个QOM框架来支撑面向对象编程。

其主要提供了面向对象编程的这几个能力：
1. 抽象
2. 继承
3. 多重继承，使用interface实现
4. 多态

对于封装，QOM无法限制对象的私有数据被外部访问，这种能力应该是需要语言特性配合编译器来实现的。

### QOM的类定义

QOM的类定义可以分为三个部分: type, object class, object。

其中，type定义了继承关系，object class和object的size，构造和解构函数等，主要涉及两个结构体，struct TypeInfo是type的配置，通过定义一个TypeInfo变量并进行注册，就会创建一个TypeImpl变量，即type实体，这个type实体存储在一张hash表中，然后我们可以通过TYPE_NAME获取对应的TypeImpl。

```c
struct TypeInfo
{
    const char *name;
    const char *parent;
    size_t instance_size;
    size_t instance_align;
    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);
    bool abstract;
    size_t class_size;
    void (*class_init)(ObjectClass *klass, void *data);
    void (*class_base_init)(ObjectClass *klass, void *data);
    void *class_data;
    InterfaceInfo *interfaces;
};

struct TypeImpl
{
    const char *name;
    size_t class_size;
    size_t instance_size;
    size_t instance_align;
    void (*class_init)(ObjectClass *klass, void *data);
    void (*class_base_init)(ObjectClass *klass, void *data);
    void *class_data;
    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);
    bool abstract;
    const char *parent;
    TypeImpl *parent_type;
    ObjectClass *class;
    int num_interfaces;
    InterfaceImpl interfaces[MAX_INTERFACES];
}
```

object class则定义了类变量和类方法，object则定义了实例变量和实例方法。他们的基类定义如下：struct ObjectClass和struct Object。object class和object在这里是一个抽象的概念，可以认为它们指的是表示类定义的c语言结构体。

由于c语言没有定义专门的class关键字来帮助创建类实体，因此类的创建是通过type来完成的，首先根据TYPE_NAME查找到对应的type实体，然后根据type->class_size指定的size在内存中申请一个object class实例，最后调用构造函数type->class_init初始化object class实例，这样我们就获得了一个类实体，并且类实体中有指向type实体的指针：ObjectClass->type。

而类的实例化则是通过ObjectClass->type找到type实体，然后根据type->instance_size指定的size在内存中申请一个object实例，最后调用构造函数type->initance_init初始化object实例，并且实例中包含了指向类实体的指针：Object->class。这里需要注意的是每个object class只会有一个实例，而每个object可以有多个实例，可以认为object class和object一起组成了一个完整的对象定义，object class之所以只需要实例化一次，是因为它包含的是类变量和类方法，所有实例都只是去引用同一类实体中的类变量和类方法。也因此，object实例化之前必须先完成object class的实例化。

```c
struct ObjectClass
{
    Type type;
    GSList *interfaces;
    const char *object_cast_cache[OBJECT_CLASS_CAST_CACHE];
    const char *class_cast_cache[OBJECT_CLASS_CAST_CACHE];
    ObjectUnparent *unparent;
    GHashTable *properties;
};

struct Object
{
    ObjectClass *class;
    ObjectFree *free;
    GHashTable *properties;
    uint32_t ref;
    Object *parent;
};
```

### QOM的继承

类的继承关系建立涉及到三个部分：
1. 在创建type实体的时候指定类型的父类；
2. 在定义object class结构体的时候包含第一步所有指定父类的object class，并且放在结构体的第一个成员；
3. 在定义object结构体的时候包含第一步所指定父类的object，并且放在结构体的第一个成员。

这里以TYPE_X86_MACHINE为例，看看QOM如何实现继承：

```c
static const TypeInfo x86_machine_info = {
    .name = TYPE_X86_MACHINE,
    .parent = TYPE_MACHINE,
    .abstract = true,
    .instance_size = sizeof(X86MachineState),
    .instance_init = x86_machine_initfn,
    .class_size = sizeof(X86MachineClass),
    .class_init = x86_machine_class_init,
    .interfaces = (InterfaceInfo[]) {
         { TYPE_NMI },
         { }
    },
};
//object class
struct X86MacineClass {
	MachineClass parent;

	......
}
//object
struct X86MachineState {
	MacineState parent;

	......
}
```

TYPE_X86_MACHINE的TypeInfo中定义了其父类为TYPE_MACHINE，对应的TYPE_X86_MACHINE的object class(X86MacineClass)中包含了TYPE_MACHINE的object(MachineClass)，object也一样。这里需要注意的是结构体中的成员parent不是一个指针，而是一个实体，基类都是TYPE_OBJECT。

这里很巧妙的是，由于父类的object或者object class总是放在子类结构体定义的第一个成员，所以继承自父类的成员肯定在它所有子类成员的前面，当我们希望根据一个object实例获取他的父类或者子类的object实例时，只需要强转它的指针类型即可，object class也是如此。

对于继承关系，type创建过程中，如果发现其有父类，会先创建父类的type。object class的实例化是调用的type_initialize()，它会在实例化TYPE_X86_MACHINE的object class之前，先实例化其父类TYPE_MACHINE的object class，然后再将父类的object class实例拷贝到X86MachineState的第一个成员parent，这样子类就继承了父类的类变量和类方法。object的实例化是调用的object_new_with_type()，它会在调用TYPE_X86_MACHINE的构造函数之前，先调用其父类的构造函数来初始化结构体中的parent部分。注意，object的父类部分初始化和object class是不同，object中包含的是实例变量和实例方法，每个实例都是独特的，不能通过拷贝的方式，只能是调用父类的初始化函数，为每一个实例初始化自己的状态和属性。

### QOM的多重继承

QOM使用interface实现了多重继承，interface也是通过TypeInfo指定，如TYPE_X86_MACHINE中指定了TYPE_NMI作为它的interface，interface和正常的type很相似，但是interface中只定义了object class，没有object(WHY?)。另外，子类也会继承父类的interface。

interface在调用type_initialize()的时候被初始化，其初始化函数为：type_initialize_interface(type, interface, parent_interface)。初始化函数主要做了三件事情：

1. 创建一个新的interface type：iface_impl，它是interface的一个子类，其和父类仅仅名字有差别，这里之所以需要创建一个新的子类interface，是因为每一个继承interface的type都需要一个与interface对应的object class实例；
2. 基于new_interface创建object class，它是InterfaceClass的一个子类；
3. 初始化new_interface的InterfacClass，并插入type的链表。

```c
struct InterfaceClass {
	ObjectClass parent_class;
	ObjectClass *concrete_class;
	Type interface_type;
};
static void type_initialize_interface(TypeImpl *ti,
                                      TypeImpl *interface_type,
                                      TypeImpl *parent_type)
{
    InterfaceClass *new_iface;
    TypeInfo info = { };
    TypeImpl *iface_impl;

    info.parent = parent_type->name;
    info.name = g_strdup_printf("%s::%s",
                        ti->name, interface_type->name);
    info.abstract = true;

    iface_impl = type_new(&info);
    iface_impl->parent_type = parent_type;
    type_initialize(iface_impl);
    g_free((char *)info.name);

    new_iface = (InterfaceClass *)iface_impl->class;
    new_iface->concrete_class = ti->class;
    new_iface->interface_type = interface_type;

    ti->class->interfaces =
         g_slist_append(ti->class->interfaces, new_iface);
}
```

### QOM的抽象

QOM提供了一些接口，可以使我们方便的访问某个实例的不同抽象层级，即其父类或者子类，而且在debug模式下，还可以check我们希望获取的父类或者子类是否合法，interface也可以使用这种方法来获取，其具体的函数是object_dynamic_cast()，一般我会在定义一个类的时候，定义几个宏来方便我们获取这个类型所对应的object和object class实例，如下，然后就可以很方便的使用MY_DEVICE(obj)获取到MY_DEVICE的object实例，使用MY_DEVICE_CLASS(class)获取到MY_DEVICE的object class实例。

```c
// declared macro
OBJECT_DECLARE_SIMPLE_TYPE(MyDevice, my_device,
                           MY_DEVICE, DEVICE)

// This is equivalent to the following
typedef struct MyDevice MyDevice;
typedef struct MyDeviceClass MyDeviceClass;

G_DEFINE_AUTOPTR_CLEANUP_FUNC(MyDeviceClass, object_unref)

#define MY_DEVICE_GET_CLASS(void *obj) \
        OBJECT_GET_CLASS(MyDeviceClass, obj, TYPE_MY_DEVICE)
#define MY_DEVICE_CLASS(void *klass) \
        OBJECT_CLASS_CHECK(MyDeviceClass, klass, TYPE_MY_DEVICE)
#define MY_DEVICE(void *obj)
        OBJECT_CHECK(MyDevice, obj, TYPE_MY_DEVICE)

struct MyDeviceClass {
    DeviceClass parent_class;
};
```

### QOM的多态

QOM多态是用于object class这部分，从前面继承部分可以知道，正常情况下object class会完整的拷贝父类的类变量和方法，当我们想在子类中替换掉父类的某个方法或者变量的时候，我们在拷贝完之后，直接修改子类object class实例中的对应方法或者变量即可。但是当替换之后，父类的方法或者变量也就丢失了，不可以再追溯。QOM给了一个建议的方法来让我们手动保存保存被替换掉的父类方法或变量，在子类的object class中定义一个parent_do_something成员保存父类的方法。
```c
static void derived_class_init(ObjectClass *oc, void *data)
{
    MyClass *mc = MY_CLASS(oc);
    DerivedClass *dc = DERIVED_CLASS(oc);

    dc->parent_do_something = mc->do_something;
    mc->do_something = derived_do_something;
}

static const TypeInfo derived_type_info = {
    .name = TYPE_DERIVED,
    .parent = TYPE_MY,
    .class_size = sizeof(DerivedClass),
    .class_init = derived_class_init,
};
```

## Reference
1. [object-oriented programming (OOP)][1]
2. [Object-oriented programming][2]

[1]: https://www.techtarget.com/searchapparchitecture/definition/object-oriented-programming-OOP
[2]: https://en.wikipedia.org/wiki/Object-oriented_programming
