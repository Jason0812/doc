# Alluxio Master 代码框架

1. `cli`: format the journal node, it will interact with Under FileSystem;

​	format master process: delete ufs folder -> ufs mkdir it -> touch file;

​	format work process: delete data;

```Java
for (int level = 0; level < storageLevels; level++) {
        PropertyKey tierLevelDirPath =
            PropertyKeyFormat.WORKER_TIERED_STORE_LEVEL_DIRS_PATH_FORMAT.format(level);
        String[] dirPaths = Configuration.get(tierLevelDirPath).split(",");
        String name = "TIER_" + level + "_DIR_PATH";
        for (String dirPath : dirPaths) {
          String dirWorkerDataFolder = PathUtils.concatPath(dirPath.trim(), workerDataFolder);
          UnderFileSystem ufs = UnderFileSystem.Factory.get(dirWorkerDataFolder);
          if (ufs.isDirectory(dirWorkerDataFolder)) {
            if (!formatFolder(name, dirWorkerDataFolder)) {
              throw new RuntimeException(String.format("Failed to format worker data folder %s",
                  dirWorkerDataFolder));
            }
          }
        }
```

2. Master
   1. Block: Manage the Block Info and worker info
   2. File: AsyncPersisteHandler
   3. journal
   4. lineage

## AsyncPersisteHandler