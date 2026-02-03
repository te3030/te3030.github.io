# GoLang实现基于本地文件的海量数据存储和读取
基于 MMAP(Memory mapping)技术，将硬盘映射到内存，轻松映射超过 100G 硬盘空间至内存（不影响实际内存占用），相当于无限内存。

<!--more-->

开源项目：https://github.com/vito-go/msearch/

> 本文添加个人理解的注释，未修改代码

## README main

```golang
package main

import (
	"fmt"
	"github.com/vito-go/msearch"
)

func main() {
	fileName := "test.msearch" // 底层文件名
	ms, err := msearch.NewMsearch(fileName, 0)
	if err != nil {
		panic(err)
	}
	user := "example@example.com"
	userLi := "userLi@example.com"
	values := []string{"abc429298@example.com", "abc429179@example.com", "abc429178@example.com", "abc429177@example.com", "abc429176@example.com", "kadhx11@example.com", "kadhx1@example.com", "1101010022@example.com"}
	err = ms.Add(user, values...) // 添加一组values
	if err != nil {
		panic(err)
	}
	fmt.Println(ms.Get(user))               // 获取一个key对应的一组values
	// 添加一个value
	if err=ms.Add(userLi, "aaa@163.com|nickname1");err!=nil{
		panic(err)
	}
	ms.Del(user, values[:len(values)-2]...) // 删除多个value
	fmt.Println(ms.Get(user))               // 获取一组values
	fmt.Println(ms.Get(`userNil`))          // 获取一组values,如果没有values返回空数组。
}
```

## 主要方法

部分注释个人添加，方便理解

