---
title: "Achieve Better Parallel Code with Clang Static Thread Analysis"
date: 2021-02-08T22:50:42+08:00
draft: true
slug: "clang-thread-analysis"
description:  "Achieve thread safety without headache"
categories:
    - Blog
    - Programing Languages
tags:
    - Clang
    - Static Analysis
---

In the recent two years, thanks to AMD's great job to push the industry forward,  we can see the trend that the number of cores of CPU grow rapidly, reaching 8c16t for gaming laptops and high-performance PCs, while the growth of single core performance seems to be slower. That's probably why parallelism plays such a important role in modern programming. With out it, it is impossible to make the most of the progress of hardware.   

Nevertheless, it's none of a easy job. Parallel programs are known for difficulties in debugging.  And, as an effective way to synchronization, locks are widely used, but the programs are not that easy to reason about. Surely programmers should learn well about the techniques for concurrency, but, hey, when coding a real project we can hardly spare enough effort to reason about locks while solving the problem, can we? Fortunately, clang provide a mechanism called [Thread Safety Analysis](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html) to help us out! 

## What is Clang Thread Safety Analysis

As said by the official document:   
> Clang Thread Safety Analysis is a C++ language extension which warns about potential race conditions in code. The analysis is completely static (i.e. compile-time); there is no run-time overhead.   


In a word, by applying simple attributes on classes and functions the compiler can help us detect thread safety problems without losing any runtime performance.  

## Race Condition

When working with parallelism, it matters a lot  how the access to data, for example, a variable, is controlled. Consider the following code: a simple method that should keep the invariant `x_==0` always: 

```c++
class foo
{
public:

	void bar()
	{
		while (true)
		{
			x_ = x_ + 1;
			std::this_thread::yield();
			x_ = x_ - 1;
			assert(x_ == 0);
		}
	}

private:
	int x_{ 0 };
};
```

When called with a single thread, it without doubts keeps the invariant and therefore loops forever. But when called with two threads, it crashes because the assert testing fails sometime when executing `bar()`.  

```c++
class foo
{
public:

	void bar()
	{
		while (true)
		{
			x_ = x_ + 1;
			std::this_thread::yield();
			x_ = x_ - 1;
			assert(x_ == 0);
		}
	}

private:
	int x_{ 0 };
};

int main()
{
	foo baz;

	std::thread t1(&foo::bar, &baz);
	std::thread t2(&foo::bar, &baz);

	t1.join();
	t2.join();
}
```

For those who is familiar with parallism, apparently, there's a critical section in `bar()`, which may mess things up when called by multiple threads, because two of the threads may accidentally read `x_` and do the arithmetics at the right time, when `x_=x_+1` is executed twice before one thread reaches the `x_=x_-1` and then the assertion, result in a failure. A effective way to deal with it is to protect the access to `x_` with a lock like this  :  

```c++
class foo
{
public:

	void bar()
	{
		while (true)
		{
			std::lock_guard g{ lock_ };
			x_ = x_ + 1;
			std::this_thread::yield();
			x_ = x_ - 1;
			assert(x_ == 0);
		}
	}

private:
	int x_{ 0 };
	std::mutex lock_{};
};

int main()
{
	foo baz;

	std::thread t1(&foo::bar, &baz);
	std::thread t2(&foo::bar, &baz);

	t1.join();
	t2.join();
}

```
It loops forever again, without a single failure.  
Here, it's a piece of cake to figure out which variable is protected by some certain lock, but in a real project the relationship can be extremely complicated, making it hard to figure out. Moreover, we are all human. We make mistakes. It threatens the thread safety, and as well known, such kind of bug cuased by  **race condition** is especially hard to debug, and standard way of avoiding bugs like unit testing can not effectively deal with such a problem. Can we keep them in order by some tool? Remember how we deal with the fact that man make mistakes in type? We declare variables and specified its type explicitly. Similarly, we can also declare (optionally) how access to that data is controlled in a multi-threaded environment, given that we have a proper tool. Now here come the [Thread Safety Analysis](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html) to be the very tool that we need. To use it.  


## Capabilities  

Thread safety analysis ensures the calling thread cannot access the *resource* unless it has the *capabilities* associated with C++ objects. Here, *resource* refers to things like function calling, read and write.  Different methods can acquire or release the capability. As the example in it document:  

> if `mu` is a mutex, then calling `mu.Lock()` causes the calling thread to acquire the capability to access data that is protected by `mu`. Similarly, calling `mu.Unlock()` releases that capability.  

Holding the capability is either exclusive or shared. An exclusive capability can be held by only one thread at a time, while a shared capability can be held by many threads at the same time. This mechanism enforces a multiple-reader, single-writer pattern. Write operations to protected data require exclusive access, while read operations require only shared access. A thread may hold a specific set of capabilities. By using thread annotations, we claim how those capabilities are held without caring about the underlaying mechanism used to acquire and release them, assuming that the underlaying implementation like the mutex is reliable enough to handle the handoff of capabilities.  

