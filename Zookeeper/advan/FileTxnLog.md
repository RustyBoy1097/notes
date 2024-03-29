## FileTxnLog

持久化的类主要在包`org.apache.zookeeper.server.persistence`下，此次也主要是对其下的类进行分析，其包下总体的类结构如下图所示。

![](../images/persistence.png)

* `TxnLog`，接口类型，读取事务性日志的接口。
* `FileTxnLog`，实现TxnLog接口，添加了访问该事务性日志的API。
* `Snapshot`，接口类型，持久层快照接口。
* `FileSnap`，实现Snapshot接口，负责存储、序列化、反序列化、访问快照。
* `FileTxnSnapLog`，封装了TxnLog和SnapShot。
* `Util`，工具类，提供持久化所需的API。

下面先来分析TxnLog和FileTxnLog的源码。

### 1. TxnLog源码分析

TxnLog是接口，规定了对日志的响应操作。
```java
public interface TxnLog {

    /**
     * roll the current
     * log being appended to
     * @throws IOException 
     */
    // 回滚日志
    void rollLog() throws IOException;
    /**
     * Append a request to the transaction log
     * @param hdr the transaction header
     * @param r the transaction itself
     * returns true iff something appended, otw false 
     * @throws IOException
     */
    // 添加一个请求至事务性日志
    boolean append(TxnHeader hdr, Record r) throws IOException;

    /**
     * Start reading the transaction logs
     * from a given zxid
     * @param zxid
     * @return returns an iterator to read the 
     * next transaction in the logs.
     * @throws IOException
     */
    // 读取事务性日志
    TxnIterator read(long zxid) throws IOException;
    
    /**
     * the last zxid of the logged transactions.
     * @return the last zxid of the logged transactions.
     * @throws IOException
     */
    // 事务性操作的最新zxid
    long getLastLoggedZxid() throws IOException;
    
    /**
     * truncate the log to get in sync with the 
     * leader.
     * @param zxid the zxid to truncate at.
     * @throws IOException 
     */
    // 清空日志，与Leader保持同步
    boolean truncate(long zxid) throws IOException;
    
    /**
     * the dbid for this transaction log. 
     * @return the dbid for this transaction log.
     * @throws IOException
     */
    // 获取数据库的id
    long getDbId() throws IOException;
    
    /**
     * commmit the trasaction and make sure
     * they are persisted
     * @throws IOException
     */
    // 提交事务并进行确认
    void commit() throws IOException;
   
    /** 
     * close the transactions logs
     */
    // 关闭事务性日志
    void close() throws IOException;
    /**
     * an iterating interface for reading 
     * transaction logs. 
     */
    // 读取事务日志的迭代器接口
    public interface TxnIterator {
        /**
         * return the transaction header.
         * @return return the transaction header.
         */
        // 获取事务头部
        TxnHeader getHeader();
        
        /**
         * return the transaction record.
         * @return return the transaction record.
         */
        // 获取事务
        Record getTxn();
     
        /**
         * go to the next transaction record.
         * @throws IOException
         */
        // 下个事务
        boolean next() throws IOException;
        
        /**
         * close files and release the 
         * resources
         * @throws IOException
         */
        // 关闭文件释放资源
        void close() throws IOException;
    }
}
```
其中，TxnLog除了提供读写事务日志的API外，还提供了一个用于读取日志的迭代器接口TxnIterator。

### 2. FileTxnLog源码分析

对于LogFile而言，其格式可分为如下三部分

`LogFile`: **FileHeader == TxnList == ZeroPad**

`FileHeader`格式如下
```
FileHeader: {
    magic 4bytes (ZKLG)
    version 4bytes
    dbid 8bytes
}
```

`TxnList`格式如下
```
TxnList:
    Txn || Txn TxnList
```

`Txn`格式如下
```
Txn:
    checksum Txnlen TxnHeader Record 0x42
```
`Txnlen`格式如下
```
Txnlen:
    len 4bytes
```

`TxnHeader`格式如下
```
TxnHeader: {
    
    sessionid 8bytes
    cxid 4bytes
    zxid 8bytes
    time 8bytes
    type 4bytes
}
```
`Record`格式如下
```
Record:
    See Jute definition file for details on the various record types
```

