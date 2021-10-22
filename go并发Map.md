## GO并发编程
#### Mutex

    在并发编程中，如果程序中的一部分会被并发访问或修改，那么，为了避免并发访问导致的意想不到的结果，这部分程序需要被保护起来，这部分被保护起来的程序，就叫做临界区
    
    可以使用互斥锁，限定临界区只能同时由一个线程持有
    
    Go 标准库中，它提供了 Mutex 来实现互斥锁这个功能
    
    互斥锁 Mutex 就提供两个方法 Lock 和 Unlock
    
    当一个 goroutine 通过调用 Lock 方法获得了这个锁的拥有权后， 其它请求锁的 goroutine 就会阻塞在 Lock 方法的调用上，直到锁被释放并且自己获取到了这个锁的拥有权。
    
    易错点：
    1.Lock/Unlock 不是成对出现
    2.Copy 已使用的 Mutex
    3.Mutex 不是可重入的锁(怎么设计成可重入的待看)

```go
type Counter struct { 
    mu sync.Mutex 
    Count uint64
}
```

#### RWMutex
    Lock/Unlock：写操作时调用的方法。如果锁已经被 reader 或者 writer 持有，那么，Lock 方法会一直阻塞，直到能获取到锁；Unlock 则是配对的释放锁的方法。
    
    RLock/RUnlock：读操作时调用的方法。如果锁已经被 writer 持有的话，RLock 方法会一直阻塞，直到能获取到锁，否则就直接返回；而 RUnlock 是 reader 释放锁的方法。
    
#### WaitGroup
```go
func (wg *WaitGroup) Add(delta int) 
func (wg *WaitGroup) Done() 
func (wg *WaitGroup) Wait()
```
    Add，用来设置 WaitGroup 的计数值；
    Done，用来将 WaitGroup 的计数值减 1，其实就是调用了 Add(-1)；
    Wait，调用这个方法的 goroutine 会一直阻塞，直到 WaitGroup 的计数值变为 0。
    
#### Atomic
    Package sync/atomic 实现了同步算法底层的原子的内存操作原语，它提供了一些实现原子操作的方法
    
    atomic 为了支持 int32、int64、uint32、uint64、uintptr、Pointer（Add 方法不支持）类型，分别提供了 AddXXX、CompareAndSwapXXX、SwapXXX、LoadXXX、StoreXXX 等方法
    
    
```go
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)

func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)

func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)

func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)

```
    Value类型
    可以原子地存取对象类型，不能 CAS 和 Swap
    
```go
func (v *Value) Load() (x interface{})

func (v *Value) Store(x interface{}) 
```

### 并发安全Map
#### go原生Map
    不是协程安全的
    
```go
func TestMap(t *testing.T) {
	m := make(map[int64]int64, 0)
	go func() {
		for {
			m[1] = 1
		}
	}()
	go func() {
		for {
			_ = m[1]
		}
	}()
}
```

