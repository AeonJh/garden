---
title: "std::enable_shared_from_this 的简单剖析"
date: 2024-09-22
tags: ["cpp", "memory-management", "smart-pointers", "programming"]
---

> 引言： 当涉及到 C++ 中的内存管理时，`std::shared_ptr` 是一个强大的工具，用于实现共享所有权的智能指针。然而，在某些情况下，我们可能需要从类的成员函数中获取指向该对象的 `std::shared_ptr`。这就是 `std::enable_shared_from_this` 派上用场的时候。本文将深入探讨 `std::enable_shared_from_this` 的工作原理、实现方式以及在实际应用中的最佳实践。

## 什么是 std::enable_shared_from_this?

`std::enable_shared_from_this` 是C++11引入的一个标准库模板类，位于 `<memory>` 头文件中。它允许一个类的成员函数安全地创建指向该对象的 `std::shared_ptr`，前提是该对象已经由至少一个 `std::shared_ptr` 管理。它旨在解决从 `this` 指针创建 `std::shared_ptr` 时可能出现的悬挂指针和双重删除问题。

### 问题场景

让我们看看下面的代码片段，了解如果不使用 `std::enable_shared_from_this` 会发生什么：

```cpp {4}
class MyClass {
public:
    std::shared_ptr<MyClass> getPtr() {
        return std::shared_ptr<MyClass>(this); // WARN: DANGEROUS!
    }
};

int main() {
    auto ptr1 = std::make_shared<MyClass>();
    auto ptr2 = ptr1->getPtr(); // Creates independent control block
} // ERROR: ptr1 and ptr2 don't share ownership - double deletion will occur!
```

