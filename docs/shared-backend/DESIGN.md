Introduction
------------

Shared backend is a new feature which enables running multiple Minio instances on a single shared backend safely. This allows for sharing the I/O load across many minio servers running remotely while sharing the same backend.

Motivation
----------

Since Minio instances serve the purpose of a single tenant there is an increasing requirement where users want to run Minio instances on a backend which is managed by an existing NAS (NFS, GlusterFS, Other distributed filesystems) rather than a local disk. This feature is implemented also with minimal disruption in mind for the user and overall UI.

Restrictions
------------

* A PutObject() is blocked and waits if another GetObject() is in progress.
* A CompleteMultipartUpload() is blocked and waits if another PutObject() or GetObject() is in progress.
* Cannot run FS mode as a remote disk RPC.

--> Add more here ^^

## How To Run?

Running Minio instances on shared backend is no different than running on a stand-alone disk. Command line doesn't change at all. Below examples will clarify further for each operating system of your choice:

### Ubuntu 16.04 LTS

Example 1: Start Minio instance on a shared backend mounted and available at `/mnt/nfs`.

On linux server1
```shell
minio server /mnt/nfs
```

On linux server2
```shell
minio server /mnt/nfs
```

### Windows 2012 Server

Example 1: Start Minio instance on a shared backend mounted and available at `\\remote-server\cifs`.

On windows server1
```cmd
minio.exe server \\remote-server\cifs\export
```

On windows server2
```cmd
minio.exe server \\remote-server\cifs\export
```

Alternatively if `\\remote-server\cifs` is mounted as `D:\` drive.

On windows server1
```cmd
minio.exe server D:\export
```

On windows server2
```cmd
minio.exe server D:\export
```

Architecture
------------------

## POSIX/Win32 Locks

### Lock process

With in the same Minio instance locking is handled by existing in-memory namespace locks (**sync.RWMutex** et. al).  To synchronize locks between many Minio instances we leverage POSIX `fcntl()` locks on Unixes and on Windows `LockFileEx()` Win32 API. Requesting write lock block if there are any read locks held by neighboring Minio instance on the same path. So does the read lock if there are any active write locks in-progress.

### Unlock process

Unlocking happens on filesystems locks by just closing the file descriptor (fd) which was initially requested for lock operation. Closing the fd tells the kernel to relinquish all the locks held on the path by the current process. This gets trickier when there are many readers on the same path by the same process, it would mean that closing an fd relinquishes locks for all concurrent readers as well. To properly manage this situation a simple fd reference count is implemented, the same fd is shared between many readers. When readers start closing on the fd we start reducing the reference count, once reference count has reached zero we can be sure that there are no more readers active. So we proceed and close the underlying file descriptor which would relinquish the read lock held on the path.

This doesn't apply for the writes because there is always one writer and many readers for any unique object.

## Handling Concurrency.

An example here shows how the contention is handled with GetObject().

GetObject() holds a read lock on `fs.json`.
```go

	fsMetaPath := pathJoin(fs.fsPath, minioMetaBucket, bucketMetaPrefix, bucket, object, fsMetaJSONFile)
	var rlk *lock.RLockedFile
	rlk, err = fs.rwPool.Open(fsMetaPath)
	if err != nil {
		return toObjectErr(traceError(err), bucket, object)
	}
	defer rlk.Close()

... you can perform other operations here ...

	_, err = io.CopyBuffer(writer, reader, buf)

... after successful copy operation unlocks the read lock ...

```

A concurrent PutObject is requested on the same object, PutObject() attempts a write lock on `fs.json`.

```go
	fsMetaPath := pathJoin(fs.fsPath, minioMetaBucket, bucketMetaPrefix, bucket, object, fsMetaJSONFile)
	wlk, err := fs.rwPool.Create(fsMetaPath)
	if err != nil {
		return ObjectInfo{}, toObjectErr(err, bucket, object)
	}
	// This close will allow for locks to be synchronized on `fs.json`.
	defer wlk.Close()
```

Now from the above snippet the following code one can notice that until the GetObject() returns writing to the client. Following portion of the code will block.

```go
	wlk, err := fs.rwPool.Create(fsMetaPath)
```

This restriction is needed so that corrupted data is not returned to the client in between I/O. The logic works vice-versa as well an on-going PutObject(), GetObject() would wait for the PutObject() to complete.
