---
title: Android智能指针
date: 2020-12-01 19:33:57
categories: 
- Android系统
tags:
- Android
- 系统
---

在开发中通常会使用引用计数来维护对象的生命周期。

每当有一个指针指向了一个new出来的对象时，就对这个对象的引用计数增加1，每当有一个指针不再使用这个对象时，就对这个对象的引用计数减少1，每次减1之后，如果发现引用计数值为0时，那么，就要delete这个对象了，这样就避免了忘记delete对象或者这个对象被delete之后其它地方还在使用的问题了。

那如何实现这个对象的引用计数呢？如果由开发人员维护，显然既不可靠，易出错，又不方便编写和维护。因此Android使用智能指针技术—能自动维护对象引用计数的技术。需要注意的是，智能指针是一个对象，而不是一个指针。这个对象代表的是另外一个真实使用的对象，当智能指针指向实际对象的时候，就是智能指针对象创建的时候，当智能指针不再指向实际对象的时候，就是智能指针对象销毁的时候

考虑这样的一个场景，系统中有两个对象A和B，在对象A的内部引用了对象B，而在对象B的内部也引用了对象A。当两个对象A和B都不再使用时，垃圾收集系统会发现无法回收这两个对象的所占据的内存的，因为系统一次只能收集一个对象，而无论系统决定要收回对象A还是要收回对象B时，都会发现这个对象被其它的对象所引用，因而就都回收不了，这样就造成了内存泄漏。这样，就要采取另外的一种引用计数技术了，即对象的引用计数同时存在强引用和弱引用两种计数，当父对象要引用子对象时，就对子对象使用强引用计数技术，而当子对象要引用父对象时，就对父对象使用弱引用计数技术，而当垃圾收集系统执行对象回收工作时，只要发现对象的强引用计数为0，而不管它的弱引用计数是否为0，都可以回收这个对象，但是，如果我们只对一个对象持有弱引用计数，当我们要使用这个对象时，就不直接使用了，必须要把这个弱引用升级成为强引用时，才能使用这个对象，在转换的过程中，如果对象已经不存在，那么转换就失败了，这时候就说明这个对象已经被销毁了，不能再使用了。

Android系统提供了强大的智能指针技术供我们使用，这些智能指针实现方案既包括简单的引用计数技术，也包括了复杂的引用计数技术，即对象既有强引用计数，也有弱引用计数，对应地，这三种智能指针分别就称为轻量级指针（Light Pointer）、强指针（Strong Pointer）和弱指针（Weak Pointer）。

无论是轻量级指针，还是强指针和弱指针，它们的实现框架都是一致的，即由对象本身来提供引用计数器，但是它不会去维护这个引用计数器的值，而是由智能指针来维护。



##轻量级指针

`/system/core/include/utils/RefBase.h`

```h
template <class T>
class LightRefBase
{
public:
    inline LightRefBase() : mCount(0) { }
    inline void incStrong(__attribute__((unused)) const void* id) const {
        mCount.fetch_add(1, std::memory_order_relaxed);
    }
    inline void decStrong(__attribute__((unused)) const void* id) const {
         //fetch_sub将原子对象的封装值减 val，并返回原子对象的旧值，整个过程是原子的
        if (mCount.fetch_sub(1, std::memory_order_release) == 1) {
            std::atomic_thread_fence(std::memory_order_acquire);
            delete static_cast<const T*>(this);
        }
    }
    //! DEBUGGING ONLY: Get current strong ref count.
    inline int32_t getStrongCount() const {
        return mCount.load(std::memory_order_relaxed);
    }

    typedef LightRefBase<T> basetype;

protected:
    inline ~LightRefBase() { }

private:
    friend class ReferenceMover;
    inline static void renameRefs(size_t n, const ReferenceRenamer& renamer) { }
    inline static void renameRefId(T* ref,
            const void* old_id, const void* new_id) { }

private:
    mutable std::atomic<int32_t> mCount;
};

```

首先它是一个模板类，拥有一个atomic<int32_t> 类型的计数器（atomic是一个原子操作类，在c++11中新定义的）提供了desString 和 incString的计数器增减方案，还有一个getStringCount的获取计数器数量的方法。

