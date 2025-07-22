+++
isCJKLanguage = true
title = "一个轻量级的 Go 数据流处理库 - Lapluma"
description = "一个轻量级的 Go 数据流处理库 - Lapluma"
keywords = ["Go"]
date = 2025-07-22T15:38:27+08:00
authors = ["木章永"]
tags = ["Go"]
categories = ["Go"]
cover = "/images/Go.png"
+++

最近在学习`Go`, 打算写点小项目来练手，实现的过程中发现需要在`slice`上执行`Filter`操作，但是标准库没有提供，像`go-stream`这些库提供的又是比较高级的抽象，所以就有了`Lapluma`这个库

仓库地址：[lapluma](https://github.com/muzhy/lapluma)

# 核心设计理念
`Lapluma`旨在提供一套简洁、可组合且易于理解的数据处理工具，通过提供一组正交的基础操作，开发者将这些模块进行组合，构建出满足需求的数据处理流水线

`Lapluma`提供了两个核心组件：`Iterator`和`Pipe`

## 1. `Iterator` - 串行数据流
`Iterator` 是一个前向迭代器接口，它定义了对数据序列的逐一访问。

**主要操作:**
- `FromSlice(data []E) Iterator[E]`: 从切片创建迭代器。
- `FromMap(data map[K]V) Iterator[Pair[K,V]]`: 从 map 创建迭代器。
- `Map[E, R](it Iterator[E], handler func(E) R) Iterator[R]`: 对每个元素应用一个无错误的转换。
- `Filter[E](it Iterator[E], filter func(E) bool) Iterator[E]`: 过滤不符合条件的元素。
- `Reduce[E, R](it Iterator[E], handler func(R, E) R, initial R) R`: 将序列聚合为单个值。
- `Collect[E](it Iterator[E]) []E`: 将迭代器中的所有元素收集到切片中。

**示例:**
```go
// 创建迭代器
data := []int{1, 2, 3, 4, 5}
it := iterator.FromSlice(data)

// 链式操作
result := iterator.Collect(
    iterator.Filter(
        iterator.Map(it, func(x int) int { return x * 2 }),
        func(x int) bool { return x > 5 }
    )
) // [6, 8, 10]
```

## 2. `Pipe` - 并发数据流
`Pipe` 基于 Go 的 channel 构建，每个操作（如 `Map`, `Filter`）都在一个独立的 goroutine 中运行，形成一条处理流水线。

所有的 `Pipe` 操作都与 `context.Context` 集成，可以轻松实现超时控制和优雅退出。


**主要操作:**
- `FromSlice(data []E, ctx context.Context) *Pipe[E]`: 从切片创建并发管道。
- `FromIterator(it iterator.Iterator[E], ctx context.Context) *Pipe[E]`: 从迭代器创建并发管道。
- `Map`, `Filter`, `Reduce` 等函数与 `Iterator` 版本功能相同，但以并发方式执行。

Pipe 提供的 Map、Filter、Reduce 等函数与 Iterator 版本功能类似，但它们在内部会启动 Goroutine 进行并发处理。可以为 Map 和 Filter 操作指定并行度和缓冲区大小，从而精细控制并发资源的利用。
PS: 现在还每想好具体的并行控制参数，后续打算将并行控制参数用一个`struct`表示，现在的方案为临时方案

**示例:**
```go
ctx := context.Background()

// 创建并发管道
p := pipe.FromSlice([]int{1, 2, 3, 4, 5}, ctx)

// 并行处理（3个工作协程）
result := pipe.Collect(
    pipe.Filter(
        pipe.Map(p, cpuIntensiveTask, 3), // 并行度3
        func(x int) bool { return x > 10 },
        2, // 并行度2
    )
)
```

## 标准迭代器集成
`Pipe`也实现了`Iterator`的接口，所以也算是一种迭代器，兼容 Go 1.23+ 的标准 iter 包, 可以直接通过 for-range 语法遍历

```go
import "iter"

// 兼容 Go 1.23+ 的 for-range 语法
itForRange := iterator.Filter(
	iterator.Map(iterator.FromSlice([]string{"1", "2", "3", "4"}), func(s string) int {
		val, _ := strconv.Atoi(s)
		return val * 3
	}),
	func(x int) bool { return x < 10 },
)
fmt.Print("for-range 遍历结果: ")
for data := range iterator.Iter(itForRange) {
	fmt.Printf("%d ", data) // 输出: 3 6 9
}
fmt.Println()
```

## 错误处理
`LaPluma` 在设计上有意简化了核心转换函数的签名，例如 `Map` 的 `handler` 是 `func(T) R` 而不是 `func(T) (R, error)`。这并非忽略错误，而是一种设计选择：**将错误视为数据流的一部分来处理**。

推荐以下两种模式来处理可能失败的操作：

### 模式一：前置过滤 (Pre-filtering)

如果某些数据从一开始就是非法的，或者不符合处理条件，应该在进入核心处理逻辑前，使用 `Filter` 将其剔除。

```go
// 示例：只处理正数
pipe := FromSlice([]int{1, -2, 3, -4}, ctx)
positivePipe := Filter(pipe, func(n int) bool {
    return n > 0
})
// ... 后续操作只会看到 {1, 3}
```

### 模式二：使用 TryMap 处理可失败的转换

当数据转换过程本身可能失败时（例如，解析字符串、调用外部 API），使用 TryMap 函数。它的 handler 签名为 func(T) (R, error)。当 handler 返回一个非 nil 的 error 时，TryMap 会自动跳过（丢弃） 这个元素，并继续处理下一个。这使得流水线可以在遭遇“数据级”错误时保持运行，而不会被中断。
```go
import (
    "strconv"
    "errors"
)

// 示例：将字符串转换为整数，失败则跳过
stringPipe := FromSlice([]string{"1", "two", "3", "four"}, ctx)

// 使用 TryMap，handler 返回 (int, error)
intPipe := TryMap(stringPipe, func(s string) (int, error) {
    i, err := strconv.Atoi(s)
    if err != nil {
        // 返回错误，这个元素将被丢弃
        return 0, errors.New("not a number")
    }
    return i, nil
})

// 最终 Reduce 只会处理成功转换的 {1, 3}
sum := Reduce(intPipe, func(acc, n int) int { return acc + n }, 0)
// sum 的结果是 4
```

PS：若需要收集`Map`过程中的错误,可以考虑使用在`util.go`中`Result[T]`作为返回值，要如何设计此场景的错误处理机制还没想好：通过在调用时添加一个`onError`参数来处理错误；或者返回两个`Pipe`，用其中一个来处理错误信息；或者其他方案

# 运行测试
```sh
go test ./...
```

# 后续计划
- 提供更丰富的转换操作, 如`Distinct`, `Zip`, `Peek`
- 完善错误处理机制
- 规范`Pipe`的并发控制参数