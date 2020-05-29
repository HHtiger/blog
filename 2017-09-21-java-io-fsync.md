---
layout: post
title: 关于java io 的 write 与 操作系统
tag: [os]
---

sync、fsync与fdatasync 

一个简单的问题：在*nix操作系统上，怎样保证对文件的更新内容成功持久化到硬盘？

### sync

sync函数只是将所有修改过的块缓冲区排入写队列，然后就返回，它并不等待实际写磁盘操作结束。

通常称为update的系统守护进程会周期性地（一般每隔30秒）调用sync函数。这就保证了定期冲洗内核的块缓冲区。命令sync(1)也调用sync函数。

### fsync

fsync函数只对由文件描述符filedes指定的单一文件起作用，并且等待写磁盘操作结束，然后返回。

fsync可用于数据库这样的应用程序，这种应用程序需要确保将修改过的块立即写到磁盘上。

### fdatasync

fdatasync函数类似于fsync，但它只影响文件的数据部分。而除数据外，fsync还会同步更新文件的属性。

对于提供事务支持的数据库，在事务提交时，都要确保事务日志（包含该事务所有的修改操作以及一个提交记录）完全写到硬盘上，才认定事务提交成功并返回给应用层。



但是java和其关系如何呢？

先上个栗子
```
    @Test
    public void file(){

        try (FileOutputStream fo = new FileOutputStream("D:\\temp\\file",true)) {

            for(int i=0;i<17;i++){
                fo.write(String.format("Line : %s \n",i).getBytes());
                fo.flush();
                if(i%3==0){
                    fo.getFD().sync();
                }
                TimeUnit.MILLISECONDS.sleep(1000);
                System.out.println(i);
            }

        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

```

一般来说我们会用flush，但是在需要确保数据落盘的场景，我们如何保证呢？java的实现是哪个级别的sync?

FileDescriptor 内的sync声明如下
> public native void sync() throws SyncFailedException;

然后参阅openjdk的实现
openjdk/jdk/src/java.base/unix/native/libjava/FileDescriptor_md.c
```
JNIEXPORT void JNICALL
Java_java_io_FileDescriptor_sync(JNIEnv *env, jobject this) {
    FD fd = THIS_FD(this);
    if (IO_Sync(fd) == -1) {
        JNU_ThrowByName(env, "java/io/SyncFailedException", "sync failed");
    }
}
```

可以看到jvm直接调用了IO_Sync方法，而该方法的定义位于
openjdk/jdk/src/java.base/unix/native/libjava/io_util_md.h
```
/*
 * Copyright (c) 2003, 2015, Oracle and/or its affiliates. All rights reserved.
 * DO NOT ALTER OR REMOVE COPYRIGHT NOTICES OR THIS FILE HEADER.
 *
 * This code is free software; you can redistribute it and/or modify it
 * under the terms of the GNU General Public License version 2 only, as
 * published by the Free Software Foundation.  Oracle designates this
 * particular file as subject to the "Classpath" exception as provided
 * by Oracle in the LICENSE file that accompanied this code.
 *
 * This code is distributed in the hope that it will be useful, but WITHOUT
 * ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
 * FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
 * version 2 for more details (a copy is included in the LICENSE file that
 * accompanied this code).
 *
 * You should have received a copy of the GNU General Public License version
 * 2 along with this work; if not, write to the Free Software Foundation,
 * Inc., 51 Franklin St, Fifth Floor, Boston, MA 02110-1301 USA.
 *
 * Please contact Oracle, 500 Oracle Parkway, Redwood Shores, CA 94065 USA
 * or visit www.oracle.com if you need additional information or have any
 * questions.
 */

#include "jni_util.h"

/*
 * Macros to use the right data type for file descriptors
 */
#define FD jint

/*
 * Prototypes for functions in io_util_md.c called from io_util.c,
 * FileDescriptor.c, FileInputStream.c, FileOutputStream.c,
 * UnixFileSystem_md.c
 */
ssize_t handleWrite(FD fd, const void *buf, jint len);
ssize_t handleRead(FD fd, void *buf, jint len);
jint handleAvailable(FD fd, jlong *pbytes);
jint handleSetLength(FD fd, jlong length);
jlong handleGetLength(FD fd);
FD handleOpen(const char *path, int oflag, int mode);

/*
 * Macros to set/get fd from the java.io.FileDescriptor.  These
 * macros rely on having an appropriately defined 'this' object
 * within the scope in which they're used.
 * If GetObjectField returns null, SET_FD will stop and GET_FD
 * will simply return -1 to avoid crashing VM.
 */

#define SET_FD(this, fd, fid) \
    if ((*env)->GetObjectField(env, (this), (fid)) != NULL) \
        (*env)->SetIntField(env, (*env)->GetObjectField(env, (this), (fid)),IO_fd_fdID, (fd))

#define GET_FD(this, fid) \
    (*env)->GetObjectField(env, (this), (fid)) == NULL ? \
        -1 : (*env)->GetIntField(env, (*env)->GetObjectField(env, (this), (fid)), IO_fd_fdID)

/*
 * Macros to set/get fd when inside java.io.FileDescriptor
 */
#define THIS_FD(obj) (*env)->GetIntField(env, obj, IO_fd_fdID)

/*
 * Route the routines
 */
#define IO_Sync fsync
#define IO_Read handleRead
#define IO_Write handleWrite
#define IO_Append handleWrite
#define IO_Available handleAvailable
#define IO_SetLength handleSetLength
#define IO_GetLength handleGetLength

#ifdef _ALLBSD_SOURCE
#define open64 open
#define fstat64 fstat
#define stat64 stat
#define lseek64 lseek
#define ftruncate64 ftruncate
#define IO_Lseek lseek
#else
#define IO_Lseek lseek64
#endif

/*
 * On Solaris, the handle field is unused
 */
#define SET_HANDLE(fd) return (jlong)-1

/*
 * Retry the operation if it is interrupted
 */
#define RESTARTABLE(_cmd, _result) do { \
    do { \
        _result = _cmd; \
    } while((_result == -1) && (errno == EINTR)); \
} while(0)

/*
 * IO helper function(s)
 */
void fileClose(JNIEnv *env, jobject this, jfieldID fid);

#ifdef MACOSX
jstring newStringPlatform(JNIEnv *env, const char* str);
#endif
```

所以，fo.getFD().sync()方法本质上调用了系统的fsync方法。