`ZeroPad`格式如下
```
ZeroPad:
    0 padded to EOF (filled during preallocation stage)
```

了解LogFile的格式对于理解源码会有很大的帮助。

#### 2.1 属性

```
public class FileTxnLog implements TxnLog {
    private static final Logger LOG;
    // 预分配大小 64M
    static long preAllocSize =  65536 * 1024;
    // 魔术数字，默认为1514884167
    public final static int TXNLOG_MAGIC = ByteBuffer.wrap("ZKLG".getBytes()).getInt();
    // 版本号
    public final static int VERSION = 2;

    /** Maximum time we allow for elapsed fsync before WARNing */
    // 进行同步时，发出warn之前所能等待的最长时间
    private final static long fsyncWarningThresholdMS;

    // 静态属性，确定Logger、预分配空间大小和最长时间
    static {
        LOG = LoggerFactory.getLogger(FileTxnLog.class);

        String size = System.getProperty("zookeeper.preAllocSize");
        if (size != null) {
            try {
                // code move to FilePadding
                preAllocSize = Long.parseLong(size) * 1024;
            } catch (NumberFormatException e) {
                LOG.warn(size + " is not a valid value for preAllocSize");
            }
        }
        fsyncWarningThresholdMS = Long.getLong("fsync.warningthresholdms", 1000);
    }
    
    // 最大(新)的zxid
    long lastZxidSeen;
    // 存储数据相关的流
    volatile BufferedOutputStream logStream = null;
    volatile OutputArchive oa;
    volatile FileOutputStream fos = null;

    // log目录文件
    File logDir;
    
    // 是否强制同步
    private final boolean forceSync = !System.getProperty("zookeeper.forceSync", "yes").equals("no");;
    
    // 数据库id
    long dbId;
    
    // 流列表
    private LinkedList<FileOutputStream> streamsToFlush = new LinkedList<FileOutputStream>();
    
    // 当前大小
    long currentSize;
    // 写日志文件
    File logFileWrite = null;
}
```

#### 2.2 核心函数

##### 2.2.1 append函数

```
public synchronized boolean append(TxnHeader hdr, Record txn) throws IOException {
    if (hdr != null) { // 事务头部不为空
        if (hdr.getZxid() <= lastZxidSeen) { // 事务的zxid小于等于最后的zxid
            LOG.warn("Current zxid " + hdr.getZxid() + " is <= " + lastZxidSeen + " for " + hdr.getType());
        }
        if (logStream==null) { // 日志流为空
            if(LOG.isInfoEnabled()){
                LOG.info("Creating new log file: log." + Long.toHexString(hdr.getZxid()));
            }
            // 
            logFileWrite = new File(logDir, ("log." + Long.toHexString(hdr.getZxid())));
            fos = new FileOutputStream(logFileWrite);
            logStream=new BufferedOutputStream(fos);
            oa = BinaryOutputArchive.getArchive(logStream);
            // 
            FileHeader fhdr = new FileHeader(TXNLOG_MAGIC,VERSION, dbId);
            // 序列化
            fhdr.serialize(oa, "fileheader");
            // Make sure that the magic number is written before padding.
            // 刷新到磁盘
            logStream.flush();
            
            // 当前通道的大小
            currentSize = fos.getChannel().position();
            // 添加fos
            streamsToFlush.add(fos);
        }
        
        // 填充文件
        padFile(fos);
        
        // Serializes transaction header and transaction data into a byte buffer.
        // 将事务头和事务数据序列化成Byte Buffer
        byte[] buf = Util.marshallTxnEntry(hdr, txn);
        if (buf == null || buf.length == 0) { // 为空，抛出异常
            throw new IOException("Faulty serialization for header " + "and txn");
        }
        // 生成一个验证算法
        Checksum crc = makeChecksumAlgorithm();
        // Updates the current checksum with the specified array of bytes
        // 使用Byte数组来更新当前的Checksum
        crc.update(buf, 0, buf.length);
        // 写long类型数据
        oa.writeLong(crc.getValue(), "txnEntryCRC");
        // Write the serialized transaction record to the output archive.
        // 将序列化的事务记录写入OutputArchive
        Util.writeTxnBytes(oa, buf);
        
        return true;
    }
    return false;
}
```