```golang
//go:build !windows

// Package msearch  基于mmap技术的，以本地文件为基础的搜索技术。提供增加、删、查（简单的替代mysql。）
// 单个 value 长度不能超过255. // TODO if needed?
// [_8(total) _1 key  _1(len) xxx _1(len) xxx  _8(overflow offset)]

package msearch

import (
	"encoding/binary"
	"errors"
	"os"
	"strings"
	"sync"
	"syscall"
)

// notExist 标记不存在的key. // TODO 好像这个标记没什么用
const notExist = -1

// DefaultLength 默认映射空间大小 64 GB，不影响实际内存大小。
const DefaultLength = 64 << 30

type MSearcher interface {
	Add(key string, values ...string) error
	Del(key string, values ...string)
	Get(key string) []string
	DelByPrefix(key string, values ...string)
	Update(key string, values ...string) error
	Exist(key string) bool
}

// Msearch  It's safe for concurrent use by multiple goroutines.
type Msearch struct {
	mu sync.RWMutex // mu to protect the follow fields
	f  *os.File     // After the syscall.Mmap() call has returned, the file descriptor, fd, can be closed immediately
	// without invalidating the mapping. But after f.Close(), we can't write any data to the file.
	// So, the f should not call Close().
	offset    int            // last offset of the f
	keyMap    map[string]int // store all keys, value is the offset in bytesAddr of every key
	bytesAddr []byte         // bytesAddr is the virtual address space of the process
}

// NewMsearch create a new Msearch by file and length。
// file is the path of the underlying file.
// the length argument specifies the length of the mapping (which must be greater than 0)
// it has no impact on the real memory. the default value is 64GB.
func NewMsearch(file string, length int) (*Msearch, error) {
	f, err := os.OpenFile(file, os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		return nil, err
	}
	if length <= DefaultLength {
		length = DefaultLength
	}
	// 追加用f.Write 读取和修改用MMap
	bytesAddr, err := syscall.Mmap(int(f.Fd()), 0, length, syscall.PROT_WRITE|syscall.PROT_READ, syscall.MAP_SHARED)
	if err != nil {
		return nil, err
	}
	return &Msearch{
		mu:        sync.RWMutex{},
		f:         f,
		offset:    0,
		keyMap:    make(map[string]int, 1<<10),
		bytesAddr: bytesAddr,
	}, nil
}

// Get one or more value.
func (s *Msearch) Get(key string) []string {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return s.gets(key)
}

// Add one or more value.
func (s *Msearch) Add(key string, values ...string) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.adds(key, values...)
}

// Del one or more value.
func (s *Msearch) Del(key string, values ...string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.dels(key, values...)
}

// DelByPrefix delete values by prefix.
func (s *Msearch) DelByPrefix(key string, values ...string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.delsPrefix(key, values...)
}

// Update will delete all the old values of key and set it to the newValues.
func (s *Msearch) Update(key string, newValues ...string) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	oldValues := s.gets(key)
	s.dels(key, oldValues...)
	err := s.adds(key, newValues...)
	return err
}

// Exist check the key whether exists.
func (s *Msearch) Exist(key string) bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	if offset, ok := s.keyMap[key]; ok && offset != notExist {
		return true
	}
	s.keyMap[key] = notExist
	return false
}

func (s *Msearch) delsPrefix(key string, values ...string) {
	offset, ok := s.keyMap[key]
	if !ok {
		return
	}

	if len(values) == 0 {
		return
	}
	for {
		d := s.delPrefix(offset, values...)
		if d == 0 {
			break
		}
		offset = d
	}
}

func (s *Msearch) dels(key string, values ...string) {
	offset, ok := s.keyMap[key]
	if !ok {
		return
	}
	valueMap := make(map[string]struct{}, len(values))
	for _, value := range values {
		valueMap[value] = struct{}{}
	}
	if len(valueMap) == 0 {
		return
	}
	for {
		d := s.del(offset, valueMap)
		if d == 0 {
			break
		}
		offset = d
	}
}

func (s *Msearch) gets(key string) []string {
	offset, ok := s.keyMap[key]
	if !ok || offset == notExist {
		return nil
	}
	var lists []string
	var d int
	for {
		var list []string
		list, d = s.get(offset)
		lists = append(lists, list...)
		if d == 0 {
			break
		}
		offset = d
	}
	return lists
}

// empty 插入判断是否有空位，以及空位的长度.
func (s *Msearch) empty(offset int) (o int, start int, end int, t bool) {
	var lastDec int
	for {
		o, lastDec, start, end, t = s.empty1(offset)
		if lastDec == 0 || t {
			break
		}
		offset = lastDec
	}
	return
}

// getB8byOffset 这个offset是每个value的起始offset 得到最后的一个8位 offset只能通过s.keyMap 获得。
func (s *Msearch) getB8byOffset(offset int) (b8 []byte) {
	var lastDec int
	for {
		lastDec, b8 = s.b8(offset)
		if lastDec == 0 {
			break
		}
		offset = lastDec
	}
	return
}

// empty1 是否有空位，以及空位的长度.
func (s *Msearch) empty1(offset int) (o int, lastDec int, start int, end int, t bool) {
	// t为false的时候 也就是没有空位 有b8
	var first bool
	// 获取kv的前8位，得到数据的总长度
	total := bigUint64(s.bytesAddr[offset : offset+8])
	// 获取kv的全部数据
	b := s.bytesAddr[offset : offset+total]
	o = offset
	for i := int(b[8] + 1 + 8); i < len(b[:len(b)-8]); {
		// value长度为0
		if b[i] == 0 {
			if !first {
				// 第一次遇到空值时候
				first = true
				t = true
				start = i
			}
			i++
			continue
		}
		if t {
			end = i
			return
		}
		i += int(b[i]) + 1
	}
	if t && end == 0 {
		end = total - 8
	}
	lastDec = bigUint64(b[total-8 : total])
	return
}

func (s *Msearch) b8(offset int) (lastDec int, b8 []byte) {
	// t为false的时候 也就是没有空位 有b8
	if offset >= s.offset {
		// 当前的偏移量已超过总的数据长度，说明没有位置了，返回nil，后面会追加
		return 0, nil
	}
	// 拿到当前offset的后面（具体是后面8是下一次kv的总长度）的值，即next的lens
	// 获取当前kv在磁盘的总长度
	total := bigUint64(s.bytesAddr[offset : offset+8])
	// 获取数据的最后8位，
	b8 = s.bytesAddr[offset+total-8 : offset+total]
	// 获取全部数据
	b := s.bytesAddr[offset : offset+total]
	// 截取最后8位数据
	lastDec = bigUint64(b[total-8 : total])
	return
}

// data = {
// [0 - 7] : 当前kv的总长度
// [8] : key字符串的长度
// [9 - len(key)] : 字符串key内容
// [9+len(key)+n*len(value) - 9+len(key)+(n+1)*len(next_value)] : 每个value的内容
// [ -8 : ] : 最后8位是kv链表的下一链
// }
func (s *Msearch) add(b8 []byte, key string, values ...string) (int, error) {
	// 初始化一块地方
	var b = make([]byte, 1<<10)
	// 保存key字符串的长度
	b[8] = byte(len(key))
	// 保存在[9 - len(key)+8 ]的key字符串内容
	n := copy(b[9:], key)
	// 计算idx的位置，做为存储values的开始
	idx := n + 1 + 8
	for _, value := range values {
		if len(b) < idx+len(value)+2 {
			// 容量不足就扩容 扩容一定要覆盖下面的copy
			b = append(b, make([]byte, 1<<10)...)
		}
		// todo len(value)大于255？？
		if len(value) > 255 {
			return 0, errors.New("value exceed max length 255")
		}
		// 保存这个value字符串的长度
		b[idx] = byte(len(value))
		// 保存value字符串内容
		// 一定要注意copy的地方
		copy(b[idx+1:], value)
		idx += 1 + len(value)
	}
	// 总数据长度+8字节保留
	total := idx + 8
	// binary.BigEndian.PutUint64(b[idx:], uint64(total+s.offset)) // todo 是否有必要？？
	// 切片所需数据的完整长度
	b = b[:total]
	// 设置total到前8位，作为当前kv的总长度
	binary.BigEndian.PutUint64(b[:8], uint64(total))
	// 写到文件
	_, err := s.f.Write(b)
	if err != nil {
		return 0, err
	}
	if i, ok := s.keyMap[key]; !ok || i == notExist {
		// 如果key不存在，就将添加key 的map中，value为当前数据的在总位置上的偏移量（第一个是0）
		s.keyMap[key] = s.offset
	}
	if len(b8) > 0 {
		// b8是上一链的结尾
		// 末尾的
		binary.BigEndian.PutUint64(b8, uint64(s.offset))
	}
	// 计算为下一个的kv的开始偏移量
	s.offset += total
	return total, err
}
func (s *Msearch) adds(key string, values ...string) error {
	if len(values) == 0 {
		return nil
	}
	offset, ok := s.keyMap[key]
	// 不存在
	if !ok || offset == notExist {
		// 不存在时候，需要直接追加写入磁盘
		_, err := s.add(nil, key, values...)
		return err
	}
	// t 是否能插空 插空进入
	// s.bytesAddr[offset:offset+8]
	if len(values) == 1 {
		value := values[0]
		o, start, end, t := s.empty(offset)
		if t && len(value) < (end-start) {
			total := bigUint64(s.bytesAddr[offset : offset+8])
			b := s.bytesAddr[o : o+total]
			b[start] = byte(len(value))
			copy(b[start+1:], value)
			return nil
		}
	}
	b8 := s.getB8byOffset(offset)
	// 存在但是有多个value值 或者 没有空的地方插入时候，需要直接追加写入磁盘
	_, err := s.add(b8, key, values...)
	return err
}

func (s *Msearch) del(offset int, valueMap map[string]struct{}) int {
	total := bigUint64(s.bytesAddr[offset : offset+8])
	if total == 0 {
		return 0
	}
	// 全部数据
	b := s.bytesAddr[offset : offset+total]
	// 遍历value数据字节
	for i := int(b[8] + 1 + 8); i < len(b[:len(b)-8]); {
		bi := int(b[i])
		if bi == 0 {
			// 跳过空数据
			i++
			continue
		}
		// 构建value值
		value := string(b[i+1 : i+1+int(b[i])])
		// value在需要删除的列表里
		// 这里为什么使用map保存数据，因为这边要遍历所有value，map自带的算法更快
		if _, ok := valueMap[value]; ok {
			// 将value数据copy位0值
			copy(b[i+1:i+1+int(b[i])], make([]byte, int(b[i])))
			// 将长度设置为0值
			b[i] = 0
		}
		// 	跳到下一个value位置
		i += bi + 1
	}
	// 返回结尾的8位数据
	return bigUint64(b[total-8 : total])
}
func (s *Msearch) delPrefix(offset int, values ...string) int {
	total := bigUint64(s.bytesAddr[offset : offset+8])
	if total == 0 {
		return 0
	}
	b := s.bytesAddr[offset : offset+total]
	for i := int(b[8] + 1 + 8); i < len(b[:len(b)-8]); {
		bi := int(b[i])
		if bi == 0 {
			i++
			continue
		}
		value := string(b[i+1 : i+1+int(b[i])])
		// 这里为什么手动遍历，因为自己遍历有更多的判断和选择
		for _, v := range values {
			// 字符串是否以前缀开头
			if strings.HasPrefix(value, v) {
				copy(b[i+1:i+1+int(b[i])], make([]byte, int(b[i])))
				b[i] = 0
			}
		}
		i += bi + 1

	}
	return bigUint64(b[total-8 : total])
}

// 获取这一链的全部数据，并返回value list和下一链offset
func (s *Msearch) get(offset int) ([]string, int) {
	// 数据长度
	total := bigUint64(s.bytesAddr[offset : offset+8])
	// 全部数据
	b := s.bytesAddr[offset : offset+total]
	var list []string
	for i := int(b[8] + 1 + 8); i < len(b[:len(b)-8]); {
		if b[i] == 0 {
			// 0是表示这个空间无意义嘛？是被删除了设置为0值嘛？
			i++
			continue
		}
		list = append(list, string(b[i+1:i+1+int(b[i])]))
		i += int(b[i]) + 1
	}
	lastDec := bigUint64(b[total-8 : total])
	return list, lastDec
}

// bigUint64 对大数字进行解码 长度为0-8位的字节切片. binary.BigEndian.PutUint64 是编码.
// Deprecated: please use binary.BigEndian.Uint64
func bigUint64(buf []byte) int {
	if len(buf) > 8 {
		return 0
	}
	var x int
	for _, b := range buf {
		x = x<<8 | int(b)
	}
	return x
}
```


