---
title: golang map takes too much memory
layout: post
category : golang
tags : [golang]
---

## golang 1.4.2 map takes too much memory

开发中发现内存占用超出预计，反复review代码之后发现只可能是数据结构的问题。于是做了一个简单的数据结构内存占用测试，于是乎就发现了这个平时没有注意的问题。

场景是有一个struct的属性是 `map[string]string` 这样的键值对。如果这个struct被创建很多内存占用会异常大。这个主要是和`[]string`这样的数据结构比会异常大。 基本上是`[]string`这样数据结构的**4倍以上**的内存占用。 *而java做类似对比差异就小很多*。

不知道在golang新的版本中是否有类似的问题。所以还是**尽量避免创建大量的`map[string]string`**才好。

```golang
package bench

import (
	"fmt"
	"runtime"
	"testing"
	"time"
)

type empty_struct struct{}

var mem_stats runtime.MemStats

func Test_datatype_memory_empty_struct(t *testing.T) {
	time.Sleep(time.Second)

	runtime.ReadMemStats(&mem_stats)
	prev_heap := mem_stats.HeapSys

	objects := []empty_struct{}
	for i := 0; i < 100000; i++ {
		objects = append(objects, empty_struct{})
	}

	runtime.ReadMemStats(&mem_stats)
	t.Logf("prev_heap: %v, heap now: %v, delta: %v/kb", prev_heap, mem_stats.HeapSys, (mem_stats.HeapSys-prev_heap)/1024)
}

func Test_datatype_memory_string(t *testing.T) {
	time.Sleep(time.Second)

	runtime.ReadMemStats(&mem_stats)
	prev_heap := mem_stats.HeapSys

	objects := []string{}
	for i := 0; i < 100000; i++ {
		objects = append(objects, fmt.Sprintf("string%d", i))
	}

	runtime.ReadMemStats(&mem_stats)
	t.Logf("prev_heap: %v, heap now: %v, delta: %v/kb", prev_heap, mem_stats.HeapSys, (mem_stats.HeapSys-prev_heap)/1024)
}

// map memory usage is 4x bigger than []string !!!
func Test_datatype_memory_map_string(t *testing.T) {
	time.Sleep(time.Second)

	runtime.ReadMemStats(&mem_stats)
	prev_heap := mem_stats.HeapSys

	objects := []map[string]string{}
	for i := 0; i < 100000; i++ {
		objects = append(objects, map[string]string{"key": fmt.Sprintf("string%d", i)})
	}
	runtime.ReadMemStats(&mem_stats)
	t.Logf("prev_heap: %v, heap now: %v, delta: %v/kb", prev_heap, mem_stats.HeapSys, (mem_stats.HeapSys-prev_heap)/1024)
}
```


```bash
# golang 1.4.2 windows 64
=== RUN Test_datatype_memory_empty_struct
--- PASS: Test_datatype_memory_empty_struct (1.00s)
        datatype_memory_test.go:26: prev_heap: 983040, heap now: 983040, delta: 0/kb
=== RUN Test_datatype_memory_string_array
--- PASS: Test_datatype_memory_string_array (1.04s)
        datatype_memory_test.go:41: prev_heap: 950272, heap now: 8323072, delta: 7200/kb
=== RUN Test_datatype_memory_map_string
--- PASS: Test_datatype_memory_map_string (1.09s)
        datatype_memory_test.go:56: prev_heap: 8290304, heap now: 38731776, delta: 29728/kb

# golang 1.4.2 ubuntu 12.04 amd64
=== RUN Test_datatype_memory_empty_struct
--- PASS: Test_datatype_memory_empty_struct (1.00s)
        datatype_memory_test.go:37: prev_heap: 851968, heap now: 851968, delta: 0/kb
=== RUN Test_datatype_memory_string_array
--- PASS: Test_datatype_memory_string_array (1.04s)
        datatype_memory_test.go:52: prev_heap: 884736, heap now: 8716288, delta: 7648/kb
=== RUN Test_datatype_memory_map_string
--- PASS: Test_datatype_memory_map_string (1.09s)
        datatype_memory_test.go:67: prev_heap: 8749056, heap now: 39124992, delta: 29664/kb
```