![image](https://user-images.githubusercontent.com/46525758/138409166-2a25d872-8b26-4da7-bb47-dc2e6e8a6e55.png)

#### 加读写锁的Map
WMS 项目已有,其中len()方法应该加锁
```go
type ConcurrentIntMap struct {
	Map  map[interface{}]interface{}
	Lock sync.RWMutex
}

func NewConnManger() *ConcurrentIntMap {
	cm := &ConcurrentIntMap{
		Map: make(map[interface{}]interface{}),
	}
	return cm
}

func (cm *ConcurrentIntMap) Add(id interface{}, value interface{}) {
	cm.Lock.Lock()
	defer cm.Lock.Unlock()
	cm.Map[id] = value
}

func (cm *ConcurrentIntMap) Remove(id interface{}) {
	cm.Lock.Lock()
	defer cm.Lock.Unlock()
	delete(cm.Map, id)
}
func (cm *ConcurrentIntMap) Get(id interface{}) (interface{}, error) {
	cm.Lock.RLock()
	defer cm.Lock.RUnlock()
	conn, ok := cm.Map[id]
	if !ok {
		return "", errors.New("connmanager get conn error ")
	}
	return conn, nil
}
```

#### 分段加锁Map
    直接加读写锁的Map加锁粒度太大，在大量并发读写的情况下，锁的竞争会非常激烈
  
    减少锁的粒度和锁的持有时间
    
    在第读写锁Map中，加锁的对象是整个 map，协程 A 对 map 中的 key 进行修改操作，会导致其它协程无法对其它 key 进行读写操作。可以将这个 map 分成 n 块，每个块之间的读写操作都互不干扰，从而降低冲突的可能性。
    
![image](https://user-images.githubusercontent.com/46525758/138409238-e0085853-6edf-4618-ba75-ae734af39477.png)

```go
// 分片Map
type SliceConcurrentMap struct {
	m map[string]interface{}
	sync.RWMutex
}
// 分片数
var SliceCount = 32

type ConcurrentMap2 []*SliceConcurrentMap

// 根据建值获取分片Map
func (s ConcurrentMap2) GetSliceMap(key string) *SliceConcurrentMap {}
```

    ConcurrentMap2 其实就是一个切片，切片的每个元素都是第一种方法中携带了读写锁的 map
    
    通过hash取模的方式找到当前访问的key处于哪一个分片之上，再对该分片进行加锁之后再读写
    
#### Sync.Map
    在内置的 sync 包中（Go 1.9+）也有一个线程安全的 map，通过将读写分离的方式实现了某些特定场景下的性能提升
    
    sync.map使用场景有一定局限性，要么是一写多读，要么是各个协程操作的 key 集合没有交集（或者交集很少），第二种是为什么？
    
    优化点：
    空间换时间。通过冗余的两个数据结构（只读的 read 字段、可写的 dirty），来减少加锁对性能的影响。对只读字段（read）的操作不需要加锁。
    
    优先从 read 字段读取、更新、删除，因为对 read 字段的读取不需要锁。
    
    动态调整。miss 次数多了之后，将 dirty 数据提升为 read，避免总是从 dirty 中加锁读取。double-checking。加锁之后先还要再检查 read 字段，确定真的不存在才操作 dirty 字段。
    
    延迟删除。删除一个键值只是打标记，只有在提升 dirty 字段为 read 字段的时候才清理删除的数据。
    
```go

type Map struct {
	mu Mutex    //互斥锁，用于锁定dirty map

	read atomic.Value //优先读map,支持原子操作

	dirty map[interface{}]*entry // dirty是一个当前最新的map，允许读写

	misses int // 主要记录read读取不到数据加锁读取read map以及dirty map的次数，当misses等于dirty的长度时，会将dirty复制到read
}

type readOnly struct {
    m       map[interface{}]*entry
    amended bool // 当dirty中包含read没有的数据时为true，比如新增一条数据
}

// expunged是用来标识此项已经删掉的指针
// 当map中的一个项目被删除了，只是把它的值标记为expunged，以后才有机会真正删除此项
var expunged = unsafe.Pointer(new(interface{}))

// entry代表一个值
type entry struct {
    // nil: 表示为被删除，调用Delete()可以将read map中的元素置为nil
	// expunged: 也是表示被删除，但是该键只在read而没有在dirty中，这种情况出现在将read复制到dirty中，即复制的过程会先将nil标记为expunged，然后不将其复制到dirty
	//  其他: 表示存着真正的数据
    p unsafe.Pointer // *interface{}
}
```

    atomic.Value 这个类型有两个公开的指针方法，Load 和 Store 。
    
    Load 方法用于原子地的读取原子值实例中存储的值，
    
    Store 方法用于原子地在原子值实例中存储一个值，它接受一个 interface{} 类型的参数
    
    通过引入两个map将读写分离到不同的map，其中read map提供并发读和已存元素原子写，而dirty map则负责读写。 
    
    read map就可以在不加锁的情况下进行并发读取,当read map中没有读取到值时,再加锁进行后续读取,并累加未命中数。
    
    当未命中数大于等于dirty map长度,将dirty map上升为read map。
    
    两个map，但是底层数据存储的是指针，指向的是同一份值。
    
    执行过程如下：
    
 1. 初始化Map
 2. 写入值{X:1,Y:2,Z:3}
![image](https://user-images.githubusercontent.com/46525758/138409343-02bc0e45-4e1d-4c7a-bf70-244a59d06096.png)
3. 读数据，read中不存在，加锁读dirty，并累计未命中次数
4. 未命中次数大于等于dirty长度时，提升dirty为read
![image](https://user-images.githubusercontent.com/46525758/138409381-5c848bcc-420c-4948-b39c-173cbca35fad.png)
5. 更新元素时，先尝试更新read
![image](https://user-images.githubusercontent.com/46525758/138409405-24495177-c588-4430-9313-4216d610690f.png)
6.新增元素全部写到dirty
![image](https://user-images.githubusercontent.com/46525758/138409424-eadbe1f1-e82e-4912-a27e-5971e3448a85.png)

在 dirty 重新生成新的时候，是需要遍历整个 read，这里会消耗很多性能
