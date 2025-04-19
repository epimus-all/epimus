# 背景
prometheus提供了promql这个强大而易于学习的工具，使得基于时序数据的查询变得简单，但是很多时候也带来了很多困扰。比如下面这个例子

```arkts
iops_ecs / iops_ecs_threshold > 90
```

上面这个promql里面，iops_ecs代表某台ecs实时iops时序指标，prometheus会实时采集对应指标的值；iops_ecs_threshold代表对应ecs的iops阈值，所以iops_ecs / iops_ecs_threshold可以理解为iops的使用率，如果这个使用率大于90，则进行告警。



一般来说iops_ecs_threshold跟iops_ecs一样的抓取，但是iops_ecs_threshold这个阈值一般很少变化，如果每次都进行抓取的话，会造成严重网络和存储上的浪费。如果是push数据的模式，则可以进行一个优化，比如每一个小时上传一次数据，通过last_over_time函数进行处理，那么promql则变成了

```arkts
iops_ecs / last_over_time(iops_ecs_threshold[1h]) > 90
```

数据量和网络带宽浪费的问题解决了，但是客户端仍然需要定时传输时序数据，一旦这个过程中出现了一次中断，就会造成一个小时的查询异常。



如果采取下面这种极端的例子的情况呢，只上传一次数据，然后100年都不用重新上传。

```arkts
iops_ecs / last_over_time(iops_ecs_threshold[36000d]) > 90
```

不过prometheus的默认最长存储时间只有15d，而且即便prometheus能够存储那么久的数据，promethesu内部按照每两个小时实例话一个Block，即使理论可行，这个查询效率也是及其低效的。



其实我们想要的效果是类似于下面的效果

```arkts
iops_ecs / last_over_time(iops_ecs_threshold) > 90
```

即没有指定时间范围的last_over_time函数，不过对于默认的prometheus存储来说还是有一定挑战的，要允许一部分低频指标通过**非时间范围**分区的方式来进行组织，这样一来就需要对默认的存储引擎做一定的调整。



# 设计
## 指标配置
指标配置的首要目的就是区分出哪些是「普通指标」，哪些是「非易变指标」



在配置文件中prometheus.yml中加入对应的内容

```yaml
metric_config:
  - name: iops_ecs_threshold
    type: stable
    undo_rentetion : 100 #最多保留100个值
```

## 存储引擎
![画板](1744958990680-8bc5e6be-b658-41d1-86c2-c02e08e30bc2.jpeg)

这里将存储分为原先的「时序存储」和将来要构建的「非时序存储」。



时序存储完全沿用之前的逻辑，基本分为三层compaction

1. 当head内存中的chuck数据达成<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">chunkRange*3/2的时候就会触发compaction，调用mmap函数落盘；</font>
2. <font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">当磁盘的Chuck达到两个小时的时间跨度的时候，会compact形成一个block，每个block都有自己专属的时间范围；</font>
3. <font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">当block数据多到一定程度的时候，会继续进行compaction，同时会处理数据乱序问题和block的时间范围重叠问题。</font>

<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);"></font>

<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">右边的非时序存储是本次新加入的，第一步mmap的compaction仍然沿用原先prometheus的步骤，但是第二步compaction的时候，首先判断是否是「非易变指标」，如果是则进行右边非时序存储后面的两步compaction</font>

2. 每次新加入的数据跟原先的Block做全量合并，然后生成一个新的Block，保证L0层的Block只有一个，由于每次都做全量合并，要求L0层的Block的大小不能太大。当L0层的block大到某个阈值的时候，则向下进行第三步的合并；
3. 将L0层的block和L1层的block进行合并，这个时候将会根据指标配置的undo_rentetion参数进行数据汰换，比如合并之后某个指标有150个数据点（sample），这个指标点的undo_retention设置为100，那么合并之后只会保留最近的100个数据点。



通过上面的设计可以看出，非时序存储的Block只有两个，任何时间点的指标查询只会去查询两个block，这个io开销是很小的。同时block的结构仍然保持跟原有的prometheus一致，这样改动也是非常的低。

