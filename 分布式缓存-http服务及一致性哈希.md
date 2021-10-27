# 分布式缓存-http服务及一致性哈希

### http服务
    利用go语言提供的http库实现服务端
    
    需要实现ServeHTTP接口
    
```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

    定义HTTPPool,包含地址及url前缀字段
```go
type HTTPPool struct {
	self    string    // ip+端口
	baseUrl string    // url前缀
}
```

    实现ServeHTTP方法,主要包括解析url参数，获取Group实体及获取缓存值三步
    
```go
func (h *HTTPPool) ServeHTTP(resp http.ResponseWriter, req *http.Request) {
	url := req.URL.Path
	
	param, err := parseParams(url)

	group := GetGroup(param.GroupName)
	
	data, err := group.Get(param.Key)
	
	resp.Write(data.ByteSlice())
}
```


* * *

### 一致性哈希

    分布式缓存节点选取问题
    
    请求key在本机没有缓存值时，需要从远程结点获取，如果随机选择结点获取则大概率每次都需要从DB重新读缓存值
    
    通过哈希取余来选择结点可以保证相同的key每次都去相同结点获取缓存值，但如果有结点移除或新增时，会导致选择结点时发生错位，此时的请求大部分都要重新去DB获取缓存值，引起缓存雪崩
    
    通过一致性哈希算法解决该问题
    
    

* * *

    一致性哈希算法大致原理
    
    1. 一致性哈希算法通过一个叫作一致性哈希环的数据结构实现。这个环的起点是 0，终点是 2^32 - 1，并且起点与终点连接，故这个环的整数分布范围是 [0, 2^32-1]
    
    2. 将对象放置到哈希环，使用哈希函数计算这个对象的 hash 值，值的范围是 [0, 2^32-1]
    
 ![image](https://user-images.githubusercontent.com/46525758/139072226-d2ea1b93-2d82-416b-b6b6-b26a97dbb9fa.png)

    3.将服务器放置到哈希环，使用同样的哈希函数，我们将服务器也放置到哈希环上，可以选择服务器的 IP 或主机名作为键进行哈希，这样每台服务器就能确定其在哈希环上的位置

![image](https://user-images.githubusercontent.com/46525758/139072271-641d68d9-b50b-471b-b585-d4099adb564c.png)

    4.为对象选择服务器，将对象和服务器都放置到同一个哈希环后，在哈希环上顺时针查找距离这个对象的 hash 值最近的机器，即是这个对象所属的机器
    
![image](https://user-images.githubusercontent.com/46525758/139072309-6d4ff174-91ec-4c49-8607-dd092097f5a2.png)

    5.服务器增加的情况
    
![image](https://user-images.githubusercontent.com/46525758/139072330-65cb4bd9-503a-421a-b754-da4921fd77ec.png)

    6.服务器减少的情况

![image](https://user-images.githubusercontent.com/46525758/139072350-db0bbb88-3af4-4784-83de-20d5b5632b8e.png)

    一致性哈希算法，在新增/删除节点时，只需要重新定位该节点附近的一小部分数据，而不需要重新定位所有的节点
    
    数据倾斜问题：当服务器数量较小时，可能存在大部分key被分到某一台或者几台服务上，另外，增加新的服务器时，只能够分担相邻服务器的压力，而服务下线时，会造成相邻服务压力增大，有可能引发连锁宕机
    
    引入虚拟节点解决该问题
    
    将每台物理服务器虚拟为一组虚拟服务器，将虚拟服务器放置到哈希环上，如果要确定对象的服务器，需先确定对象的虚拟服务器，再由虚拟服务器确定物理服务器。

### 一致性哈希实现

    主体结构ConsistentHash, 包含自定义哈希函数，虚拟节点扩充倍数，哈希环，虚拟节点到真实节点的map
    
```go
type Hash func(key []byte) uint32

type ConsistentHash struct {
	hash              Hash           // 哈希函数
	replica           int            // 虚拟节点倍数
	virtualNodeValues []int          // 虚拟节点哈希值
	hashMap           map[int]string // 虚拟节点->节点map
}
```

    定义增加节点及根据key获取真实节点的方法
    
```go
func (c *ConsistentHash) Add(nodes ...string){
    // 真实节点+编号扩充虚拟节点
    // 计算虚拟节点哈希值
    // 虚拟节点哈希值加入哈希环中
    // 维护虚拟节点哈希值到真实节点映射关系
    // 哈希环值排序
}

func (c *ConsistentHash) Get(key string) string {
    // 计算key哈希值
    // 二分法在哈希环上查找第一个大于的虚拟节点
    // 获取虚拟节点对应的真实节点
}
```

代码地址：https://gitlab.com/itsRae/gocache/-/tree/day-3/consistenthash
