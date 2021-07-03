---
title: Android系统-PMS scanPackageDirtyLI的LI是什么意思
date: 2021-05-13 19:22:36
categories: 
- Android系统
tags:
- Android
- PMS
---

如标题，这个问题是最近在面试的时候被问到的，当我听到这个问题的时候脑子里确实是一点概念都没有，平时看源码流程从一个方法跳到另一个方法，但是从来没有考虑过这个函数名称为什么是这个样子的，考虑的太少了。面试结束后回来就立马查一下这个问题，在这里也记录一下。

在PMS的源码中除了LI结尾的方法，还有其他例如**LIF、LPw、LPr**结尾的函数，下面列举了一部分。

| 后缀 |                            方法名                            |
| :--: | :----------------------------------------------------------: |
|  LI  | collectCertificatesLI()<br/>installPackageLI()<br/>scanPackageLI()<br/>scanDirLI() |
| LIF  | deletePackageLIF()<br/>deleteSystemPackageLIF()<br/>clearApplicationUserDataLIF()<br/>clearAppDataLIF() |
|  LP  | grantPermissionsLPw()<br/>updatePermissionsLPw()<br/>enableSystemPackageLPw()<br/>setInstallerPackageNameLPw() |
| LPr  | getRequiredInstallerLPr()<br/>verifyPackageUpdateLPr()<br/>normalizePackageNameLPr()<br/>needsNetworkVerificationLPr() |

首先看一下PMS源码中的注释

```java
/**
 * Keep track of all those APKs everywhere.
 * <p>
 * // 内部有两个重要的锁
 * Internally there are two important locks:
 * <ul>
 *  mPackages 用于保护所有内存中解析的包细节和其他相关状态。它是一个细粒度的锁，只应暂时保持，因为它是系统中最有竞争力的锁之一
 * <li>{@link #mPackages} is used to guard all in-memory parsed package details
 * and other related state. It is a fine-grained lock that should only be held
 * momentarily, as it's one of the most contended locks in the system.
 
 * mInstallLock用于保护所有installd访问,其操作通常涉及到磁盘上的应用程序数据的重载,installd是单线程的，它的操作通常很慢,当已经持有mPackages时，不应获取此锁。相反，在已经持有mInstallLock的情况下，暂时获取mPackages是安全的。
 
 * <li>{@link #mInstallLock} is used to guard all {@code installd} access, whose
 * operations typically involve heavy lifting of application data on disk. Since
 * {@code installd} is single-threaded, and it's operations can often be slow,
 * this lock should never be acquired while already holding {@link #mPackages}.
 * Conversely, it's safe to acquire {@link #mPackages} momentarily while already
 * holding {@link #mInstallLock}.
 * </ul>
 * Many internal methods rely on the caller to hold the appropriate locks, and
 * this contract is expressed through method name suffixes:
 * <ul>
 * <li>fooLI(): the caller must hold {@link #mInstallLock}
 * <li>fooLIF(): the caller must hold {@link #mInstallLock} and the package
 * being modified must be frozen
 * <li>fooLPr(): the caller must hold {@link #mPackages} for reading
 * <li>fooLPw(): the caller must hold {@link #mPackages} for writing
 * </ul>
 * <p>
 * Because this class is very central to the platform's security; please run all
 * CTS and unit tests whenever making modifications:
 *
 * <pre>
 * $ runtest -c android.content.pm.PackageManagerTests frameworks-core
 * $ cts-tradefed run commandAndExit cts -m AppSecurityTests
 * </pre>
 */
```

从上面的注释中可以了解到PMS中有两个锁，**mPackages**和**mInstallLock**，这两个锁在PMS中非常重要。而LI、LIF、LPw、LPr这些方法，L指的是Lock，I和P就指的是mInstallLock和mPackages。最后面w表示writing，r表示reading，F表示Freeze。

在上面注释中也可以看到mInstallLock是单线程的，操作慢，所以在已经持有`mPackages`同步锁的时候，千万不要再请求`mInstallLock`同步锁。

**错误方式**

```java
synchronized (mPackages) {
synchronized (mInstaller) {
    // 这种情况是绝对不允许的。因为Install的处理时间会很长，导致对mPackages锁住的时间加长，会使得其他对mPackages操作的请求处于长时间等待。
}
}
```

**正确方式**

```java
synchronized (mInstaller) {
synchronized (mPackages) {
    // 这种情况是允许的。因为mPackages处理完之后，其他对mPackages操作的请求可以对mPackages处理，不需要等待太久。
    // 由于处理Install的时间本身很长，synchronized (mPackages)又较快，所以不会对原本长时间持有mInstaller锁的情况有大的影响。
}
}
```

许多内部方法依赖于调用者来持有适当的锁，在调用函数时，要遵循以下规则

|    方法名    |                           使用方式                           |
| :----------: | :----------------------------------------------------------: |
| **fooLI()**  |          调用fooLI()，必须先持有`mInstallLock`锁。           |
| **fooLIF()** | 调用fooLIF()，必须先持有`mInstallLock`锁，并且正在被修改的包（package）必须被冻结（be frozen）。 |
| **fooLPr()** | 调用fooLPr()，必须先持有`mPackages`锁，并且只用于**读**操作。 |
| **fooLPw()** | 调用fooLPw()，必须先持有`mPackages`锁，并且只用于**写**操作。 |

例如要调用`installPackageTracedLI`，必须先持有`mInstallLock`锁，因此，调用为：

```java
synchronized (mInstallLock) {
         installPackageTracedLI(args, res);
}
```

再例如`verifyPackageUpdateLPr`

```java
// writer
synchronized (mPackages) {
     ......
		if (!verifyPackageUpdateLPr(origPackage, pkg)) {
         // New package is not compatible with original.
         origPackage = null;
         continue;
    }
    ......
}
```

最后还有一个注解**`@GuardedBy`**用于标记哪些变量要用同步锁保护起来。

```java
    @GuardedBy("mPackages")
    private boolean mDexOptDialogShown;

    // Used for privilege escalation. MUST NOT BE CALLED WITH mPackages
    // LOCK HELD.  Can be called with mInstallLock held.
    //用于权限提升。不能在保持mPackages锁的情况下调用。可以在保持mInstallLock的情况下调用。
    @GuardedBy("mInstallLock")
    final Installer mInstaller;

    @GuardedBy("mPackages")
    final ArrayMap<String, PackageParser.Package> mPackages =
            new ArrayMap<String, PackageParser.Package>();
```

这个注解表示在使用变量的地方，必须获取对应的锁，以防止并发错误

```java
synchronized (mPackages) {
    dexOptDialogShown = mDexOptDialogShown;
}
```

嗯，又学习到了一个知识点。

