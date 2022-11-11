# go_concurrent_map

这个类是改造的别人的一个实现：https://github.com/orcaman/concurrent-map
但是这个版本虽然使用了泛型，但是key强制使用字符串，
众所周知，在数据量比较大的时候（10， 000，000），使用大量的字符串会对GC造成很大的影响，
所以，我将这个类改造为纯泛型，这样就可以使用uint64作为主键；

基础用法并没有改变，比如我要实现一个计数器，


  // Create a new map.
	m := cache.NewConcurrentMap[uint64, string]()

	// Sets item within map, sets "bar" under key "foo"
	m.Set(199010212, "bar")

	// Retrieve item from map.
	bar, ok := m.Get(199010212)
	fmt.Println(bar, ok)


假如我需要实现几个计数器，实现并发安全更新计数，
// 计数器
type BaseCounter struct {
	Count     uint64
	CountLast uint64
}
我们需要使用其中带回调的更新函数：

// 定义一个全局变量
var MapOfAppVistedCount ConcurrentMap[uint64, *BaseCounter]

// 初始化
func InitMaps() {
	MapOfAppVistedCount = NewConcurrentMap[uint64, *BaseCounter]()
}

// 会在更新时候加锁期间回调
// 没有值，则设置；如果有，则更新; 新增的部分通过新的值传递过来！
func appAddCallBack(exist bool, valueInMap *BaseCounter, newValue *BaseCounter) *BaseCounter {
	if exist == false {
		return newValue
	} else {
		valueInMap.Count += newValue.Count
		return valueInMap
	}
}

// 对应用计数器加i
func AppAddBy(key uint64, i uint64) uint64 {
	c := BaseCounter{i, i}
	res := MapOfAppVistedCount.Upsert(key, &c, appAddCallBack)
	if res != nil {
		return res.Count
	}
	return 0
}

使用时候就是这样的：
  cache.InitMaps()
  count := cache.AppAddBy(1, 1)
	fmt.Println(count)
	count = cache.AppAddBy(1, 2)
	fmt.Println(count)
	count = cache.AppAddBy(1, 3)
	fmt.Println(count)
  
  