The compiler will approximate the real condition of capabilities held at run time as *capability environment*. It describes the set of capabilities that are statically known to be held or not held at some particular point. By analyzing these approximations, compiler give warnings about potential mistakes about capabilities.  

## Attributes

By put attributes on named declarations like classes ,  methods and data members, programmers can declare threading constraints. Those attributes affect only how compilers give you warnings. It doesn't affect the generated code and run time behaviors at all.  

As is advised in the official document, programmers are encouraged to use the macros defined in [mutex.h](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html#mutexheader)  to apply these attributes.  

There are many different attributes. Before using them, I recommend reading [the official document](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html) first. Here are some constantly-used ones.  

### Usage

To enable thread safety analysis for clang, programmers need to add a compiler option: `−Wthread−safety`. What's more, you also need to annotate your classes and data members with right attributes. You may also have fun with more features with `-Wthread-safety-negative` and `-Wthread-safety-beta` like I did in my OS kernel project [project-dionysus](https://github.com/SmartPolarBear/project-dionysus).  

### Standard Library Support
As is stated in (D14731)[https://reviews.llvm.org/D14731], libc++ has added the support for thread annotations for `std::mutex` and `std::lock_guard`, so simply adding `-Wthread-safety` may be enough. For other implementations like libstdc++, a straight-forward and brute-force way is to add a class warpper for the concerning library components.   

The following usages are selected from the official document.  

### Basic Attributes

#### GUARDED_BY and PT_GUARDED_BY

`GUARDED_BY` is an attribute on data members, which declares that the data member is protected by the given capability. Read operations on the data require shared access, while write operations require exclusive access.

`PT_GUARDED_BY` is similar, but is intended for use on pointers and smart pointers. There is no constraint on the data member itself, but the data that it points to is protected by the given capability.

```
Mutex mu;
int *p1             GUARDED_BY(mu);
int *p2             PT_GUARDED_BY(mu);
unique_ptr<int> p3  PT_GUARDED_BY(mu);

void test() {
  p1 = 0;             // Warning!

  *p2 = 42;           // Warning!
  p2 = new int;       // OK.

  *p3 = 42;           // Warning!
  p3.reset(new int);  // OK.
}
```

#### REQUIRES(…), REQUIRES_SHARED(…)

`REQUIRES` is an attribute on functions or methods, which declares that the calling thread must have exclusive access to the given capabilities. More than one capability may be specified. The capabilities must be held on entry to the function, and must still be held on exit.

`REQUIRES_SHARED` is similar, but requires only shared access.

```
Mutex mu1, mu2;
int a GUARDED_BY(mu1);
int b GUARDED_BY(mu2);

void foo() REQUIRES(mu1, mu2) {
  a = 0;
  b = 0;
}

void test() {
  mu1.Lock();
  foo();         // Warning!  Requires mu2.
  mu1.Unlock();
}
```

#### ACQUIRE(…), ACQUIRE_SHARED(…), RELEASE(…), RELEASE_SHARED(…), RELEASE_GENERIC(…)

`ACQUIRE` and `ACQUIRE_SHARED` are attributes on functions or methods declaring that the function acquires a capability, but does not release it. The given capability must not be held on entry, and will be held on exit (exclusively for ACQUIRE, shared for ACQUIRE_SHARED).

`RELEASE`, `RELEASE_SHARED`, and` RELEASE_GENERIC `declare that the function releases the given capability. The capability must be held on entry (exclusively for RELEASE, shared for RELEASE_SHARED, exclusively or shared for RELEASE_GENERIC), and will no longer be held on exit.

```
Mutex mu;
MyClass myObject GUARDED_BY(mu);

void lockAndInit() ACQUIRE(mu) {
  mu.Lock();
  myObject.init();
}

void cleanupAndUnlock() RELEASE(mu) {
  myObject.cleanup();
}                          // Warning!  Need to unlock mu.

void test() {
  lockAndInit();
  myObject.doSomething();
  cleanupAndUnlock();
  myObject.doSomething();  // Warning, mu is not locked.
}
```

If no argument is passed to ACQUIRE or RELEASE, then the argument is assumed to be this, and the analysis will not check the body of the function. This pattern is intended for use by classes which hide locking details behind an abstract interface. For example:

```
template <class T>
class CAPABILITY("mutex") Container {
private:
  Mutex mu;
  T* data;

public:
  // Hide mu from public interface.
  void Lock()   ACQUIRE() { mu.Lock(); }
  void Unlock() RELEASE() { mu.Unlock(); }

  T& getElem(int i) { return data[i]; }
};

void test() {
  Container<int> c;
  c.Lock();
  int i = c.getElem(0);
  c.Unlock();
}

```

#### EXCLUDES(…)

`EXCLUDES` is an attribute on functions or methods, which declares that the caller must not hold the given capabilities. This annotation is used to prevent deadlock. Many mutex implementations are not re-entrant, so deadlock can occur if the function acquires the mutex a second time.

```
Mutex mu;
int a GUARDED_BY(mu);

void clear() EXCLUDES(mu) {
  mu.Lock();
  a = 0;
  mu.Unlock();
}

void reset() {
  mu.Lock();
  clear();     // Warning!  Caller cannot hold 'mu'.
  mu.Unlock();
}

```

Unlike `REQUIRES`,`EXCLUDES` is optional. The analysis will not issue a warning if the attribute is missing, which can lead to false negatives in some cases. This issue is discussed further in Negative Capabilities.

#### RETURN_CAPABILITY(c)

`RETURN_CAPABILITY` is an attribute on functions or methods, which declares that the function returns a reference to the given capability. It is used to annotate getter methods that return mutexes.

```
class MyClass {
private:
  Mutex mu;
  int a GUARDED_BY(mu);

public:
  Mutex* getMu() RETURN_CAPABILITY(mu) { return &mu; }

  // analysis knows that getMu() == mu
  void clear() REQUIRES(getMu()) { a = 0; }
};

```

#### CAPABILITY(string)

`CAPABILITY` is an attribute on classes, which specifies that objects of the class can be used as a capability. The string argument specifies the kind of capability in error messages, e.g. "mutex". See the Container example given above, or the Mutex class in [[mutex.h](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html#mutexheader)](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html#mutexheader).   


#### TRY_ACQUIRE(bool, …), TRY_ACQUIRE_SHARED(bool, …)

These are attributes on a function or method that tries to acquire the given capability, and returns a boolean value indicating success or failure. The first argument must be true or false, to specify which return value indicates success, and the remaining arguments are interpreted in the same way as `ACQUIRE`. See [[mutex.h](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html#mutexheader)](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html#mutexheader), below, for example uses.

Because the analysis doesn’t support conditional locking, a capability is treated as acquired after the first branch on the return value of a try-acquire function.

```
Mutex mu;
int a GUARDED_BY(mu);

void foo() {
  bool success = mu.TryLock();
  a = 0;         // Warning, mu is not locked.
  if (success) {
    a = 0;       // Ok.
    mu.Unlock();
  } else {
    a = 0;       // Warning, mu is not locked.
  }
}
```

#### ASSERT_CAPABILITY(…) and ASSERT_SHARED_CAPABILITY(…)

These are attributes on a function or method which asserts the calling thread already holds the given capability, for example by performing a run-time test and terminating if the capability is not held. Presence of this annotation causes the analysis to assume the capability is held after calls to the annotated function. See [mutex.h](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html#mutexheader), below, for example uses.

### RAII

#### SCOPED_CAPABILITY

`SCOPED_CAPABILITY` is an attribute on classes that implement RAII-style locking, in which a capability is acquired in the constructor, and released in the destructor. Such classes require special handling because the constructor and destructor refer to the capability via different names; see the MutexLocker class in [mutex.h](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html#mutexheader), below.

Scoped capabilities are treated as capabilities that are implicitly acquired on construction and released on destruction. They are associated with the set of (regular) capabilities named in thread safety attributes on the constructor. Acquire-type attributes on other member functions are treated as applying to that set of associated capabilities, while `RELEASE` implies that a function releases all associated capabilities in whatever mode they’re held.

### Disabling in Code

#### NO_THREAD_SAFETY_ANALYSIS
`NO_THREAD_SAFETY_ANALYSIS` is an attribute on functions or methods, which turns off thread safety checking for that method. It provides an escape hatch for functions which are either (1) deliberately thread-unsafe, or (2) are thread-safe, but too complicated for the analysis to understand. Reasons for (2) will be described in the Known Limitations, below.

class Counter {
  Mutex mu;
  int a GUARDED_BY(mu);

  void unsafeIncrement() NO_THREAD_SAFETY_ANALYSIS { a++; }
};
Unlike the other attributes, `NO_THREAD_SAFETY_ANALYSIS` is not part of the interface of a function, and should thus be placed on the function definition (in the .cc or .cpp file) rather than on the function declaration (in the header).


## Epilogue  

As a great static analysis tool, clang thread safety analysis save our time by preventing undesirable access to data members. It simplify the process of designing, coding and debugging if you don't disable or ignore the warnings. Though when adding them to existing project, the warnings are somehow annoying.  

## Further Reading

1. [Thread Safety Analysis](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html)  
Official document which is worth a carful reading before getting started.  

2. [C/C++ Thread Safety Analysis](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/42958.pdf)  

Google's paper about thread safty analysis.  

