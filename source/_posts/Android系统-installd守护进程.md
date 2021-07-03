---
title: Android系统-installd守护进程
date: 2021-04-27 18:48:39
categories: 
- Android系统
tags:
- installd
- 守护进程
---

从前面[Android包管理机制(三)PMS处理APK安装](https://david1840.github.io/2021/03/24/Android%E5%8C%85%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6-%E4%B8%89-PMS%E5%A4%84%E7%90%86APK%E5%AE%89%E8%A3%85/)中可以了解到PMS在安装APK的时候调用了Installer的函数，而Installer实际是调用了installd中的方法。今天我们就来了解下守护进程installd，看看它做了什么，以及为什么需要有一个守护进程？

## installd启动

`frameworks/native/cmds/installd/installd.rc`

```c
service installd /system/bin/installd
    class main
    socket installd stream 600 system system
```

installd将作为service被init进程启动，同时会创建一个名为installd的socket

`frameworks/native/cmds/installd/installd.cpp`

```c
int main(const int argc, char *argv[]) {
    return android::installd::installd_main(argc, argv);
}

static int installd_main(const int argc ATTRIBUTE_UNUSED, char *argv[]) {
    char buf[BUFFER_MAX];
    struct sockaddr addr;
    socklen_t alen;
    int lsocket, s;
    int selinux_enabled = (is_selinux_enabled() > 0);

    setenv("ANDROID_LOG_TAGS", "*:v", 1);
    android::base::InitLogging(argv);

    ALOGI("installd firing up\n");

    union selinux_callback cb;
    cb.func_log = log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);

    if (!initialize_globals()) {
        ALOGE("Could not initialize globals; exiting.\n");
        exit(1);
    }

    if (initialize_directories() < 0) {
        ALOGE("Could not create directories; exiting.\n");
        exit(1);
    }

    if (selinux_enabled && selinux_status_open(true) < 0) {
        ALOGE("Could not open selinux status; exiting.\n");
        exit(1);
    }

    //得到"installd" socket
    lsocket = android_get_control_socket(SOCKET_PATH);
    if (lsocket < 0) {
        ALOGE("Failed to get socket from environment: %s\n", strerror(errno));
        exit(1);
    }
    //socket监听
    if (listen(lsocket, 5)) {
        ALOGE("Listen on socket failed: %s\n", strerror(errno));
        exit(1);
    }
    fcntl(lsocket, F_SETFD, FD_CLOEXEC);

    for (;;) {
        alen = sizeof(addr);
        // 接收信息
        s = accept(lsocket, &addr, &alen);
        if (s < 0) {
            ALOGE("Accept failed: %s\n", strerror(errno));
            continue;
        }
        fcntl(s, F_SETFD, FD_CLOEXEC);

        ALOGI("new connection\n");
        for (;;) {
            unsigned short count;
            if (readx(s, &count, sizeof(count))) {
                ALOGE("failed to read size\n");
                break;
            }
            if ((count < 1) || (count >= BUFFER_MAX)) {
                ALOGE("invalid size %d\n", count);
                break;
            }
            if (readx(s, buf, count)) {
                ALOGE("failed to read command\n");
                break;
            }
            buf[count] = 0;
            if (selinux_enabled && selinux_status_updated() > 0) {
                selinux_android_seapp_context_reload();
            }
            //执行命令
            if (execute(s, buf)) break;
        }
        ALOGI("closing connection\n");
        close(s);
    }

    return 0;
}
```

上面的代码也比较简单，启动后获取installd socket，监听连接，获取客户端发送的命令并执行。



## 执行命令

```c
static int execute(int s, char cmd[BUFFER_MAX])
{
    char reply[REPLY_MAX];
    char *arg[TOKEN_MAX+1];
    unsigned i;
    unsigned n = 0;
    unsigned short count;
    int ret = -1;

        /* default reply is "" */
    reply[0] = 0;

        /* n is number of args (not counting arg[0]) */
    arg[0] = cmd;
    //解析参数的个数
    while (*cmd) {
         //当发现空格时
        if (isspace(*cmd)) {
            *cmd++ = 0;
            //参数个数+1
            n++;
            //保存参数
            arg[n] = cmd;
            if (n == TOKEN_MAX) {
                ALOGE("too many arguments\n");
                goto done;
            }
        }
        if (*cmd) {
          cmd++;
        }
    }

    //cmds是一个数组，保存了所有可以执行的操作
    for (i = 0; i < sizeof(cmds) / sizeof(cmds[0]); i++) {
        if (!strcmp(cmds[i].name,arg[0])) {
            if (n != cmds[i].numargs) {
                ALOGE("%s requires %d arguments (%d given)\n",
                     cmds[i].name, cmds[i].numargs, n);
            } else {
                //参数正确时，调用对应的执行函数进行处理
                ret = cmds[i].func(arg + 1, reply);
            }
            goto done;
        }
    }
    ALOGE("unsupported command '%s'\n", arg[0]);
    return 0;
}
```

execute就是在接收到命令后在cmds中查找对应的指令，然后执行。

```c
struct cmdinfo cmds[] = {
    { "ping",                 0, do_ping },

    { "create_app_data",      7, do_create_app_data },
    { "restorecon_app_data",  6, do_restorecon_app_data },
    { "migrate_app_data",     4, do_migrate_app_data },
    { "clear_app_data",       5, do_clear_app_data },
    { "destroy_app_data",     5, do_destroy_app_data },
    { "move_complete_app",    7, do_move_complete_app },
    { "get_app_size",         6, do_get_app_size },
    { "get_app_data_inode",   4, do_get_app_data_inode },

    { "create_user_data",     4, do_create_user_data },
    { "destroy_user_data",    3, do_destroy_user_data },

    { "dexopt",              10, do_dexopt },
    { "markbootcomplete",     1, do_mark_boot_complete },
    { "rmdex",                2, do_rm_dex },
    { "freecache",            2, do_free_cache },
    { "linklib",              4, do_linklib },
    { "idmap",                3, do_idmap },
    { "createoatdir",         2, do_create_oat_dir },
    { "rmpackagedir",         1, do_rm_package_dir },
    { "clear_app_profiles",   1, do_clear_app_profiles },
    { "destroy_app_profiles", 1, do_destroy_app_profiles },
    { "linkfile",             3, do_link_file },
    { "move_ab",              3, do_move_ab },
    { "merge_profiles",       2, do_merge_profiles },
    { "dump_profiles",        3, do_dump_profiles },
    { "delete_odex",          3, do_delete_odex },
};
```

这个就是cmds数组，第一列是指令名称，第二列是要求的参数个数，第三列是指令对应的执行函数。

以create_app_data为例

`frameworks/native/cmds/installd/installd.cpp`

```c
static int do_create_app_data(char **arg, char reply[REPLY_MAX] ATTRIBUTE_UNUSED) {
    /* const char *uuid, const char *pkgname, userid_t userid, int flags,
            appid_t appid, const char* seinfo, int target_sdk_version */
    return create_app_data(parse_null(arg[0]), arg[1], atoi(arg[2]), atoi(arg[3]),
                           atoi(arg[4]), arg[5], atoi(arg[6]));
}
```

`frameworks/native/cmds/installd/commands.cpp`

```c
int create_app_data(const char *uuid, const char *pkgname, userid_t userid, int flags,
        appid_t appid, const char* seinfo, int target_sdk_version) {
    uid_t uid = multiuser_get_uid(userid, appid);
    mode_t target_mode = target_sdk_version >= MIN_RESTRICTED_HOME_SDK_VERSION ? 0700 : 0751;
    if (flags & FLAG_STORAGE_CE) {
        auto path = create_data_user_ce_package_path(uuid, userid, pkgname);
        if (prepare_app_dir(path, target_mode, uid) ||
                prepare_app_dir(path, "cache", 0771, uid) ||
                prepare_app_dir(path, "code_cache", 0771, uid)) {
            return -1;
        }

        // Consider restorecon over contents if label changed
        if (restorecon_app_data_lazy(path, seinfo, uid) ||
                restorecon_app_data_lazy(path, "cache", seinfo, uid) ||
                restorecon_app_data_lazy(path, "code_cache", seinfo, uid)) {
            return -1;
        }

        // Remember inode numbers of cache directories so that we can clear
        // contents while CE storage is locked
        if (write_path_inode(path, "cache", kXattrInodeCache) ||
                write_path_inode(path, "code_cache", kXattrInodeCodeCache)) {
            return -1;
        }
    }
    if (flags & FLAG_STORAGE_DE) {
        auto path = create_data_user_de_package_path(uuid, userid, pkgname);
        if (prepare_app_dir(path, target_mode, uid)) {
            // TODO: include result once 25796509 is fixed
            return 0;
        }

        // Consider restorecon over contents if label changed
        if (restorecon_app_data_lazy(path, seinfo, uid)) {
            return -1;
        }

        if (property_get_bool("dalvik.vm.usejitprofiles")) {
            const std::string profile_path = create_data_user_profile_package_path(userid, pkgname);
            // read-write-execute only for the app user.
            if (fs_prepare_dir_strict(profile_path.c_str(), 0700, uid, uid) != 0) {
                PLOG(ERROR) << "Failed to prepare " << profile_path;
                return -1;
            }
            std::string profile_file = create_primary_profile(profile_path);
            // read-write only for the app user.
            if (fs_prepare_file_strict(profile_file.c_str(), 0600, uid, uid) != 0) {
                PLOG(ERROR) << "Failed to prepare " << profile_path;
                return -1;
            }
            const std::string ref_profile_path = create_data_ref_profile_package_path(pkgname);
            // dex2oat/profman runs under the shared app gid and it needs to read/write reference
            // profiles.
            appid_t shared_app_gid = multiuser_get_shared_app_gid(uid);
            if (fs_prepare_dir_strict(
                    ref_profile_path.c_str(), 0700, shared_app_gid, shared_app_gid) != 0) {
                PLOG(ERROR) << "Failed to prepare " << ref_profile_path;
                return -1;
            }
        }
    }
    return 0;
}
```

创建cache和code_cache文件夹，并授予相应的权限。

## Installer服务

installd在Framework中对应的是Installer服务。

`frameworks/base/services/core/java/com/android/server/pm/Installer.java`

```java
public final class Installer extends SystemService
```

它是一个SystemService，在系统启动的时候作为核心服务在SystemServer中启动。

```java
 private void startBootstrapServices() {
        //启动installer服务
        Installer installer = mSystemServiceManager.startService(Installer.class);
        ......
 }
```

Install中部分代码

```java
    public Installer(Context context) {
        super(context);
        mInstaller = new InstallerConnection();
    }

    // Package-private installer that accepts a custom InstallerConnection. Used for
    // OtaDexoptService.
    Installer(Context context, InstallerConnection connection) {
        super(context);
        mInstaller = connection;
    }

    /**
     * Yell loudly if someone tries making future calls while holding a lock on
     * the given object.
     */
    public void setWarnIfHeld(Object warnIfHeld) {
        mInstaller.setWarnIfHeld(warnIfHeld);
    }

    @Override
    public void onStart() {
        Slog.i(TAG, "Waiting for installd to be ready.");
        mInstaller.waitForConnection();
    }

    public void createAppData(String uuid, String pkgname, int userid, int flags, int appid,
            String seinfo, int targetSdkVersion) throws InstallerException {
        mInstaller.execute("create_app_data", uuid, pkgname, userid, flags, appid, seinfo,
            targetSdkVersion);
    }
```

可以看到，Installer在被创建的时候InstallerConnection也被创建了一个实例，createAppData实际调用也是InstallerConnection中的execute方法。所以Installer服务实际是使用InstallerConnection在做具体操作。

`frameworks/base/core/java/com/android/internal/os/InstallerConnection.java`

```java
public class InstallerConnection {

    public synchronized String transact(String cmd) {
        if (mWarnIfHeld != null && Thread.holdsLock(mWarnIfHeld)) {
            Slog.wtf(TAG, "Calling thread " + Thread.currentThread().getName() + " is holding 0x"
                    + Integer.toHexString(System.identityHashCode(mWarnIfHeld)), new Throwable());
        }

        if (!connect()) {
            Slog.e(TAG, "connection failed");
            return "-1";
        }

        if (!writeCommand(cmd)) {
            Slog.e(TAG, "write command failed? reconnect!");
            if (!connect() || !writeCommand(cmd)) {
                return "-1";
            }
        }
    }

    public String[] execute(String cmd, Object... args) throws InstallerException {
     
        final String[] resRaw = transact(builder.toString()).split(" ");
        int res = -1;
        try {
            res = Integer.parseInt(resRaw[0]);
        } catch (ArrayIndexOutOfBoundsException | NumberFormatException ignored) {
        }
       
        return resRaw;
    }

    public void dexopt(String apkPath, int uid, String instructionSet, int dexoptNeeded,
            int dexFlags, String compilerFilter, String volumeUuid, String sharedLibraries)
            throws InstallerException {
        dexopt(apkPath, uid, "*", instructionSet, dexoptNeeded, null /*outputPath*/, dexFlags,
                compilerFilter, volumeUuid, sharedLibraries);
    }

    public void dexopt(String apkPath, int uid, String pkgName, String instructionSet,
            int dexoptNeeded, String outputPath, int dexFlags, String compilerFilter,
            String volumeUuid, String sharedLibraries) throws InstallerException {
 execute("dexopt",apkPath,uid,pkgName,instructionSet,dexoptNeeded,outputPath,dexFlags,compilerFilter,volumeUuid,sharedLibraries);
    }

  
    private boolean connect() {
        if (mSocket != null) {
            return true;
        }
        Slog.i(TAG, "connecting...");
        try {
            mSocket = new LocalSocket();

            LocalSocketAddress address = new LocalSocketAddress("installd",
                    LocalSocketAddress.Namespace.RESERVED);

            mSocket.connect(address);

            mIn = mSocket.getInputStream();
            mOut = mSocket.getOutputStream();
        } catch (IOException ex) {
            disconnect();
            return false;
        }
        return true;
    }

    public void disconnect() {
        Slog.i(TAG, "disconnecting...");
        IoUtils.closeQuietly(mSocket);
        IoUtils.closeQuietly(mIn);
        IoUtils.closeQuietly(mOut);

        mSocket = null;
        mIn = null;
        mOut = null;
    }


    private boolean writeCommand(String cmdString) {
        final byte[] cmd = cmdString.getBytes();
        final int len = cmd.length;
        if ((len < 1) || (len > buf.length)) {
            return false;
        }

        buf[0] = (byte) (len & 0xff);
        buf[1] = (byte) ((len >> 8) & 0xff);
        try {
            mOut.write(buf, 0, 2);
            mOut.write(cmd, 0, len);
        } catch (IOException ex) {
            Slog.e(TAG, "write error");
            disconnect();
            return false;
        }
        return true;
    }

    public void waitForConnection() {
        for (;;) {
            try {
                execute("ping");
                return;
            } catch (InstallerException ignored) {
            }
            Slog.w(TAG, "installd not ready");
            SystemClock.sleep(1000);
        }
    }

    public static class InstallerException extends Exception {
        public InstallerException(String detailMessage) {
            super(detailMessage);
        }
    }
}
```

可以看到在InstallerConnection中通过socket连接installd服务端，然后将Installer传入的cmd通过socket再传给installd处理。

## 为什么需要installd

Android已经有了PMS这么复杂的一个服务了，为什么还需要再有一个installd守护进程？

答案其实很简单，system_server以system用户的身份运行，PMS运行在system_server中，所以PMS也是system用户。installd是root用户。

```
USER      PID   PPID  VSIZE  RSS   WCHAN              PC  NAME

system    1698  699   2408960 149752 SyS_epoll_ 0000000000 S system_server
root      706   1     9972   2612  unix_strea 0000000000 S /system/bin/installd

```

system用户并没有访问应用程序目录的权限。但是作为root用户的installd却可以访问data/data/下的目录，这就是installd存在的原因。