在decStrong方法中，如果强引用全部删除之后，我们会delete本对象。

轻量级指针的实现类为sp，它不仅仅是LightRefBase计数器的智能指针，同时也是强指针引用计数器的智能指针。

`system/core/include/utils/StrongPointer.h`

```h
template<typename T>
class sp {
public:
    inline sp() : m_ptr(0) { }

    sp(T* other);
    sp(const sp<T>& other);
    sp(sp<T>&& other);
    template<typename U> sp(U* other);
    template<typename U> sp(const sp<U>& other);
    template<typename U> sp(sp<U>&& other);

    ~sp();

    // Assignment

    sp& operator = (T* other);
    sp& operator = (const sp<T>& other);
    sp& operator = (sp<T>&& other);

    template<typename U> sp& operator = (const sp<U>& other);
    template<typename U> sp& operator = (sp<U>&& other);
    template<typename U> sp& operator = (U* other);

    //! Special optimization for use by ProcessState (and nobody else).
    void force_set(T* other);

    // Reset

    void clear();

    // Accessors

    inline  T&      operator* () const  { return *m_ptr; }
    inline  T*      operator-> () const { return m_ptr;  }
    inline  T*      get() const         { return m_ptr; }

    // Operators

    COMPARE(==)
    COMPARE(!=)
    COMPARE(>)
    COMPARE(<)
    COMPARE(<=)
    COMPARE(>=)

private:    
    template<typename Y> friend class sp;
    template<typename Y> friend class wp;
    void set_pointer(T* ptr);
    T* m_ptr;
};
```

sp类也是一个模版类，T表示对象的实际类型，它也是必须继承了LightRefBase的。成员变量m_ptr是一个指针，它是在构造函数里初始化的，指向实际引用的对象。所以下面这两个构造函数实际就是调用了LightRefBase的成员函数incStrong来增加引用计数。

构造函数

```h
template<typename T>
sp<T>::sp(T* other)
        : m_ptr(other) {
    if (other)
        other->incStrong(this);
}

template<typename T>
sp<T>::sp(const sp<T>& other)
        : m_ptr(other.m_ptr) {
    if (m_ptr)
        m_ptr->incStrong(this);
}
```

析构函数

调用了LightRefBase的成员函数decStrong来减少引用计数

```h
template<typename T>
sp<T>::~sp() {
    if (m_ptr)
        m_ptr->decStrong(this);
}
```



### 实例分析

`external/lightpointer/lightpointer.cpp`

```c++
#include <stdio.h>
#include <utils/RefBase.h>

using namespace android;

class LightClass : public LightRefBase<LightClass>
{
public:
    LightClass()
    {
        printf("Construce LightClass Object");
    }
    virtual ~LightClass()
    {
        printf("Destory LightClass Object");
    }
};

int main(int argc, char **argv)
{
    LightClass *pLightClass = new LightClass();
    sp<LightClass> lpOut = pLightClass;
    printf("Light Ref Count:%d \n", pLightClass->getStrongCount());

    {   //作用域
        sp<LightClass> lpInner = lpOut;
        printf("Light Ref Count:%d \n", pLightClass->getStrongCount());
       //作用域结束，lpInner析构
    }

    printf("Light Ref Count:%d \n", pLightClass->getStrongCount());
    return 0;
}
```

`external/lightpointer/Android.mk`

```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE := lightpointer
LOCAL_SRC_FILES := lightpointer.cpp
LOCAL_SHARED_LIBRARIES := \
			libcutils \
			libutils
include $(BUILD_EXECUTABLE)
```

编译打包：

```shell
mmm ./external/lightpointer/
make snod
```

烧录新镜像后：

```
adb shell
cd system/bin/
./lightpointer

Construce LightClass Object
Light Ref Count: 1
Light Ref Count: 2
Light Ref Count: 1 
Destory LightClass Object
```

可以看到pLightClass对象赋值给lpOut智能指针后，引用计数加1，然后在局部作用域中，定义一个智能指针lpInner，并且指向了原来的对象，引用计数再次加1，当局部作用域结束时，lpInner析构，引用计数减1。

## 强指针

`system/core/include/utils/RefBase.h`