说明：`append`函数主要用做向事务日志中添加一个条目，其大体步骤如下:

* 1.检查`TxnHeader`是否为空，若不为空，则进入**2**，否则，直接返回`false`
* 2.检查`logStream`是否为空(初始化为空)，若不为空，则进入**3**，否则，进入**5**
* 3.初始化写数据相关的流和`FileHeader`，并序列化`FileHeader`至指定文件，进入**4**
* 4.强制刷新（保证数据存到磁盘），并获取当前写入数据的大小。进入**5**
* 5.填充数据，填充`0`，进入**6**
* 6.将事务头和事务序列化成`ByteBuffer`（使用`Util.marshallTxnEntry`函数），进入**7**
* 7.使用`Checksum`算法更新步骤**6**的`ByteBuffer`。进入**9**
* 8.将更新的`ByteBuffer`写入磁盘文件，返回`true`

`append`间接调用了`padLog`函数，其源码如下

```
public static long padLogFile(FileOutputStream f,long currentSize, long preAllocSize) throws IOException{
    // 获取位置
    long position = f.getChannel().position();
    if (position + 4096 >= currentSize) { // 计算后是否大于当前大小
        // 重新设置当前大小，剩余部分填充0
        currentSize = currentSize + preAllocSize;
        fill.position(0);
        f.getChannel().write(fill, currentSize-fill.remaining());
    }
    return currentSize;
}
```

说明：其主要作用是当文件大小不满**64MB**时，向文件填充0以达到**64MB**大小。

##### 2.2.2 getLogFiles函数

```
public static File[] getLogFiles(File[] logDirList,long snapshotZxid) {
    // 按照zxid对文件进行排序
    List<File> files = Util.sortDataDir(logDirList, "log", true);
    long logZxid = 0;
    // Find the log file that starts before or at the same time as the zxid of the snapshot
    for (File f : files) { // 遍历文件
        // 从文件中获取zxid
        long fzxid = Util.getZxidFromName(f.getName(), "log");
        if (fzxid > snapshotZxid) { // 跳过大于snapshotZxid的文件
            continue;
        }
        // the files are sorted with zxid's
        if (fzxid > logZxid) { // 找出文件中最大的zxid(同时还需要小于等于snapshotZxid)
            logZxid = fzxid;
        }
    }
    // 文件列表
    List<File> v=new ArrayList<File>(5);
    for (File f : files) { // 再次遍历文件
        // 从文件中获取zxid
        long fzxid = Util.getZxidFromName(f.getName(), "log");
        if (fzxid < logZxid) { // 跳过小于logZxid的文件
            continue;
        }
        // 添加
        v.add(f);
    }
    // 转化成File[] 类型后返回
    return v.toArray(new File[0]);
}
```

说明：该函数的作用是找出刚刚小于或者等于**snapshot**的所有log文件。其步骤大致如下。

* 1.对所有**log文件**按照`zxid`进行升序排序，进入**2**
* 2.遍历所有**log文件**并记录刚刚小于或等于给定`snapshotZxid`的log文件的`logZxid`，进入**3**
* 3.再次遍历**log文件**，添加`zxid`大于等于步骤**2**中的`logZxid`的所有**log文件**，进入**4**
* 4.转化后返回

`getLogFiles`函数调用了`sortDataDir`，其源码如下：

```
public static List<File> sortDataDir(File[] files, String prefix, boolean ascending) {
    if(files==null) 
        return new ArrayList<File>(0);
    // 转化为列表
    List<File> filelist = Arrays.asList(files);
    // 进行排序，Comparator是关键，根据zxid进行排序
    Collections.sort(filelist, new DataDirFileComparator(prefix, ascending));
    return filelist;
}
```

说明：其用于排序**log文件**，可以选择根据`zxid`进行升序或降序。

`getLogFiles`函数间接调用了`getZxidFromName`，其源码如下：

```
// 从文件名中解析出zxid
public static long getZxidFromName(String name, String prefix) {
    long zxid = -1;
    // 对文件名进行分割
    String nameParts[] = name.split("\\.");
    if (nameParts.length == 2 && nameParts[0].equals(prefix)) { // 前缀相同
        try {
            // 转化成长整形
            zxid = Long.parseLong(nameParts[1], 16);
        } catch (NumberFormatException e) {}
    }
    return zxid;
}
```

