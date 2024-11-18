---
title: 1brc挑战
date: 2024-11-12 22:54:40
categories: 
- 有趣的项目
---
# 1brc挑战
[1brc](https://github.com/gunnarmorling/1brc) 是Gunnar Morling老哥发起的挑战，这位老哥看起来挺牛逼的。
![](images/My_Captures_17.png)
挑战的主要内容页很简单，就是对一个10亿行的文件，进行解析，求出每个城市温度的最大值，最小值和平均值。
![](images/My_Captures_18.png)
这幅图里面讲的非常清楚。
接下来我将讲一讲我自己的解决思路和方案。

# 一些前置处理
- 在`src/python`下提供了数据产生脚本`create_measurements.py <number>`。
- 官方测试数据集是433个城市，但是给的python脚本是10k，所以需要稍微修改下脚本。最后给的代码中修改好了！
- Programs are run from a RAM disk (i.o. the IO overhead for loading the file from disk is not relevant), 文件一开始就加载在内存当中。
- 所有的测试均在本人电脑上进行，cpu为`inter 12400 6核心12线程`，内存为`32G`。


# 最简单的原始处理。
首先是写了一个原始版本，我们仅是实现了相关逻辑。最后用时`Program execution time: 1m31.75228213s`。
```go
func main() {
	startTime := time.Now() // 记录开始时间
	f, _ := os.OpenFile("cpu.pprof", os.O_CREATE|os.O_RDWR, 0644)
	defer f.Close()
	pprof.StartCPUProfile(f)
	defer pprof.StopCPUProfile()

	if len(os.Args) != 2 {
		log.Fatalf("Missing measurements filename")
	}

	f, err := os.Open(os.Args[1])
	fi, err := f.Stat()
	size := fi.Size()
	data, err := syscall.Mmap(int(f.Fd()), 0, int(size), syscall.PROT_READ, syscall.MAP_SHARED)
	if err != nil {
		log.Fatalf("Error opening file: %v", err)
	}
	endTime := time.Now()             // 记录结束时间
	elapsed := endTime.Sub(startTime) // 计算程序执行时间
	fmt.Printf("data load into memeory time: %s\n", elapsed)
	/////////////////////////////////////////////////////////////////////////////////////////////////////

	// 1.deal data
	startTime = time.Now()
	nowData := dealData(data)

	// 2.sort by city
	ids := make([]string, len(nowData))
	i := 0
	for k, _ := range nowData {
		ids[i] = k
		i++
	}
	sort.Strings(ids)

	// 3. print sort by city
	fmt.Print("{")
	for i, id := range ids {
		if i > 0 {
			fmt.Print(",")
		}
		m := nowData[id]
		fmt.Printf("%s=%.1f/%.1f/%.1f", id, float64(m.min)/10.0, float64(m.sum)/10.0/float64(m.count), float64(m.max)/10.0)
	}
	fmt.Println("}")

	endTime = time.Now()
	elapsed = endTime.Sub(startTime) // 计算程序执行时间
	fmt.Printf("Program execution time: %s\n", elapsed)
}

func dealData(data []byte) map[string]*Result {
	var temp int64
	var city string
	start := 0
	number := false
	tmp := &Result{}
	ans := make(map[string]*Result)

	for i, v := range data {
		if v == '\n' {
			tmp = ans[city]
			tmp.min = min(temp, tmp.min)
			tmp.max = max(temp, tmp.max)
			tmp.count += 1
			tmp.sum += temp

			start = i + 1
			temp = 0
			number = false
		} else if v == ';' {
			city = string(data[start:i])
			if ans[city] == nil {
				ans[city] = &Result{
					min: 1_000_000_000,
					max: 0,
				}
			}
			number = true
		} else if number && v != '.' {
			temp = temp*10 + int64(data[i]-'0')
		}
	}

	return ans
}
```

# 优化1-分块并行处理data
首先我们考虑的是这里不应该上锁，锁的重量级是会严重影响竞争激烈的程序的，所以我们的思路应该是分治合并。所以考虑先通过多线程处理data后合并即可。这个时候我们只需要把前面的dealData 函数交给多个线程并行执行即可。
具体代码如下。
```go
func dealData(data []byte) map[string]*Result {
	nChunks := runtime.NumCPU()

	chunkSize := len(data) / nChunks
	if chunkSize == 0 {
		chunkSize = len(data)
	}

	chunks := make([]int, 0, nChunks)
	offset := 0
	for offset < len(data) {
		offset += chunkSize
		if offset >= len(data) {
			chunks = append(chunks, len(data))
			break
		}

		nlPos := bytes.IndexByte(data[offset:], '\n')
		if nlPos == -1 {
			chunks = append(chunks, len(data))
			break
		} else {
			offset += nlPos + 1
			chunks = append(chunks, offset)
		}
	}

	var wg sync.WaitGroup
	wg.Add(len(chunks))
	results := make([]map[string]*Result, len(chunks))
	start := 0
	for i, chunk := range chunks {
		go func(data []byte, i int) {
			results[i] = processChunk(data)
			wg.Done()
		}(data[start:chunk], i)
		start = chunk
	}
	wg.Wait()

	ans := make(map[string]*Result)
	for _, r := range results {
		for id, rm := range r {
			m := ans[id]
			if m == nil {
				ans[id] = rm
			} else {
				m.min = min(m.min, rm.min)
				m.max = max(m.max, rm.max)
				m.sum += rm.sum
				m.count += rm.count
			}
		}
	}
	return ans
}

func processChunk(data []byte) map[string]*Result {
	var temp int64
	var city string
	start := 0
	number := false
	tmp := &Result{}
	ans := make(map[string]*Result)

	for i, v := range data {
		if v == '\n' {
			tmp = ans[city]
			tmp.min = min(temp, tmp.min)
			tmp.max = max(temp, tmp.max)
			tmp.count += 1
			tmp.sum += temp

			start = i + 1
			temp = 0
			number = false
		} else if v == ';' {
			city = string(data[start:i])
			if ans[city] == nil {
				ans[city] = &Result{
					min: 1_000_000_000,
					max: 0,
				}
			}
			number = true
		} else if number && v != '.' {
			temp = temp*10 + int64(data[i]-'0')
		}
	}

	return ans
}
```
分块并行处理后时间很快的降到了13s。`Program execution time: 13.0s`。
# 优化2-设计以byte为key的map
这个时候我们来看看程序的火焰图，看看什么比较费时间。
![](images/My_Captures_19.png)

时间消耗的主要集中在`runtime.mapaccess1_faststr`和`runtime.slicebytetostring`。也就是processChunk中的这段代码。
```c
if v == '\n' {
	tmp = ans[city]
	tmp.min = min(temp, tmp.min)
	tmp.max = max(temp, tmp.max)
	tmp.count += 1
	tmp.sum += temp

	start = i + 1
	temp = 0
	number = false
} else if v == ';' {
	city = string(data[start:i])
	if ans[city] == nil {
		ans[city] = &Result{
			min: 1_000_000_000,
			max: 0,
		}
	}
	number = true
}
```


所以我们设计了如下的map。
```c
const (  
    hashFactor = 131  
    maxHash    = 1<<9 - 1  
)  
  
type Entity struct {  
    key   []byte  
    value *Result  
}  
type Result struct {  
    min, max, sum, count int64  
}  
  
// MyByteMap byteHash(byte) -> key -> valuetype MyByteMap struct {  
    Value [][]*Entity  
}  
  
func NewMyByteMap() *MyByteMap {  
    return &MyByteMap{  
       Value: make([][]*Entity, maxHash+1),  
    }  
}  
  
// Insert don't check key existfunc (m *MyByteMap) Insert(byteKey []byte, value *Result) {  
    key := m.GetKey(byteKey)  
    m.InsertByKey(key, byteKey, value)  
}  
  
func (m *MyByteMap) InsertByKey(key uint16, byteKey []byte, value *Result) {  
    if m.Value[key] == nil {  
       m.Value[key] = make([]*Entity, 0, 3)  
    }  
    for _, v := range m.Value[key] {  
       if bytes.Equal(v.key, byteKey) {  
          return  
       }  
    }  
  
    m.Value[key] = append(m.Value[key], &Entity{key: byteKey, value: value})  
}  
  
func (m *MyByteMap) FindByKey(key uint16, byteKey []byte) *Result {  
    if m.Value[key] == nil {  
       return nil  
    }  
  
    for _, v := range m.Value[key] {  
       if bytes.Equal(v.key, byteKey) {  
          return v.value  
       }  
    }  
  
    return nil  
}  
  
func (m *MyByteMap) Find(byteKey []byte) *Result {  
    key := m.GetKey(byteKey)  
    return m.FindByKey(key, byteKey)  
}  
func (m *MyByteMap) GetKey(data []byte) uint16 {  
    ans := uint16(0)  
    for i := 0; i < len(data); i++ {  
       ans = ans*hashFactor + uint16(data[i])  
       if ans > maxHash {  
          ans &= maxHash  
       }  
    }  
    return ans  
}
```
在插入433条数据，同时find 10亿次上进行对比（电脑跑10亿次太慢了）。
```
byteMap execution time: 1m12.103117669s
map execution time: 1m22.276880473s
```
这个map主要是优化了`byte -> string`，优化之后的时间。

最后的程序如下。这个时候的运行时间大概为`Program execution time: 6.9s`
```go
func processChunk(data []byte) map[string]*Result {
	// byte -> string need optimization
	var temp int64
	var city []byte
	start := 0
	number := false
	tmp := &Result{}
	tmpAns := NewMyByteMap()
	var key uint16

	for i, v := range data {
		if v == '\n' {
			tmp = tmpAns.FindByKey(key, city)
			tmp.min = min(temp, tmp.min)
			tmp.max = max(temp, tmp.max)
			tmp.count += 1
			tmp.sum += temp

			start = i + 1
			temp = 0
			number = false
		} else if v == ';' {
			city = data[start:i]
			key = tmpAns.GetKey(city)
			if tmpAns.FindByKey(key, city) == nil {
				tmpAns.InsertByKey(key, city, &Result{
					min: 1_000_000_000,
					max: 0,
				})
			}
			number = true
		} else if number && v != '.' {
			temp = temp*10 + int64(data[i]-'0')
		}
	}

	ans := make(map[string]*Result, len(tmpAns.Value))
	// tmpAns.Value -> initToByte
	for _, v := range tmpAns.Value {
		if v == nil {
			continue
		}
		for _, tv := range v {
			ans[string(tv.key)] = tv.value
		}
	}
	return ans
}
```

感觉后续的优化可能更多的是靠近于编译器层面的优化，可能更适合c语言来操作。对字节的直接操作可以省去更多方法的调用。

# 代码地址
https://cnb.cool/silky/1brc