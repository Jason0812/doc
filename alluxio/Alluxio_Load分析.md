# Alluxio Load分析

本文档主要分析Alluxio是如何将UFS中的数据Load到Alluxio的Space中，以及在Alluxio Space中是如何存放的；

```java
FileInStream in = closer.register(mFileSystem.openFile(filePath, options));
byte[] buf = new byte[8 * Constants.MB];
while (in.read(buf) != -1) {
}
```

代码中Alluxio `open`完文件之后，就是不断地去读取数据；

```java
  public int read(byte[] b, int off, int len) throws IOException {
    while (bytesLeftToRead > 0 && remaining() > 0) {
      updateStreams();
      
      int bytesRead = mCurrentBlockInStream.read(b, currentOffset, bytesToRead);
      if (bytesRead > 0) {
        if (mCurrentCacheStream != null) {
          try {
            mCurrentCacheStream.write(b, currentOffset, bytesRead);
          } catch (IOException e) {
            handleCacheStreamIOException(e);
          }
        }
      }
    }
}
```

首先调用`updateStreams`方法，然后去用`mCurrentBlockInStream`去读取数据，如果`mCurrentCacheStream`不为空，就用它去写数据；

接下来我们分析看`updateStreams`，这个方法中做了些什么工作；

```java
  private void updateStreams() throws IOException {
    long currentBlockId = getCurrentBlockId();
    if (shouldUpdateStreams(currentBlockId)) {
      // The following two function handle negative currentBlockId (i.e. the end of file)
      // correctly.
      updateBlockInStream(currentBlockId);
      updateCacheStream(currentBlockId);
      mStreamBlockId = currentBlockId;
    }
  }
```

