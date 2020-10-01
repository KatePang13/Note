# kafka 源码学习——Log

## Log Segment

### 日志结构

一般情况下，一个Kafka主题有多个分区，每个分区就对应一个 Log 对象，在物理磁盘上则对应一个子目录，比如，创建一个双分区的主题 test-topic, 那么，Kafka 会在磁盘上创建2个子目录: test-topic-1, test-topic-2。每个子目录下包含多组日志段 segment,

每个segment 主要包含日志文件和索引文件按2部分，文件名为该segment的基位移，表示该日志的初识位移，log 文件包含实际的消息，index 保存从逻辑位移到物理文件中位置的映射

`[base_offset].index`

`[base_offset].log`

### 代码

core/src/main/scala/kafka/log/LogSegment.scala

主要包含了  

`class LogSegment`

`object LogSegment`

这里是scala的一个特色的语法，叫伴生关系，object 定义了一个单例对象，用于定义一些静态的变量和方法，而class就是常规的类。

```java
/* @param log The file records containing log entries
 * @param lazyOffsetIndex The offset index
 * @param lazyTimeIndex The timestamp index
 * @param txnIndex The transaction index
 * @param baseOffset A lower bound on the offsets in this segment
 * @param indexIntervalBytes The approximate number of bytes between entries in the index
 * @param rollJitterMs The maximum random jitter subtracted from the scheduled segment roll time
 * @param time The time instance
 */
@nonthreadsafe
class LogSegment private[log] (val log: FileRecords,
                               val lazyOffsetIndex: LazyIndex[OffsetIndex],
                               val lazyTimeIndex: LazyIndex[TimeIndex],
                               val txnIndex: TransactionIndex,
                               val baseOffset: Long,
                               val indexIntervalBytes: Int,
                               val rollJitterMs: Long,
                               val time: Time) extends Logging {}
```

#### 主要成员变量

- FileRecords  包含消息的对象
- lazyOffsetIndex， lazyTimeIndex   延迟初始化相关的索引
- txnIndex  事务索引
- baseOffset  segment 的基偏移量，磁盘上的文件名就是这个baseOffset  
- indexIntervalBytes     控制日志段对象新增索引项的频率
- rollJitterMs    日志段对象新增倒计时的“扰动值”
  - 避免在同一时刻创建多个日志段，缓解物理磁盘的 I/O 负载瓶颈
- time 统计计时相关

#### 主要方法

- append 
- read
- recover

```java
  /**
   * Append the given messages starting with the given offset. Add
   * an entry to the index if needed.
   * 从指定的offset开始，append若干个消息
   * 需要的时候，在 index文件上添加一个index项
   *
   * It is assumed this method is being called from within a lock.
   * 假设该方法调用时，外部有加锁
   *
   * @param largestOffset The last offset in the message set  消息集中的最大索引值
   * @param largestTimestamp The largest timestamp in the message set. 消息集中的最大时间戳
   * @param shallowOffsetOfMaxTimestamp The offset of the message that has the largest timestamp in the messages to append.  
   * @param records The log entries to append.   消息集
   * @return the physical position in the file of the appended records 返回append的物理位置
   * @throws LogSegmentOffsetOverflowException if the largest offset causes index offset overflow
   */
  @nonthreadsafe
  def append(largestOffset: Long,
             largestTimestamp: Lon
             shallowOffsetOfMaxTimestamp: Long,
             records: MemoryRecords): Unit = {
    if (records.sizeInBytes > 0) {
      trace(s"Inserting ${records.sizeInBytes} bytes at end offset $largestOffset at position ${log.sizeInBytes} " +
            s"with largest timestamp $largestTimestamp at shallow offset $shallowOffsetOfMaxTimestamp")
      val physicalPosition = log.sizeInBytes()
      if (physicalPosition == 0)
        rollingBasedTimestamp = Some(largestTimestamp)

      ensureOffsetInRange(largestOffset)

      // append the messages
      val appendedBytes = log.append(records)
      trace(s"Appended $appendedBytes to ${log.file} at end offset $largestOffset")
      // Update the in memory max timestamp and corresponding offset.
      if (largestTimestamp > maxTimestampSoFar) {
        maxTimestampSoFar = largestTimestamp
        offsetOfMaxTimestampSoFar = shallowOffsetOfMaxTimestamp
      }
      // append an entry to the index (if needed)
      if (bytesSinceLastIndexEntry > indexIntervalBytes) {
        offsetIndex.append(largestOffset, physicalPosition)
        timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestampSoFar)
        bytesSinceLastIndexEntry = 0
      }
      bytesSinceLastIndexEntry += records.sizeInBytes
    }
  }
```