说明：`getZxidFromName`主要用作从文件名中解析`zxid`，并且需要从指定的前缀开始。

##### 2.2.3 getLastLoggedZxid函数

```
public long getLastLoggedZxid() {
    // 获取已排好序的所有的log文件
    File[] files = getLogFiles(logDir.listFiles(), 0);
    // 获取最大的zxid(最后一个log文件对应的zxid)
    long maxLog=files.length>0? Util.getZxidFromName(files[files.length-1].getName(),"log"):-1;

    // if a log file is more recent we must scan it to find the highest zxid
    long zxid = maxLog;
    // 迭代器
    TxnIterator itr = null;
    try {
        // 新生FileTxnLog
        FileTxnLog txn = new FileTxnLog(logDir);
        // 开始读取从给定zxid之后的所有事务
        itr = txn.read(maxLog);
        while (true) { // 遍历
            if(!itr.next()) // 是否存在下一项
                break;
            // 获取事务头
            TxnHeader hdr = itr.getHeader();
            // 获取zxid
            zxid = hdr.getZxid();
        }
    } catch (IOException e) {
        LOG.warn("Unexpected exception", e);
    } finally {
        // 关闭迭代器
        close(itr);
    }
    return zxid;
}
```

说明：该函数主要用于获取记录在log中的最后一个`zxid`。其步骤大致如下: 

* 1.获取**已排好序的所有log文件**，并从最后一个文件中取出`zxid`作为候选的最大`zxid`，进入**3**
* 2.新生成`FileTxnLog`并读取步骤**1**中`zxid`之后的所有事务，进入**3**
* 3.遍历所有事务并提取出相应的`zxid`，最后返回。

其中`getLastLoggedZxid`调用了`read`函数，其源码如下

```
public TxnIterator read(long zxid) throws IOException {
    // 返回事务文件访问迭代器
    return new FileTxnIterator(logDir, zxid);
}
```

说明：`read`函数会生成一个`FileTxnIterator`，其是`TxnLog.TxnIterator`的子类，之后在`FileTxnIterator`构造函数中会调用`init`函数，其源码如下

```
void init() throws IOException {
    // 新生成文件列表
    storedFiles = new ArrayList<File>();
    // 进行排序
    List<File> files = Util.sortDataDir(FileTxnLog.getLogFiles(logDir.listFiles(), 0), "log", false);
    for (File f: files) { // 遍历文件
        if (Util.getZxidFromName(f.getName(), "log") >= zxid) { // 添加zxid大于等于指定zxid的文件
            storedFiles.add(f);
        } else if (Util.getZxidFromName(f.getName(), "log") < zxid) { // 只添加一个zxid小于指定zxid的文件，然后退出
            // add the last logfile that is less than the zxid
            storedFiles.add(f);
            break;
        }
    }
    // go to the next logfile
    // 进入下一个log文件
    goToNextLog();
    if (!next()) // 不存在下一项，返回
        return;
    while (hdr.getZxid() < zxid) { // 从事务头中获取zxid小于给定zxid，直到不存在下一项或者大于给定zxid时退出
        if (!next())
            return;
    }
}
```

说明：`init`函数用于进行初始化操作，会根据`zxid`的不同进行不同的初始化操作，在`init`函数中会调用`goToNextLog`函数，其源码如下

```
private boolean goToNextLog() throws IOException {
    if (storedFiles.size() > 0) { // 存储的文件列表大于0
        // 取最后一个log文件
        this.logFile = storedFiles.remove(storedFiles.size()-1);
        // 针对该文件，创建InputArchive
        ia = createInputArchive(this.logFile);
        // 返回true
        return true;
    }
    return false;
}
```

说明：`goToNextLog`表示选取下一个**log文件**，在`init`函数中还调用了`next`函数，其源码如下