## 查询逻辑
第一个改造跟上面的那个是一样的，就是查询的时候根据指标配置来判断是否是「非易变指标」，如果是的话，则切换到「非时序存储」的block上进行查询



```go
func (ng *Engine) populateSeries(ctx context.Context, querier storage.Querier, s *parser.EvalStmt) {
    // Whenever a MatrixSelector is evaluated, evalRange is set to the corresponding range.
    // The evaluation of the VectorSelector inside then evaluates the given range and unsets
    // the variable.
    var evalRange time.Duration

    parser.Inspect(s.Expr, func(node parser.Node, path []parser.Node) error {
        switch n := node.(type) {
            case *parser.VectorSelector:
            start, end := getTimeRangesForSelector(s, n, path, evalRange)
            interval := ng.getLastSubqueryInterval(path)
            if interval == 0 {
                interval = s.Interval
            }
            hints := &storage.SelectHints{
                Start: start,
                End:   end,
                Step:  durationMilliseconds(interval),
                Range: durationMilliseconds(evalRange),
                Func:  extractFuncFromPath(path),
            }
            evalRange = 0
            hints.By, hints.Grouping = extractGroupsFromPath(path)
            n.UnexpandedSeriesSet = querier.Select(ctx, false, hints, n.LabelMatchers...)

            case *parser.MatrixSelector:
            evalRange = n.Range
        }
        return nil
    })
```

也就是在24行的querier.Select的时候进行BlockReader的切换。



第二个改造就是涉及到核心的查询逻辑的更改，原先的查询逻辑大体如下

![画板](1744960921575-b0818120-0115-4a34-aa61-26114286e15b.jpeg)

打个比方这里有三个sample，就是那三个白点。这个时候promql来查询t0时刻指标的值，prometheus不会只是真的去查询t0这个时刻的点，而是向后搜寻lookbackDelta（默认是5）分钟，找到第一个符合要求的点。



对应到实现来说ChunckSeries的迭代器都实现了

```go
type Iterator interface {
	// Next advances the iterator by one and returns the type of the value
	// at the new position (or ValNone if the iterator is exhausted).
	Next() ValueType
	// Seek advances the iterator forward to the first sample with a
	// timestamp equal or greater than t. If the current sample found by a
	// previous `Next` or `Seek` operation already has this property, Seek
	// has no effect. If a sample has been found, Seek returns the type of
	// its value. Otherwise, it returns ValNone, after which the iterator is
	// exhausted.
	Seek(t int64) ValueType
```

通过Seek方法能够找到第一个大于t0的点，这个例子里面是没有的所以就是chunkenc.ValNone



第二步判断

```go
	if valueType == chunkenc.ValNone || t > refTime {
		var ok bool
		t, v, h, ok = it.PeekPrev()
		if !ok || t <= refTime-durationMilliseconds(ev.lookbackDelta) {
			return 0, 0, nil, false
		}
	}
```
it.PeekPrev()就是取上一个点的意思，chunkenc.ValNone的上一个点就是v3，如果t0 - 5m的时间戳是大于等于v3的话则返回nil，我们这个例子里面显然是不成立的，可以直接返回v3。



回到我们「非易变指标」查询逻辑上，lookbackDelta参数是肯定不需要的，我们仅仅需要找到最近的小于t0的点即可。

![画板](1744961223779-958adfac-4aec-4069-a774-11ff4aec95a9.jpeg)

回到程序实现上，原先的两步判断直接变成一步判断即可，就是第一步调用iterator.Seek()找到第一个大于t0的sample，然后返回它的上一个点it.PeekPrev()即可。




同时这里「非易变指标」本身就是变动较小的指标，即便储存最近的100个数据，存储的时间跨度也是非常的大，这样在存储成本和查询效率等各个方面都是大大的飞跃。

# 总结
让我们回到开头的那个例子，我们只需要配置好通过iops_ecs_threshold的「非易变指标」的属性，再通过push的方式上传一次关于iops阈值的数据，以后都可以直接通过简单的promql来进行查询了。而且这种方式相对于之前的搞法，在存储空间上有巨大进步，同时查询效率上也有质的飞跃。

```arkts
iops_ecs / iops_ecs_threshold > 90
```