## 个人修改版本
```golang

//go:build !windows

// Package msearch  基于mmap技术的，以本地文件为基础的搜索技术。提供增加、删、查（简单的替代mysql。）
// 单个 value 长度不能超过255. // TODO if needed?
// [_8(total) _1 key  _1(len) xxx _1(len) xxx  _8(overflow offset)]

package main

import (
	"encoding/binary"
	"errors"
	"fmt"
	"os"
	"strings"
	"sync"
	"syscall"
	"time"
)

// notExist 标记不存在的key. // TODO 好像这个标记没什么用
const notExist = -1

// DefaultLength 默认映射空间大小 64 GB，不影响实际内存大小。
const DefaultLength = 64 << 30

type MSearcher interface {
	Add(key string, values ...string) error
	Del(key string, values ...string)
	Get(key string) []string
	DelByPrefix(key string, values ...string)
	Update(key string, values ...string) error
	Exist(key string) bool
}

// Msearch  It's safe for concurrent use by multiple goroutines.
type Msearch struct {
	mu sync.RWMutex // mu to protect the follow fields
	f  *os.File     // After the syscall.Mmap() call has returned, the file descriptor, fd, can be closed immediately
	// without invalidating the mapping. But after f.Close(), we can't write any data to the file.
	// So, the f should not call Close().
	offset    int            // last offset of the f
	keyMap    map[string]int // store all keys, value is the offset in bytesAddr of every key
	bytesAddr []byte         // bytesAddr is the virtual address space of the process
}

func openMsearch(file string, length int) (*Msearch, error) {
	f, err := os.OpenFile(file, os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		return nil, err
	}
	fInfo, err := f.Stat()
	if err != nil {
		return nil, err
	}
	if length <= DefaultLength {
		length = DefaultLength
	}

	// 追加用f.Write 读取和修改用MMap
	bytesAddr, err := syscall.Mmap(int(f.Fd()), 0, length, syscall.PROT_WRITE|syscall.PROT_READ, syscall.MAP_SHARED)
	if err != nil {
		return nil, err
	}
	var keyMap map[string]int = make(map[string]int, 1<<10)
	if length > 0 {
		// _ = fInfo
		keyMap = restart(int(fInfo.Size()), bytesAddr)
	}
	a := &Msearch{
		mu:        sync.RWMutex{},
		f:         f,
		offset:    0,
		keyMap:    keyMap,
		bytesAddr: bytesAddr,
	}
	// a.empty1(782944)
	return a, nil
}

// NewMsearch create a new Msearch by file and length。
// file is the path of the underlying file.
// the length argument specifies the length of the mapping (which must be greater than 0)
// it has no impact on the real memory. the default value is 64GB.
func NewMsearch(file string, length int) (*Msearch, error) {
	bakFile := file + "." + time.Now().Format("2006_01_02_15_04_05")
	os.Rename(file, bakFile)
	m, err := openMsearch(bakFile, length)
	if err != nil {
		return nil, err
	}
	relM, err := openMsearch(file, length)
	if err != nil {
		return nil, err
	}
	for k, _ := range m.keyMap {
		values := m.Get(k)
		relM.Add(k, values...)
	}
	m.f.Close()
	return relM, err
}

func restart(fileSize int, bytesAddr []byte) map[string]int {
	var keyMap map[string]int = make(map[string]int, 1<<10)
	// return keyMap

	var offset int
	for offset < fileSize {
		// 读取一个数据长度
		total := bigUint64(bytesAddr[offset : offset+8])
		keyLen := bigUint64(bytesAddr[offset+8 : offset+12])
		if keyLen != 0 {
			key := bytesAddr[offset+12 : offset+12+keyLen]
			// valueLen := bigUint64(bytesAddr[offset+12+keyLen : offset+12+keyLen+4])
			// value := bytesAddr[offset+12+keyLen+4 : offset+12+keyLen+4+valueLen]
			// _ = value
			// // 读取一个数据
			// b := bytesAddr[offset : offset+total]
			// var list []string
			// key := b[ 9 : 9+int(b[8])]
			// value := bytesAddr[ 9+int(b[8]) : ]
			// for i := int(b[8] + 1 + 8); i < len(b[:len(b)-8]); {
			// 	if b[i] == 0 {
			// 		// 0是表示这个空间无意义嘛？是被删除了设置为0值嘛？
			// 		i++
			// 		continue
			// 	}
			// 	list = append(list, string(b[i+1:i+1+int(b[i])]))
			// 	i += int(b[i]) + 1
			// }
			// lastDec := bigUint64(b[total-8 : total])
			// return list, lastDec
			// fmt.Printf("key = %#v \n",string(key))
			// fmt.Printf(" value = %#v \n",string(value))
			// fmt.Printf("offset = %#v \n",offset)
			// return nil
			// if i, ok := s.keyMap[key]; !ok || i == notExist {
			_, ok := keyMap[string(key)]
			if !ok {
				keyMap[string(key)] = offset
			}
		}
		offset += total
	}
	return keyMap
}

func (s *Msearch) Print() {
	s.mu.RLock()
	defer s.mu.RUnlock()
	fmt.Printf("会话数据：%#v\n", s.keyMap)
}

// Get one or more value.
func (s *Msearch) Get(key string) []string {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return s.gets(key)
}

// Add one or more value.
func (s *Msearch) Add(key string, values ...string) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.adds(key, values...)
}

// Del one or more value.
func (s *Msearch) Del(key string, values ...string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.dels(key, values...)
}

// DelByPrefix delete values by prefix.
func (s *Msearch) DelByPrefix(key string, values ...string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.delsPrefix(key, values...)
}

// Update will delete all the old values of key and set it to the newValues.
func (s *Msearch) Update(key string, newValues ...string) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	oldValues := s.gets(key)
	s.dels(key, oldValues...)
	err := s.adds(key, newValues...)
	return err
}

// Exist check the key whether exists.
func (s *Msearch) Exist(key string) bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	if offset, ok := s.keyMap[key]; ok && offset != notExist {
		return true
	}
	s.keyMap[key] = notExist
	return false
}

func (s *Msearch) delsPrefix(key string, values ...string) {
	offset, ok := s.keyMap[key]
	if !ok {
		return
	}

	if len(values) == 0 {
		return
	}
	for {
		d := s.delPrefix(offset, values...)
		if d == 0 {
			break
		}
		offset = d
	}
}

func (s *Msearch) dels(key string, values ...string) {
	offset, ok := s.keyMap[key]
	if !ok {
		return
	}
	valueMap := make(map[string]struct{}, len(values))
	for _, value := range values {
		valueMap[value] = struct{}{}
	}
	if len(valueMap) == 0 {
		return
	}
	for {
		d := s.del(offset, valueMap)
		if d == 0 {
			break
		}
		offset = d
	}
}

func (s *Msearch) gets(key string) []string {
	offset, ok := s.keyMap[key]
	if !ok || offset == notExist {
		return nil
	}
	var lists []string
	var d int
	for {
		var list []string
		list, d = s.get(offset)
		lists = append(lists, list...)
		if d == 0 {
			break
		}
		offset = d
	}
	return lists
}

// getB8byOffset 这个offset是每个value的起始offset 得到最后的一个8位 offset只能通过s.keyMap 获得。
func (s *Msearch) getB8byOffset(offset int) (b8 []byte) {
	var lastDec int
	for {
		lastDec, b8 = s.b8(offset)
		if lastDec == 0 {
			break
		}
		offset = lastDec
	}
	return
}

func (s *Msearch) b8(offset int) (lastDec int, b8 []byte) {
	// t为false的时候 也就是没有空位 有b8
	if offset >= s.offset {
		// 当前的偏移量已超过总的数据长度，说明没有位置了，返回nil，后面会追加
		return 0, nil
	}
	// 拿到当前offset的后面（具体是后面8是下一次kv的总长度）的值，即next的lens
	// 获取当前kv在磁盘的总长度
	total := bigUint64(s.bytesAddr[offset : offset+8])
	// 获取数据的最后8位，
	b8 = s.bytesAddr[offset+total-8 : offset+total]
	lastDec = bigUint64(b8)
	return
}

// data = {
// [0 - 7] : 当前kv的总长度
// [8] : key字符串的长度
// [9 - len(key)] : 字符串key内容
// [9+len(key)+n*len(value) - 9+len(key)+(n+1)*len(next_value)] : 每个value的内容
// [ -8 : ] : 最后8位是kv链表的下一链
// }
func (s *Msearch) add(b8 []byte, key string, value string) (int, []byte, error) {
	// 初始化一块地方
	var b = make([]byte, 1<<10)
	// 保存key字符串的长度
	{

		keyLen := len(key)
		// b[8] = uint8(keyLen)
		// b[9] = uint8(keyLen >> 8)
		// b[10] = uint8(keyLen >> 16)
		// b[11] = uint8(keyLen >> 24)
		b[8] = uint8(keyLen >> 24)
		b[9] = uint8(keyLen >> 16)
		b[10] = uint8(keyLen >> 8)
		b[11] = uint8(keyLen)
	}
	// 保存在[9 - len(key)+11 ]的key字符串内容
	n := copy(b[12:], key)
	// 计算idx的位置，做为存储values的开始
	idx := n + 12
	// fmt.Printf("1 %#v, %#v, %#v\n", cap(b), len(b), idx)
	// for _, value := range values {
	for true {
		if len(b) < idx+len(value)+12 {
			// 容量不足就扩容 扩容一定要覆盖下面的copy
			b = append(b, make([]byte, 1<<10)...)
			// fmt.Printf("2 %#v, %#v, %#v , %#v\n", cap(b), len(b), idx, len(value))
		} else {
			break
		}
	}
	// 因为字符串长度为1个字节保存 len(value)不能大于255
	// if len(value) > 255 {
	// return 0, errors.New("value exceed max length 255")
	// }
	// 保存这个value字符串的长度
	valueLen := len(value)
	// b[idx] = byte(len(value))
	b[idx] = uint8(valueLen >> 24)
	b[idx+1] = uint8(valueLen >> 16)
	b[idx+2] = uint8(valueLen >> 8)
	b[idx+3] = uint8(valueLen)
	idx += 3
	// 保存value字符串内容
	// 一定要注意copy的地方
	copy(b[idx+1:], value)
	idx += 1 + len(value)
	// fmt.Printf("3 %#v, %#v, %#v\n", cap(b), len(b), idx)
	// }
	// fmt.Printf("4 %#v, %#v, %#v\n", cap(b), len(b), idx)
	// 总数据长度+8字节保留
	total := idx + 8
	// binary.BigEndian.PutUint64(b[idx:], uint64(total+s.offset)) // todo 是否有必要？？
	// 切片所需数据的完整长度
	b = b[:total]
	// 设置total到前8位，作为当前kv的总长度
	binary.BigEndian.PutUint64(b[:8], uint64(total))
	// 写到文件
	_, err := s.f.Write(b)
	if err != nil {
		return 0, nil, err
	}
	if i, ok := s.keyMap[key]; !ok || i == notExist {
		// 如果key不存在，就将添加key 的map中，value为当前数据的在总位置上的偏移量（第一个是0）
		s.keyMap[key] = s.offset
	}
	if len(b8) > 0 {
		// b8是上一链的结尾
		// 末尾的
		binary.BigEndian.PutUint64(b8, uint64(s.offset))
	}
	// 计算为下一个的kv的开始偏移量
	s.offset += total
	return total, b[total-8:], err
}
func (s *Msearch) adds(key string, values ...string) error {
	if len(values) == 0 {
		return nil
	}
	for _, v := range values {
		if len(v) >= 1024*1024*1024*4 {
			return errors.New("value exceed max length 4294967295 ")
		}
	}
	offset, ok := s.keyMap[key]
	var err error
	// 不存在
	if !ok || offset == notExist {
		// 不存在时候，需要直接追加写入磁盘
		var b8 []byte
		for _, v := range values {
			_, b8, err = s.add(b8, key, v)
			if err != nil {
				return err
			}
		}
	}
	// t 是否能插空 插空进入
	// s.bytesAddr[offset:offset+8]
	// if len(values) == 1 {
	// 	value := values[0]
	// 	o, start, end, t := s.empty(offset)
	// 	if t && len(value) < (end-start) {
	// 		total := bigUint64(s.bytesAddr[offset : offset+8])
	// 		b := s.bytesAddr[o : o+total]
	// 		b[start] = byte(len(value))
	// 		copy(b[start+1:], value)
	// 		return nil
	// 	}
	// }
	b8 := s.getB8byOffset(offset)
	// 存在但是有多个value值 或者 没有空的地方插入时候，需要直接追加写入磁盘
	for _, v := range values {
		_, b8, err = s.add(b8, key, v)
		if err != nil {
			return err
		}
	}
	return err
}

func (s *Msearch) del(offset int, valueMap map[string]struct{}) int {
	total := bigUint64(s.bytesAddr[offset : offset+8])
	if total == 0 {
		return 0
	}
	// 全部数据
	b := s.bytesAddr[offset : offset+total]
	// 遍历value数据字节

	// bytesBuffer := bytes.NewBuffer(b[8:12])
	// var i32 int32
	// binary.Read(bytesBuffer, binary.BigEndian, &i32)
	i := bigUint64(b[8:12])
	if total < i+12 {
		// ????
		panic("数据长度超过总长度了")
	}
	valueLen := bigUint64(b[8+4+i : 8+4+i+4])
	if valueLen == 0 {
		// 跳过空数据
		return 0
	}
	// 构建value值
	value := string(b[8+4+i+4 : 8+4+i+4+valueLen])
	// value在需要删除的列表里
	// 这里为什么使用map保存数据，因为这边要遍历所有value，map自带的算法更快
	if _, ok := valueMap[value]; ok {
		// 将value数据copy位0值
		copy(b[8:total-8], make([]byte, total-8))
		// 将长度设置为0值
		// copy(b[8:12], make([]byte, 4))
	}
	// 返回结尾的8位数据
	return bigUint64(b[total-8 : total])
}
func (s *Msearch) delPrefix(offset int, values ...string) int {
	total := bigUint64(s.bytesAddr[offset : offset+8])
	if total == 0 {
		return 0
	}
	b := s.bytesAddr[offset : offset+total]

	i := bigUint64(b[8:12])
	if total < i+12 {
		// ????
		panic("数据长度超过总长度了")
	}
	valueLen := bigUint64(b[8+4+i : 8+4+i+4])
	if valueLen == 0 {
		// 跳过空数据
		return 0
	}
	// 构建value值
	value := string(b[8+4+i+4 : 8+4+i+4+valueLen])
	for _, v := range values {
		// 字符串是否以前缀开头
		if strings.HasPrefix(value, v) {
			copy(b[8:total-8], make([]byte, total-8))
		}
	}
	// 返回结尾的8位数据
	return bigUint64(b[total-8 : total])
}

// 获取这一链的全部数据，并返回value list和下一链offset
func (s *Msearch) get(offset int) ([]string, int) {
	// 数据长度
	total := bigUint64(s.bytesAddr[offset : offset+8])
	// 全部数据
	b := s.bytesAddr[offset : offset+total]
	keyLen := bigUint64(b[8:12])
	if keyLen == 0 {
		return nil, 0
	}
	valueLen := bigUint64(b[12+keyLen : 12+keyLen+4])
	value := b[12+keyLen+4 : 12+keyLen+4+valueLen]
	return []string{string(value)}, bigUint64(b[total-8 : total])
	// 之前写for是因为，这个链里面，可以有多个value值
	// 现在改成了一个链里面只有一个value值
	// for i := int(b[8] + 1 + 8); i < len(b[:len(b)-8]); {
	// fmt.Printf("%#v , %#v\n", i, int(b[i]))
	// l := i + 1 + int(b[i])
	// if len(b) < l {
	// 	fmt.Printf("%#v , %#v , %#v\n", l, len(b), b)
	// 	panic("ssss")
	// }
	// list = append(list, string(b[i+1:l]))
	// i += int(b[i]) + 1
	// }
}

// bigUint64 对大数字进行解码 长度为0-8位的字节切片. binary.BigEndian.PutUint64 是编码.
// Deprecated: please use binary.BigEndian.Uint64
func bigUint64(buf []byte) int {
	if len(buf) > 8 {
		return 0
	}
	var x int
	for _, b := range buf {
		x = x<<8 | int(b)
	}
	return x
}

```