```
public boolean next() throws IOException {
    if (ia == null) { // 为空，返回false
        return false;
    }
    try {
        // 读取长整形crcValue
        long crcValue = ia.readLong("crcvalue");
        // 通过input archive读取一个事务条目
        byte[] bytes = Util.readTxnBytes(ia);
        // Since we preallocate, we define EOF to be an
        if (bytes == null || bytes.length==0) { // 对bytes进行判断
            throw new EOFException("Failed to read " + logFile);
        }
        // EOF or corrupted record validate CRC
        // 验证CRC
        Checksum crc = makeChecksumAlgorithm();
        // 更新
        crc.update(bytes, 0, bytes.length);
        if (crcValue != crc.getValue()) // 验证不相等，抛出异常
            throw new IOException(CRC_ERROR);
        if (bytes == null || bytes.length == 0) // bytes为空，返回false
            return false;
        // 新生成TxnHeader
        hdr = new TxnHeader();
        // 将Txn反序列化，并且将对应的TxnHeader反序列化至hdr，整个Record反序列化至record
        record = SerializeUtils.deserializeTxn(bytes, hdr);
    } catch (EOFException e) { // 抛出异常
        LOG.debug("EOF excepton " + e);
        // 关闭输入流
        inputStream.close();
        // 赋值为null
        inputStream = null;
        ia = null;
        hdr = null;
        // this means that the file has ended we should go to the next file
        if (!goToNextLog()) { // 没有log文件，则返回false
            return false;
        }
        // if we went to the next log file, we should call next() again
        // 继续调用next
        return next();
    } catch (IOException e) {
        inputStream.close();
        throw e;
    }
    // 返回true
    return true;
}
```

说明：`next`表示将迭代器移动至下一个事务，方便读取，`next`函数的步骤如下。

* 1.读取事务的`crcValue`值，用于后续的验证，进入**2**
* 2.读取事务，使用**CRC32**进行更新并与**1**中的结果进行比对，若不相同，则抛出异常，否则，进入**3**
* 3.将事务进行反序列化并保存至相应的属性中（如事务头和事务体），会确定具体的事务操作类型。
* 4.在读取过程抛出异常时，会首先关闭流，然后再尝试调用`next`函数（即进入下一个事务进行读取）。

##### 2.2.4 commit函数

```
public synchronized void commit() throws IOException {
    if (logStream != null) {
        // 强制刷到磁盘
        logStream.flush();
    }
    for (FileOutputStream log : streamsToFlush) { // 遍历流
        // 强制刷到磁盘
        log.flush();
        if (forceSync) { // 是否强制同步
            long startSyncNS = System.nanoTime();
            
            log.getChannel().force(false);
            // 计算流式的时间
            long syncElapsedMS =
                TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startSyncNS);
            if (syncElapsedMS > fsyncWarningThresholdMS) { // 大于阈值时则会警告
                LOG.warn("fsync-ing the write ahead log in " + Thread.currentThread().getName()
                        + " took " + syncElapsedMS
                        + "ms which will adversely effect operation latency. "
                        + "See the ZooKeeper troubleshooting guide");
            }
        }
    }
    while (streamsToFlush.size() > 1) { // 移除流并关闭
        streamsToFlush.removeFirst().close();
    }
}
```

说明：该函数主要用于提交事务日志至磁盘，其大致步骤如下

* 1.若日志流`logStream`不为空，则强制刷新至磁盘，进入**2**
* 2.遍历需要刷新至磁盘的所有流`streamsToFlush`并进行刷新，进入**3**
* 3.判断是否需要强制性同步，如是，则计算每个流的流式时间并在控制台给出警告，进入**4**
* 4.移除所有流并关闭。

##### 2.2.5 truncate函数

```
public boolean truncate(long zxid) throws IOException {
    FileTxnIterator itr = null;
    try {
        // 获取迭代器
        itr = new FileTxnIterator(this.logDir, zxid);
        PositionInputStream input = itr.inputStream;
        long pos = input.getPosition();
        // now, truncate at the current position
        // 从当前位置开始清空
        RandomAccessFile raf = new RandomAccessFile(itr.logFile, "rw");
        raf.setLength(pos);
        raf.close();
        while (itr.goToNextLog()) { // 存在下一个log文件
            if (!itr.logFile.delete()) { // 删除
                LOG.warn("Unable to truncate {}", itr.logFile);
            }
        }
    } finally {
        // 关闭迭代器
        close(itr);
    }
    return true;
}
```

说明：该函数用于清空大于给定`zxid`的所有事务日志。