上面的代码中，`getPtr` 方法试图从 `this` 指针创建一个新的 `std::shared_ptr`。让我们来看看[llvm源码](https://github.com/llvm/llvm-project/blob/main/libcxx/include/__memory/shared_ptr.h#L333-L351):

```cpp {16}
  template <class _Yp,
            __enable_if_t< _And< __raw_pointer_compatible_with<_Yp, _Tp>
  // In C++03 we get errors when trying to do SFINAE with the
  // delete operator, so we always pretend that it's deletable.
  // The same happens on GCC.
#if !defined(_LIBCPP_CXX03_LANG) && !defined(_LIBCPP_COMPILER_GCC)
                                 ,
                                 _If<is_array<_Tp>::value, __is_array_deletable<_Yp*>, __is_deletable<_Yp*> >
#endif
                                 >::value,
                           int> = 0>
  _LIBCPP_HIDE_FROM_ABI explicit shared_ptr(_Yp* __p) : __ptr_(__p) {
    unique_ptr<_Yp> __hold(__p);
    typedef typename __shared_ptr_default_allocator<_Yp>::type _AllocT;
    typedef __shared_ptr_pointer<_Yp*, __shared_ptr_default_delete<_Tp, _Yp>, _AllocT> _CntrlBlk;
    __cntrl_ = new _CntrlBlk(__p, __shared_ptr_default_delete<_Tp, _Yp>(), _AllocT()); // NOTE: Create new control block
    __hold.release();
    __enable_weak_this(__p, __p);
  }
```

可以看到，`std::shared_ptr<Myclass>(this)` 会调用此构造函数创建一个新的控制块来管理传入的裸指针 `this`。这意味着 `ptr1` 和 `ptr2` 拥有不同的控制块，导致它们在析构时都会尝试删除同一个对象，从而引发未定义行为。

## 使用 std::enable_shared_from_this 解决问题

找到了问题所在，接下来我们看看如何使用 `std::enable_shared_from_this` 来解决这个问题。

### Step 1: 从 std::enable_shared_from_this 继承

```cpp
#include <memory>

class MyClass : public std::enable_shared_from_this<MyClass> {
public:
    MyClass(int value) : data(value) {}

    // Safe way to get shared_ptr to this object
    std::shared_ptr<MyClass> getPtr() {
        return shared_from_this();
    }

    void doSomething() {
        // Can safely get a shared_ptr to itself
        auto self = shared_from_this();
        // Use self for async operations, callbacks, etc.
    }
};
```

### Step 2: 始终使用 std::make_shared 来创建对象

```cpp
int main() {
    // Correct usage - object must be created with shared_ptr first
    auto ptr1 = std::make_shared<MyClass>();
    auto ptr2 = ptr1->getPtr(); // Now they share the same control block

    std::cout << "ptr1 use count: " << ptr1.use_count() << std::endl; // 2
    std::cout << "ptr2 use count: " << ptr2.use_count() << std::endl; // 2

    return 0;
}
```

### Step 3: 正确处理异常

```cpp
class SafeClass : public std::enable_shared_from_this<SafeClass> {
public:
    std::shared_ptr<SafeClass> getSelf() {
        try {
            return shared_from_this();
        } catch (const std::bad_weak_ptr& e) {
            // Object not managed by shared_ptr yet
            throw std::runtime_error("Object not managed by shared_ptr");
        }
    }
};
```

## 源码剖析

> 使用 llvm 的 libc++ 实现作为参考

为什么继承 `std::enable_shared_from_this` 并使用 `shared_from_this()` 可以解决问题？这是因为 `std::enable_shared_from_this` 内部维护了一个弱引用（`std::weak_ptr`）指向管理该对象的 `std::shared_ptr`。
当调用 `shared_from_this()` 时，它会检查这个弱引用是否有效，并返回一个新的 `std::shared_ptr`，共享相同的控制块。让我们看下这部分的[源码](https://github.com/llvm/llvm-project/blob/main/libcxx/include/__memory/shared_ptr.h#L1437-L1459):

```cpp {3,12-13}
template <class _Tp>
class enable_shared_from_this {
  mutable weak_ptr<_Tp> __weak_this_; // Maintains a weak reference to the shared_ptr managing this object

protected:
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR enable_shared_from_this() _NOEXCEPT {}
  _LIBCPP_HIDE_FROM_ABI enable_shared_from_this(enable_shared_from_this const&) _NOEXCEPT {}
  _LIBCPP_HIDE_FROM_ABI enable_shared_from_this& operator=(enable_shared_from_this const&) _NOEXCEPT { return *this; }
  _LIBCPP_HIDE_FROM_ABI ~enable_shared_from_this() {}

public:
  _LIBCPP_HIDE_FROM_ABI shared_ptr<_Tp> shared_from_this() { return shared_ptr<_Tp>(__weak_this_); } // Returns a shared_ptr sharing ownership of *this
  _LIBCPP_HIDE_FROM_ABI shared_ptr<_Tp const> shared_from_this() const { return shared_ptr<const _Tp>(__weak_this_); }

#if _LIBCPP_STD_VER >= 17
  _LIBCPP_HIDE_FROM_ABI weak_ptr<_Tp> weak_from_this() _NOEXCEPT { return __weak_this_; }

  _LIBCPP_HIDE_FROM_ABI weak_ptr<const _Tp> weak_from_this() const _NOEXCEPT { return __weak_this_; }
#endif // _LIBCPP_STD_VER >= 17

  template <class _Up>
  friend class shared_ptr;
};

```

继续往下看 `shared_ptr<_Tp>(__weak_this_)` 的[调用](https://github.com/llvm/llvm-project/blob/main/libcxx/include/__memory/shared_ptr.h#L498-L503)：

```cpp {3}
  template <class _Yp, __enable_if_t<__compatible_with<_Yp, _Tp>::value, int> = 0>
  _LIBCPP_HIDE_FROM_ABI explicit shared_ptr(const weak_ptr<_Yp>& __r)
      : __ptr_(__r.__ptr_), __cntrl_(__r.__cntrl_ ? __r.__cntrl_->lock() : __r.__cntrl_) {
    if (__cntrl_ == nullptr)
      std::__throw_bad_weak_ptr();
  }
```

可以看到，`shared_ptr` 的构造函数接受一个 `weak_ptr`，并尝试锁定它以获取一个共享的控制块。如果成功，它将返回一个新的 `shared_ptr`（使用共享控制块而不是创建一个新的控制块）, 否则会抛出 `std::bad_weak_ptr` 异常。

## 应用场景

### 1. 异步操作

非常适合需要在异步回调中保持对象存活的场景：

```cpp
class AsyncProcessor : public std::enable_shared_from_this<AsyncProcessor> {
public:
    void startProcessing() {
        // Keep object alive during async operation
        auto self = shared_from_this();

        std::thread([self]() {
            std::this_thread::sleep_for(std::chrono::seconds(1));
            self->onProcessingComplete();
        }).detach();
    }

private:
    void onProcessingComplete() {
        std::cout << "Processing completed!" << std::endl;
    }
};
```

### 2. 观察者模式实现

```cpp
class Subject;

class Observer : public std::enable_shared_from_this<Observer> {
public:
    void registerWith(std::shared_ptr<Subject> subject) {
        // Pass shared_ptr to self for registration
        subject->addObserver(shared_from_this());
    }

    virtual void notify(const std::string& message) = 0;
};

class Subject {
    std::vector<std::weak_ptr<Observer>> observers;

public:
    void addObserver(std::shared_ptr<Observer> obs) {
        observers.push_back(obs); // Store as weak_ptr to avoid cycles
    }

    void notifyAll(const std::string& message) {
        for (auto it = observers.begin(); it != observers.end();) {
            if (auto obs = it->lock()) {
                obs->notify(message);
                ++it;
            } else {
                it = observers.erase(it); // Remove expired observers
            }
        }
    }
};
```

### 3. 带有父引用的树/图结构

```cpp
class TreeNode : public std::enable_shared_from_this<TreeNode> {
public:
    void addChild(std::shared_ptr<TreeNode> child) {
        child->parent = weak_from_this(); // Use weak_ptr to avoid cycles
        children.push_back(child);
    }

    std::shared_ptr<TreeNode> getParent() {
        return parent.lock(); // Convert weak_ptr to shared_ptr
    }

    void removeFromParent() {
        if (auto p = parent.lock()) {
            auto& siblings = p->children;
            auto self = shared_from_this();
            siblings.erase(
                std::remove(siblings.begin(), siblings.end(), self),
                siblings.end()
            );
        }
    }

private:
    std::weak_ptr<TreeNode> parent;
    std::vector<std::shared_ptr<TreeNode>> children;
};
```

## 最佳实践和注意事项

### Do's:
- 始终使用 `std::make_shared` 或 `std::shared_ptr` 构造函数来创建对象
- 当需要一个 `std::weak_ptr` 来避免循环时，使用 `weak_from_this()`（C++17及以上）
- 处理调用 `shared_from_this()` 时抛出的 `std::bad_weak_ptr` 异常
- 优先使用组合而不是继承来减少复杂性

### Don'ts:
- 永远不要在构造函数或析构函数中调用 `shared_from_this()`
- 不要直接使用裸指针 `this` 来创建 `std::shared_ptr`
- 避免在对象由 `std::shared_ptr` 管理之前调用 `shared_from_this()`

### 不当示例:

```cpp
class BadExample : public std::enable_shared_from_this<BadExample> {
public:
    BadExample() {
        // WRONG! Will throw std::bad_weak_ptr
        // auto self = shared_from_this();
    }

    static std::shared_ptr<BadExample> createBroken() {
        BadExample* obj = new BadExample();
        // WRONG! Call shared_from_this() before managed by shared_ptr
        // return obj->shared_from_this();
        return std::shared_ptr<BadExample>(obj);
    }
};
```

## 结论

`std::enable_shared_from_this` 是现代 C++ 在处理 `std::shared_ptr` 时的一个重要工具。它使得在异步编程、观察者模式和复杂对象关系等场景中实现安全的自我引用成为可能。 请记住，
始终先使用 `std::make_shared` 创建对象，正确处理异常，并在适当的时候使用 `std::weak_ptr` 以防循环依赖。