```h
class RefBase
{
public:
            void            incStrong(const void* id) const;
            void            decStrong(const void* id) const;
    
            void            forceIncStrong(const void* id) const;

            //! DEBUGGING ONLY: Get current strong ref count.
            int32_t         getStrongCount() const;

    class weakref_type
    {
    public:
        RefBase*            refBase() const;
        
        void                incWeak(const void* id);
        void                decWeak(const void* id);
        
        // acquires a strong reference if there is already one.
        bool                attemptIncStrong(const void* id);
        
        // acquires a weak reference if there is already one.
        // This is not always safe. see ProcessState.cpp and BpBinder.cpp
        // for proper use.
        bool                attemptIncWeak(const void* id);

        //! DEBUGGING ONLY: Get current weak ref count.
        int32_t             getWeakCount() const;

        //! DEBUGGING ONLY: Print references held on object.
        void                printRefs() const;

        //! DEBUGGING ONLY: Enable tracking for this object.
        // enable -- enable/disable tracking
        // retain -- when tracking is enable, if true, then we save a stack trace
        //           for each reference and dereference; when retain == false, we
        //           match up references and dereferences and keep only the 
        //           outstanding ones.
        
        void                trackMe(bool enable, bool retain);
    };
    
            weakref_type*   createWeak(const void* id) const;
            
            weakref_type*   getWeakRefs() const;

            //! DEBUGGING ONLY: Print references held on object.
    inline  void            printRefs() const { getWeakRefs()->printRefs(); }

            //! DEBUGGING ONLY: Enable tracking of object.
    inline  void            trackMe(bool enable, bool retain)
    { 
        getWeakRefs()->trackMe(enable, retain); 
    }

    typedef RefBase basetype;

protected:
                            RefBase();
    virtual                 ~RefBase();
    
    //! Flags for extendObjectLifetime()
    enum {
        OBJECT_LIFETIME_STRONG  = 0x0000,
        OBJECT_LIFETIME_WEAK    = 0x0001,
        OBJECT_LIFETIME_MASK    = 0x0001
    };
    
            void            extendObjectLifetime(int32_t mode);
            
    //! Flags for onIncStrongAttempted()
    enum {
        FIRST_INC_STRONG = 0x0001
    };
    
    virtual void            onFirstRef();
    virtual void            onLastStrongRef(const void* id);
    virtual bool            onIncStrongAttempted(uint32_t flags, const void* id);
    virtual void            onLastWeakRef(const void* id);

private:
    friend class weakref_type;
    class weakref_impl;
    
                            RefBase(const RefBase& o);
            RefBase&        operator=(const RefBase& o);

private:
    friend class ReferenceMover;

    static void renameRefs(size_t n, const ReferenceRenamer& renamer);

    static void renameRefId(weakref_type* ref,
            const void* old_id, const void* new_id);

    static void renameRefId(RefBase* ref,
            const void* old_id, const void* new_id);

        weakref_impl* const mRefs;
};
```

同样定义了 incStrong 和decStrong方法，用于增减引用计数（这也是为什么强指针也能使用sp<T>类来控制引用计数的原因）。和轻量级指针引用计数不同的是，我们的引用计数量不再是一个int类型来记录引用数量，而是使用了一个weakref_impl指针类型的变量mRefs。

weakref_impl继承自上面的weakref_type

`system/core/libutils/RefBase.cpp`

```cpp
class RefBase::weakref_impl : public RefBase::weakref_type
{
public:
    std::atomic<int32_t>    mStrong;
    std::atomic<int32_t>    mWeak;
    RefBase* const          mBase;
    std::atomic<int32_t>    mFlags;

#if !DEBUG_REFS

    weakref_impl(RefBase* base)
        : mStrong(INITIAL_STRONG_VALUE)
        , mWeak(0)
        , mBase(base)
        , mFlags(0)
    {
    }

    void addStrongRef(const void* /*id*/) { }
    void removeStrongRef(const void* /*id*/) { }
    void renameStrongRefId(const void* /*old_id*/, const void* /*new_id*/) { }
    void addWeakRef(const void* /*id*/) { }
    void removeWeakRef(const void* /*id*/) { }
    void renameWeakRefId(const void* /*old_id*/, const void* /*new_id*/) { }
    void printRefs() const { }
    void trackMe(bool, bool) { }

#else
......
#endif
};
```

