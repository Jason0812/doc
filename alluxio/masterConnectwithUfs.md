#Alluxio interact with UFS

## Client

1. 在create File的时候，client端会根据WriteType去决定文件是否需要在Ufs中创建；如果需要，client会根据mUfsDelegation，决定是由client端去ufs中操作，还是说通过worker去操作ufs；（这个选项在1.5中取消了，即1.5只支持worker端去ufs中createFile，这样会存在一个问题： 就是当Alluxio的worker的数量和client的数量相差一个级别的时候，worker可能会有性能问题？？？？）

```java
if (!mUnderStorageType.isSyncPersist()) {
  mUfsPath = null;
  mUnderStorageOutputStream = null;
  mFileSystemWorkerClient = null;
  mUfsFileId = null;
} else {
  mUfsPath = options.getUfsPath();
  if (mUfsDelegation) {
    mFileSystemWorkerClient = mCloser.register(mContext.createFileSystemWorkerClient());
    Permission perm = options.getPermission();
    mUfsFileId = mFileSystemWorkerClient.createUfsFile(new AlluxioURI(mUfsPath),
        CreateUfsFileOptions.defaults().setPermission(perm));
    mUnderStorageOutputStream = mCloser.register(mUnderOutStreamFactory
        .create(mContext, mFileSystemWorkerClient.getWorkerDataServerAddress(), mUfsFileId));
  } else {
    UnderFileSystem ufs = UnderFileSystem.Factory.get(mUfsPath);
    // TODO(jiri): Implement collection of temporary files left behind by dead clients.
    // Parent directory creation in ufs is not required as FileSystemMaster will create any
    // required directories as part of inode creation if sync persist = true
    CreateOptions createOptions =
        CreateOptions.defaults().setPermission(options.getPermission());
    mUnderStorageOutputStream = mCloser.register(ufs.create(mUfsPath, createOptions));

    // Set delegation related vars to null as we are not using worker delegation for ufs ops
    mFileSystemWorkerClient = null;
    mUfsFileId = null;
  }
}
```

## Master （FileSystemMaster）

1. InodeTree createPath中会根据isPersisted，从而去Ufs中创建；

```java
for (Inode<?> inode : toPersistDirectories) {
  //todo(Jason): remove it.
  MountTable.Resolution resolution = mMountTable.resolve(getPath(inode));
  String ufsUri = resolution.getUri().toString();
  UnderFileSystem ufs = resolution.getUfs();
  Permission permission = new Permission(inode.getOwner(), inode.getGroup(), inode.getMode());
  MkdirsOptions mkdirsOptions = MkdirsOptions.defaults().setCreateParent(false)
      .setPermission(permission);
  if (ufs.isDirectory(ufsUri) || ufs.mkdirs(ufsUri, mkdirsOptions)) {
    inode.setPersistenceState(PersistenceState.PERSISTED);
  }
}
```

2. 在删除文件的时候，Alluxio Master也会去UFs中去删除文件（deleteInternal）；

在删除文件的时候，Alluxio Master的做法是先去Ufs中删除，然后删除Alluxio的Block 信息，最后删除掉Alluxio的Metadata信息；

```java
if (!replayed && delInode.isPersisted()) {
  try {
    // If this is a mount point, we have deleted all the children and can unmount it
    // TODO(calvin): Add tests (ALLUXIO-1831)
    if (mMountTable.isMountPoint(alluxioUriToDel)) {
      unmountInternal(alluxioUriToDel);
    } else {
      // Delete the file in the under file system.
      MountTable.Resolution resolution = mMountTable.resolve(alluxioUriToDel);
      String ufsUri = resolution.getUri().toString();
      UnderFileSystem ufs = resolution.getUfs();
      boolean failedToDelete = false;
      if (delInode.isFile()) {
        if (!ufs.deleteFile(ufsUri)) {
          failedToDelete = ufs.isFile(ufsUri);
          if (!failedToDelete) {
            LOG.warn("The file to delete does not exist in ufs: {}", ufsUri);
          }
        }
      } else {
        if (!ufs.deleteDirectory(ufsUri, DeleteOptions.defaults().setRecursive(true))) {
          failedToDelete = ufs.isDirectory(ufsUri);
          if (!failedToDelete) {
            LOG.warn("The directory to delete does not exist in ufs: {}", ufsUri);
          }
        }
      }
      if (failedToDelete) {
        LOG.error("Failed to delete {} from the under filesystem", ufsUri);
        throw new IOException(ExceptionMessage.DELETE_FAILED_UFS.getMessage(ufsUri));
      }
    }
  } catch (InvalidPathException e) {
    LOG.warn(e.getMessage());
  }
}
```

