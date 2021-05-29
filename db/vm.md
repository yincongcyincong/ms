# vm

### mergeTree特点：

1.每个列的数据分别存储。这减少了列扫描期间的开销，因为不需要花费资源来读取和跳过其他列的数据。这也提高了每列的压缩比，因为单个列通常包含相似的数据。

2.行按“主键”排序，主键可以跨多列。主键上没有唯一约束-多行可能具有相同的主键。这允许通过主键或其前缀进行快速行查找和范围扫描。此外，这提高了压缩比，因为连续排序的行通常包含相似的数据。

3.行被拆分为中等大小的blocks。每个块由每列的子块组成。每个块都是独立处理的。这意味着在多CPU系统上接近完美的可伸缩性-只需为所有可用的CPU核心提供独立的块。块大小可以配置，但建议使用大小在64KB-2MB范围内的子块，以便适合CPU缓存。这提高了性能，因为CPU缓存访问比RAM访问快得多。此外，当必须从具有多行的块中访问少数行时，这将减少开销。

4.块被合并为“parts”。这些部分类似于日志结构合并（LSM）树中的SSTables。ClickHouse在后台将较小的部分合并为较大的部分。与规范的LSM不同，MergeTree对大小相似的部分没有严格的级别。合并过程提高了查询性能，因为每个查询检查的部件数量较少。另外，合并过程减少了数据文件的数量，因为每个部分包含的文件数量与列数成比例。另外一个好处是合并列的压缩率更好。

5.部件按“partitions key”分组到partitions中。最初，ClickHouse允许在日期列上创建每月分区。现在可以使用任意表达式来构建分区键。分区键的不同值会导致不同的分区。这使得每个分区的数据归档/删除变得简单快捷。

6. partition目录组成：按下划线分割为rows的数量，block的数量，开始时间戳，结束时间戳，partition,

7. 组成 table(一个月一个目录)->partition（每个partition一个目录）->block（具体存储文件）

table的头
```
type table struct {
	path                string
	smallPartitionsPath string
	bigPartitionsPath   string

	getDeletedMetricIDs func() *uint64set.Set
	retentionMsecs      int64

	ptws     []*partitionWrapper
	ptwsLock sync.Mutex

	flockF *os.File

	stop chan struct{}

	retentionWatcherWG sync.WaitGroup
}
```

partition的头：
```
type part struct {
	ph partHeader

	// Filesystem path to the part.
	//
	// Empty for in-memory part.
	path string

	// Total size in bytes of part data.
	size uint64

	timestampsFile fs.MustReadAtCloser
	valuesFile     fs.MustReadAtCloser
	indexFile      fs.MustReadAtCloser

	metaindex []metaindexRow

	ibCache *indexBlockCache
}

type partHeader struct {
	// RowsCount is the total number of rows in the part.
	RowsCount uint64

	// BlocksCount is the total number of blocks in the part.
	BlocksCount uint64

	// MinTimestamp is the minimum timestamp in the part.
	MinTimestamp int64

	// MaxTimestamp is the maximum timestamp in the part.
	MaxTimestamp int64
}

type partSearch struct {
	// BlockRef contains the reference to the found block after NextBlock call.
	BlockRef BlockRef

	// p is the part to search.
	p *part

	// tsids contains sorted tsids to search.
	tsids []TSID

	// tsidIdx points to the currently searched tsid in tsids.
	tsidIdx int

	// tr is a time range to search.
	tr TimeRange

	metaindex []metaindexRow

	ibCache *indexBlockCache

	bhs []blockHeader

	compressedIndexBuf []byte
	indexBuf           []byte

	err error
}
```
block的头
```
// Block represents a block of time series values for a single TSID.
type Block struct {
	bh blockHeader

	// nextIdx is the next index for reading timestamps and values.
	nextIdx int

	timestamps []int64
	values     []int64

	// Marshaled representation of block header.
	headerData []byte

	// Marshaled representation of timestamps.
	timestampsData []byte

	// Marshaled representation of values.
	valuesData []byte
}

type blockHeader struct {
	// TSID is the TSID for the block.
	// Multiple blocks may have the same TSID.
	TSID TSID

	// MinTimestamp is the minimum timestamp in the block.
	//
	// This is the first timestamp, since rows are sorted by timestamps.
	MinTimestamp int64

	// MaxTimestamp is the maximum timestamp in the block.
	//
	// This is the last timestamp, since rows are sorted by timestamps.
	MaxTimestamp int64

	// FirstValue is the first value in the block.
	//
	// It is stored here for better compression level, since usually
	// the first value significantly differs from subsequent values
	// which may be delta-encoded.
	FirstValue int64

	// TimestampsBlockOffset is the offset in bytes for a block
	// with timestamps in timestamps file.
	TimestampsBlockOffset uint64

	// ValuesBlockOffset is the offset in bytes for a block with values
	// in values file.
	ValuesBlockOffset uint64

	// TimestampsBlocksSize is the size in bytes for a block with timestamps.
	TimestampsBlockSize uint32

	// ValuesBlockSize is the size in bytes for a block with values.
	ValuesBlockSize uint32

	// RowsCount is the number of rows in the block.
	//
	// The block must contain at least one row.
	RowsCount uint32

	// Scale is the 10^Scale multiplier for values in the block.
	Scale int16

	// TimestampsMarshalType is the marshal type used for marshaling
	// a block with timestamps.
	TimestampsMarshalType encoding.MarshalType

	// ValuesMarshalType is the marshal type used for marshaling
	// a block with values.
	ValuesMarshalType encoding.MarshalType

	// PrecisionBits is the number of significant bits when using
	// MarshalTypeNearestDelta2 encoding.
	//
	// Possible values are in the range [1...64], where
	//     1 means max 50% error,
	//     2 means max 25% error,
	//     n means max 100/(2^n)% error,
	//    64 means exact values.
	//
	// Lower PrecisionBits give better block compression and speed.
	PrecisionBits uint8
}
```

### mmap会减慢go进程运行速度
1.给定的内存地址指向内存中已经存在的热数据。这种内存称为页缓存。在这种情况下，访问可能确实比通过读/写syscall访问要快。

2.访问冷数据时，会加载这些数据到page cache。加载时，操作系统会拦截cache请求，并返回`major page fault`错误，然后将控制权返还给进程

3.go的调度中,访问内存返回`major page fault`错误时，并不会block，并把p让给其他的g

### 提升压缩效率
#### Gorilla 压缩算法
1.按单个时间序列对数据进行分组。

2.按时间戳对每个时间序列的数据点进行排序。

3.使用增量编码对后续时间戳进行编码。这将用小偏差值代替大时间戳来代替后续数据点之间的间隔。与原始时间戳相比，偏差值用更小的比特数进行编码。

4.后续浮点值的异或二进制表示。Gorilla paper clams这通常给二进制值以大量的前导和尾随零位，这可以有效地压缩成更小的位数。

#### prometheus为什么用pull
1.开发更改时，可以在笔记本电脑上运行监视。
2.您可以手动转到目标并使用web浏览器检查其运行状况。
3.自由的在固定时间片处理数据（比方说60个target，可以一秒一个target进行拉取）

#### 查询过程
通过index查找到tsid，tsid找到相关的table， partition， block，先找到头信息，进行信息比对，然后找到具体的series