这个里面实际有用的就是最上面定义的4个成员变量，mStrong 和 mWeak分别用来做强引用和弱引用的计数，mBase指向实现引用计数的对象本身，mFlags是标志位，默认值是0。

强指针的实现类依然为sp，上面我们知道sp中实际是调用了RefBase类的incStrong和decStrong来增加和减少引用计数的。我们先来看一下incStrong。

`system/core/libutils/RefBase.cpp`

```c++
void RefBase::incStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->incWeak(id);
    
    refs->addStrongRef(id);
    const int32_t c = refs->mStrong.fetch_add(1, std::memory_order_relaxed);
    ALOG_ASSERT(c > 0, "incStrong() called on %p after last strong ref", refs);
#if PRINT_REFS
    ALOGD("incStrong of %p from %p: cnt=%d\n", this, id, c);
#endif
    if (c != INITIAL_STRONG_VALUE)  {
        return;
    }

    int32_t old = refs->mStrong.fetch_sub(INITIAL_STRONG_VALUE,
            std::memory_order_relaxed);
    // A decStrong() must still happen after us.
    ALOG_ASSERT(old > INITIAL_STRONG_VALUE, "0x%x too small", old);
    refs->mBase->onFirstRef();
}
```

首先调用incWeak，弱引用加1。（**从这里可以看到增加对象的强引用，也会增加对象的弱引用计数，即一个对象的弱引用计数一定大于等于它的强引用计数**）

然后调用addStrongRef，这个函数为空实现。

关键是mStrong.fetch_add，在强引用上加1，返回值c为增加前原值，如果c等于INITIAL_STRONG_VALUE，表明是第一次调用，就会调用fetch_sub，将mStrong归1。

```
#define INITIAL_STRONG_VALUE (1<<28)
```

因为mStrong的初始值不为0，而是INITIAL_STRONG_VALUE，也就是1<<28，fetch_add后就成了1<<28+1，所以后面调用fetch_sub减掉INITIAL_STRONG_VALUE初始值，才会把mStrong值置为1。可以把这个操作看为一个判断是否第一次被强引用，相比使用boolean去标记优雅很多，后面refs->mBase->onFirstRef()调用是个空实现，由开发者自己定义。

然后看decStrong

```c++
void RefBase::decStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->removeStrongRef(id);
    const int32_t c = refs->mStrong.fetch_sub(1, std::memory_order_release);
#if PRINT_REFS
    ALOGD("decStrong of %p from %p: cnt=%d\n", this, id, c);
#endif
    ALOG_ASSERT(c >= 1, "decStrong() called on %p too many times", refs);
    if (c == 1) {
        std::atomic_thread_fence(std::memory_order_acquire);
        refs->mBase->onLastStrongRef(id);
        int32_t flags = refs->mFlags.load(std::memory_order_relaxed);
        if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            delete this;
            // Since mStrong had been incremented, the destructor did not
            // delete refs.
        }
    }
    // Note that even with only strong reference operations, the thread
    // deallocating this may not be the same as the thread deallocating refs.
    // That's OK: all accesses to this happen before its deletion here,
    // and all accesses to refs happen before its deletion in the final decWeak.
    // The destructor can safely access mRefs because either it's deleting
    // mRefs itself, or it's running entirely before the final mWeak decrement.
    refs->decWeak(id);
}
```

removeStrongRef()也是一个空实现。

fetch_sub对对象的计数减1，如果c == 1，即没有其它引用了，先调用onLastStrongRef，与onFirstRef对应，也可以自定义。

接下来先看一个枚举：

```
//! Flags for extendObjectLifetime()
    enum {
        OBJECT_LIFETIME_STRONG  = 0x0000,
        OBJECT_LIFETIME_WEAK    = 0x0001,
        OBJECT_LIFETIME_MASK    = 0x0001
    };
```

 这个枚举量是用来控制对象生命周期的，如果设置为OBJECT_LIFETIME_STRONG表示，对象如果强引用计数器为0，就表示生命周期终止，需要delete，如果设置为OBJECT_LIFETIME_WEAK,表示，只有当弱引用和强引用计数器都为0时，才会delete对象。