3. Alluxio中的master的以下API会去loadMetadata；

```java
private long loadMetadataIfNotExistAndJournal(LockedInodePath inodePath,
    LoadMetadataOptions options)
private long loadDirectoryMetadataAndJournal(LockedInodePath inodePath,
      LoadMetadataOptions options)
private long loadFileMetadataAndJournal(LockedInodePath inodePath,
      MountTable.Resolution resolution, LoadMetadataOptions options)
private long loadMetadataAndJournal(LockedInodePath inodePath, LoadMetadataOptions options)
public long loadMetadata(AlluxioURI path, LoadMetadataOptions options)
```

这些API一般用在

```java
/*用于查询file状态的信息*/
public long getFileId(AlluxioURI path)；
public FileInfo getFileInfo(AlluxioURI path)；
//根据LoadMetadata的信息，去设置File的Ufs信息；是否需要在Alluxio Space
private FileInfo getFileInfoInternal(LockedInodePath inodePath)
public List<FileInfo> listStatus(AlluxioURI path, ListStatusOptions listStatusOptions)
private FileBlockInfo generateFileBlockInfo(LockedInodePath inodePath, BlockInfo blockInfo)
```

4. 在checkConsistency的时候，Alluxio Master会和Ufs进行交互

```java
private boolean checkConsistencyInternal(Inode inode, AlluxioURI path)
    throws FileDoesNotExistException, InvalidPathException, IOException {
  //todo: jason, needed this feature????????
  MountTable.Resolution resolution = mMountTable.resolve(path);
  UnderFileSystem ufs = resolution.getUfs();
  String ufsPath = resolution.getUri().getPath();
  if (ufs == null) {
    return true;
  }
  if (!inode.isPersisted()) {
    return !ufs.exists(ufsPath);
  }
  // TODO(calvin): Evaluate which other metadata fields should be validated.
  if (inode.isDirectory()) {
    return ufs.isDirectory(ufsPath);
  } else {
    InodeFile file = (InodeFile) inode;
    return ufs.isFile(ufsPath)
        && ufs.getFileSize(ufsPath) == file.getLength();
  }
}
```

5. 在completefile的时候，会调用commitBlockInUfs

```java
void completeFileInternal(List<Long> blockIds, LockedInodePath inodePath, long length,
    long opTimeMs)
    throws FileDoesNotExistException, InvalidPathException, InvalidFileSizeException,
    FileAlreadyCompletedException {
  InodeFile inode = inodePath.getInodeFile();
  inode.setBlockIds(blockIds);
  inode.setLastModificationTimeMs(opTimeMs);
  inode.complete(length);
  //todo(jason):
  if (inode.isPersisted()) {
    // Commit all the file blocks (without locations) so the metadata for the block exists.
    long currLength = length;
    for (long blockId : inode.getBlockIds()) {
      long blockSize = Math.min(currLength, inode.getBlockSizeBytes());
      mBlockMaster.commitBlockInUFS(blockId, blockSize);
      currLength -= blockSize;
    }
  }
  Metrics.FILES_COMPLETED.inc();
}
```

6. 在rename的时候，

```java
void renameInternal(LockedInodePath srcInodePath, LockedInodePath dstInodePath, boolean replayed,long opTimeMs)
```

7. 设置属性的时候

```java
private long setAttributeAndJournal(LockedInodePath inodePath, SetAttributeOptions options, boolean rootRequired, boolean ownerRequired)
```

## Master（BlockMaster）



## Worker





## 将Alluxio和Ufs分开

设计：设置一个master端的参数：alluxio.master.ufs.separate.enabled;

从而将alluxio中的client, Master 和worker中， 和Ufs交互的全部断开，让Alluxio只是在Alluxio Space中操作，让Ufs也只是在Ufs中操作。

Alluxio和Ufs的协同操作通过client端来控制；

时间规划：此功能预计在5月15日开发工作结束，然后转入测试；