`if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG)` 判断对象的生命周期类型，如果是OBJECT_LIFETIME_STRONG,那么会直接delete自己。delete会调用析构函数。

```c++
RefBase::~RefBase()
{
    if (mRefs->mStrong.load(std::memory_order_relaxed)
            == INITIAL_STRONG_VALUE) {
        // we never acquired a strong (and/or weak) reference on this object.
        delete mRefs;
    } else {
        // life-time of this object is extended to WEAK, in
        // which case weakref_impl doesn't out-live the object and we
        // can free it now.
        int32_t flags = mRefs->mFlags.load(std::memory_order_relaxed);
        if ((flags & OBJECT_LIFETIME_MASK) != OBJECT_LIFETIME_STRONG) {
            // It's possible that the weak count is not 0 if the object
            // re-acquired a weak reference in its destructor
            if (mRefs->mWeak.load(std::memory_order_relaxed) == 0) {
                delete mRefs;
            }
        }
    }
    // for debugging purposes, clear this.
    const_cast<weakref_impl*&>(mRefs) = NULL;
}
```

如果该对象从没有过强引用，释放mRefs，如果当前对象生命周期是受弱引用控制的，那么只有在弱引用计数为0的时候，才会去销毁mRefs。

然后析构函数中做了一件很重要的事情，释放mRefs计数对象（我们在构造函数的时候new的，所以需要确保释放）。

回到decStrong中，最后调用了refs->decWeak(id),将弱引用-1。

mWeak.fetch_sub将弱引用减1，如果c != 1，说明还有弱引用，不用再进一步处理。如果等于1，就说明已经没有弱引用了，也说明了没有强引用了（一个对象的弱引用计数一定大于等于它的强引用计数），然后就需要判断是否需要释放该对象了。

```c++
void RefBase::weakref_type::decWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    impl->removeWeakRef(id);
    const int32_t c = impl->mWeak.fetch_sub(1, std::memory_order_release);
    ALOG_ASSERT(c >= 1, "decWeak called on %p too many times", this);
    if (c != 1) return;
    atomic_thread_fence(std::memory_order_acquire);

    int32_t flags = impl->mFlags.load(std::memory_order_relaxed);
    // 1、生命周期受强引用计数控制
    if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
        // This is the regular lifetime case. The object is destroyed
        // when the last strong reference goes away. Since weakref_impl
        // outlive the object, it is not destroyed in the dtor, and
        // we'll have to do it here.
        //2、没有被强引用引用过
        if (impl->mStrong.load(std::memory_order_relaxed)
                == INITIAL_STRONG_VALUE) {
            // Special case: we never had a strong reference, so we need to
            // destroy the object now.
            // 释放对象
            delete impl->mBase;
        } else {
            // ALOGV("Freeing refs %p of old RefBase %p\n", this, impl->mBase);
            // 只释放内部的引用计数器对象
            delete impl;
        }
    } else {
        // This is the OBJECT_LIFETIME_WEAK case. The last weak-reference
        // is gone, we can destroy the object.
        // 如果对象生命周期受到弱引用控制，那么当弱引用为0时，delete impl->mBase
        impl->mBase->onLastWeakRef(id);
        delete impl->mBase;
    }
}
```



## 弱指针

弱指针使用的引用计数类同样是RefBase类，实现类是wp<T>

`system/core/include/utils/RefBase.h`

```h
template <typename T>
class wp
{
public:
    typedef typename RefBase::weakref_type weakref_type;
    
    inline wp() : m_ptr(0) { }

    wp(T* other);
    wp(const wp<T>& other);
    wp(const sp<T>& other);
    template<typename U> wp(U* other);
    template<typename U> wp(const sp<U>& other);
    template<typename U> wp(const wp<U>& other);

    ~wp();
    
    // Assignment

    wp& operator = (T* other);
    wp& operator = (const wp<T>& other);
    wp& operator = (const sp<T>& other);
    
    template<typename U> wp& operator = (U* other);
    template<typename U> wp& operator = (const wp<U>& other);
    template<typename U> wp& operator = (const sp<U>& other);
    
    void set_object_and_refs(T* other, weakref_type* refs);

    // promotion to sp
    
    sp<T> promote() const;

    // Reset
    
    void clear();

    // Accessors
    
    inline  weakref_type* get_refs() const { return m_refs; }
    
    inline  T* unsafe_get() const { return m_ptr; }

    // Operators

    COMPARE_WEAK(==)
    COMPARE_WEAK(!=)
    COMPARE_WEAK(>)
    COMPARE_WEAK(<)
    COMPARE_WEAK(<=)
    COMPARE_WEAK(>=)

    inline bool operator == (const wp<T>& o) const {
        return (m_ptr == o.m_ptr) && (m_refs == o.m_refs);
    }
    template<typename U>
    inline bool operator == (const wp<U>& o) const {
        return m_ptr == o.m_ptr;
    }

    inline bool operator > (const wp<T>& o) const {
        return (m_ptr == o.m_ptr) ? (m_refs > o.m_refs) : (m_ptr > o.m_ptr);
    }
    template<typename U>
    inline bool operator > (const wp<U>& o) const {
        return (m_ptr == o.m_ptr) ? (m_refs > o.m_refs) : (m_ptr > o.m_ptr);
    }

    inline bool operator < (const wp<T>& o) const {
        return (m_ptr == o.m_ptr) ? (m_refs < o.m_refs) : (m_ptr < o.m_ptr);
    }
    template<typename U>
    inline bool operator < (const wp<U>& o) const {
        return (m_ptr == o.m_ptr) ? (m_refs < o.m_refs) : (m_ptr < o.m_ptr);
    }
                         inline bool operator != (const wp<T>& o) const { return m_refs != o.m_refs; }
    template<typename U> inline bool operator != (const wp<U>& o) const { return !operator == (o); }
                         inline bool operator <= (const wp<T>& o) const { return !operator > (o); }
    template<typename U> inline bool operator <= (const wp<U>& o) const { return !operator > (o); }
                         inline bool operator >= (const wp<T>& o) const { return !operator < (o); }
    template<typename U> inline bool operator >= (const wp<U>& o) const { return !operator < (o); }

private:
    template<typename Y> friend class sp;
    template<typename Y> friend class wp;

    T*              m_ptr;
    weakref_type*   m_refs;
};
```

wp<T>类的定义和sp<T>其实是比较接近的，最大的区别在于wp<T> 不仅保存了 T的指针 m_ptr，而且保存了T中引用计数的指针m_refs。

弱指针和强指针有一个很大的区别，弱指针不可以直接操作它所引用的对象，因为它所引用的对象可能是不受弱引用计数器控制的，即它所引用的对象有可能是一个无效的对象。实现方式就在于弱指针类没有重载*和->操作符号，而强指针重载了这两个操作符号。如果需要操作一个弱指针所引用的对象，就需要将弱指针升级为强指针。

在wp的构造函数中，会调用createWeak方法，createWeak调用incWeak增加弱引用计数，返回mRefs对象指针，构造函数中将mRefs存入m_refs中。

```h
template<typename T>
wp<T>::wp(T* other)
    : m_ptr(other)
{
    if (other) m_refs = other->createWeak(this);
}

//system/core/libutils/RefBase.cpp
RefBase::weakref_type* RefBase::createWeak(const void* id) const
{
    mRefs->incWeak(id);
    return mRefs;
}
```

wp的析构函数则调用decWeak减少引用计数，具体上面已经讲过。

接下来就是弱指针的重点，如何将弱指针升级为强指针。该过程通过调用成员函数promote实现。

```h
template<typename T>
sp<T> wp<T>::promote() const
{
    sp<T> result;
    if (m_ptr && m_refs->attemptIncStrong(&result)) {
        result.set_pointer(m_ptr);
    }
    return result;
}
```

先来看下attemptIncStrong方法

```c++
bool RefBase::weakref_type::attemptIncStrong(const void* id)
{
    incWeak(id); //增加弱引用计数一次
    
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    int32_t curCount = impl->mStrong.load(std::memory_order_relaxed); //获取当前强引用计数器的值

    ALOG_ASSERT(curCount >= 0,
            "attemptIncStrong called on %p after underflow", this);

    //如果当前已经有其他强引用，那么对象一定存在，所以就试着将强引用引用计数+1
    while (curCount > 0 && curCount != INITIAL_STRONG_VALUE) {
        // we're in the easy/common case of promoting a weak-reference
        // from an existing strong reference.
        if (impl->mStrong.compare_exchange_weak(curCount, curCount+1,
                std::memory_order_relaxed)) {
            break;
        }
        // the strong count has changed on us, we need to re-assert our
        // situation. curCount was updated by compare_exchange_weak.
    }
    
    //如果当前没有强引用的
    if (curCount <= 0 || curCount == INITIAL_STRONG_VALUE) {
        // we're now in the harder case of either:
        // - there never was a strong reference on us
        // - or, all strong references have been released
        int32_t flags = impl->mFlags.load(std::memory_order_relaxed);
        //对象生命周期受强引用控制
        if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            // this object has a "normal" life-time, i.e.: it gets destroyed
            // when the last strong reference goes away
            //有过强引用
            if (curCount <= 0) {
                // the last strong-reference got released, the object cannot
                // be revived.
                //弱引用-1作回退
                decWeak(id);
                return false;
            }

            // here, curCount == INITIAL_STRONG_VALUE, which means
            // there never was a strong-reference, so we can try to
            // promote this object; we need to do that atomically.
            // 强引用计数等于初始值，表示对象还没有强引用，自然也没有被销毁，可以转换，所以强引用+1，这里+1之后的值并不是1，而是     1<<28，所以在最后fetch_sub减掉了INITIAL_STRONG_VALUE
            while (curCount > 0) {
                if (impl->mStrong.compare_exchange_weak(curCount, curCount+1,
                        std::memory_order_relaxed)) {
                    break;
                }
                // the strong count has changed on us, we need to re-assert our
                // situation (e.g.: another thread has inc/decStrong'ed us)
                // curCount has been updated.
            }

            if (curCount <= 0) {
                // promote() failed, some other thread destroyed us in the
                // meantime (i.e.: strong count reached zero).
                decWeak(id);
                return false;
            }
        } else {
            // this object has an "extended" life-time, i.e.: it can be
            // revived from a weak-reference only.
            // Ask the object's implementation if it agrees to be revived
            // 询问是否允许升级到强指针，默认返回true表示允许，但是部分对象可能不允许做弱引用升级（重写该方法，返回false即可），如果不允许，那么久弱引用-1（回退第一步操作），直接然后返回false。如果允许弱引用升级，那么就强引用+1。
            if (!impl->mBase->onIncStrongAttempted(FIRST_INC_STRONG, id)) {
                // it didn't so give-up.
                decWeak(id);
                return false;
            }
            // grab a strong-reference, which is always safe due to the
            // extended life-time.
            curCount = impl->mStrong.fetch_add(1, std::memory_order_relaxed);
            // If the strong reference count has already been incremented by
            // someone else, the implementor of onIncStrongAttempted() is holding
            // an unneeded reference.  So call onLastStrongRef() here to remove it.
            // (No, this is not pretty.)  Note that we MUST NOT do this if we
            // are in fact acquiring the first reference.
            if (curCount != 0 && curCount != INITIAL_STRONG_VALUE) {
                impl->mBase->onLastStrongRef(id);
            }
        }
    }
    
    // 空实现
    impl->addStrongRef(id);

#if PRINT_REFS
    ALOGD("attemptIncStrong of %p from %p: cnt=%d\n", this, id, curCount);
#endif

    // curCount is the value of mStrong before we incremented it.
    // Now we need to fix-up the count if it was INITIAL_STRONG_VALUE.
    // This must be done safely, i.e.: handle the case where several threads
    // were here in attemptIncStrong().
    // curCount > INITIAL_STRONG_VALUE is OK, and can happen if we're doing
    // this in the middle of another incStrong.  The subtraction is handled
    // by the thread that started with INITIAL_STRONG_VALUE.
  
    // 如果之前没有过强引用，这里需要在强引用计数上减去1<<28。
    if (curCount == INITIAL_STRONG_VALUE) {
        impl->mStrong.fetch_sub(INITIAL_STRONG_VALUE,
                std::memory_order_relaxed);
    }

    return true;
}
```

