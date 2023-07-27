---
categories: Go
date: "2021-06-23T00:00:00Z"
title: Go入门——以LeetCode上的'#909蛇梯棋'为例
---

## 前言

由于最近Go语言比较火，本着技多不压身的想法，也顺便看了下Go语言的，跟着官方的[Go语言之旅](https://tour.go-zh.org/list)走了一遍之后，就准备自己写个小Demo试试，就去LeetCode上找一题做做看，刚好上去那天LeetCode推荐的每日一题就是这道[#909蛇梯棋](https://leetcode-cn.com/problems/snakes-and-ladders/),就用这道题目来做练习了。

## 项目结构

虽说可以用LeetCode提供的编辑器直接写代码，但是此次目的更重要在于学习Go编程。对于一个项目来说，肯定不能直接用一个文件来做开发。
Go有很多包管理器，`go mod`似乎已经成为主流了，所以使用`go mod`来作为项目的包管理工具，具体使用可以参考这篇:[go mod使用](https://www.jianshu.com/p/760c97ff644c)。个人感觉这篇写得不错

选择好包管理工具之后就是觉得项目结构了。考虑到这只是作为玩具项目，并不打算搞太复杂的结构。
简单的使用两个包，一个`leetCode`来存放解题的代码，另一个`main`包含`main.go`作为入口调用。
对于大型的项目可以参考开源的Go项目，最出名的就是[Kubernetes](https://github.com/kubernetes/kubernetes)和[Docker](https://github.com/moby/moby)。

## 解题
### 题目：
>
>N x N 的棋盘 board 上，按从 1 到 N*N 的数字给方格编号，编号 从左下角开始，每一行交替方>向。
>
>例如，一块 6 x 6 大小的棋盘，编号如下：
>
>
>r 行 c 列的棋盘，按前述方法编号，棋盘格中可能存在 “蛇” 或 “梯子”；如果 board[r][c] != >-1，那个蛇或梯子的目的地将会是 board[r][c]。
>
>玩家从棋盘上的方格 1 （总是在最后一行、第一列）开始出发。
>
>每一回合，玩家需要从当前方格 x 开始出发，按下述要求前进：
>
>选定目标方格：从编号为 x+1，x+2，x+3，x+4，x+5，或者 x+6 的方格中选出一个作为目标方格 >s ，目标方格的编号 <= N*N。
>该选择模拟了掷骰子的情景，无论棋盘大小如何，你的目的地范围也只能处于区间 [x+1, x+6] 之>间。
>传送玩家：如果目标方格 S 处存在蛇或梯子，那么玩家会传送到蛇或梯子的目的地。否则，玩家传送>到目标方格 S 。 
>注意，玩家在每回合的前进过程中最多只能爬过蛇或梯子一次：就算目的地是另一条蛇或梯子的起点，>你也不会继续移动。
>
>返回达到方格 N*N 所需的最少移动次数，如果不可能，则返回 -1。
>
> 
>
>示例：
>
>输入：[
>[-1,-1,-1,-1,-1,-1],
>[-1,-1,-1,-1,-1,-1],
>[-1,-1,-1,-1,-1,-1],
>[-1,35,-1,-1,13,-1],
>[-1,-1,-1,-1,-1,-1],
>[-1,15,-1,-1,-1,-1]]
>输出：4
>解释：
>首先，从方格 1 [第 5 行，第 0 列] 开始。
>你决定移动到方格 2，并必须爬过梯子移动到到方格 15。
>然后你决定移动到方格 17 [第 3 行，第 4 列]，必须爬过蛇到方格 13。
>然后你决定移动到方格 14，且必须通过梯子移动到方格 35。
>然后你决定移动到方格 36, 游戏结束。
>可以证明你需要至少 4 次移动才能到达第 N*N 个方格，所以答案是 4。
> 
>
>提示：
>
>2 <= board.length = board[0].length <= 20
>board[i][j] 介于 1 和 N*N 之间或者等于 -1。
>编号为 1 的方格上没有蛇或梯子。
>编号为 N*N 的方格上没有蛇或梯子。
>
>来源：力扣（LeetCode）
>链接：https://leetcode-cn.com/problems/snakes-and-ladders
>著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
>

可以看到题目并不难，用广度优先搜索即可，其他的问题在于要计算出各自编号和在数组的位置的对应关系。

### 编号与位置的对应关系
根据编号计算下标
```Go
func computePos(n int, code int) (r, c int) {
	r = n - (code+n-1)/n
	if (n-r)%2 == 0 {
		c = n - (code - (n-r-1)*n)
	} else {
		c = (code - (n-r-1)*n) - 1
	}

	return r, c
}
```
在这里，Go函数可以有多个返回值的优势就体现出来了。如果只能返回一个值，要不把值放在一个对象里面，如果是`C++`的话可以存在`pair`里面，但是总归是多了一步的。
也可以考虑在参数中传递引用返回，如:
```C++
void computePos(int n, int code, int& r, int& c)
```
但是这样就不是很直观了，或者说和直觉不太符合。

### 队列
广度优先搜索第一反应就是要用到队列，但是Go是没有提供队列这种数据结构的，好在可以使用切片很方便的模拟出队列的效果：
```Go
//初始化队列，队列中每个元素包含两个值，第一个表示当前的编号，第二个值表示走了几步
pos := [][]int{
	{1, 0},
}
//获取队首元素，即获取切片中第一个元素
currPos := pos[0]
//出队，即删除切片中第一个元素
pos = pos[1:]
//入队
pos = append(pos, []int{nextCode, currPos[1] + 1})
```
要实现队列的效果虽然不麻烦，但总归感觉有点遗憾。现在Go还不支持泛型，看到消息有说Go 1.17会添加泛型编程，不知道能不能达到C++中模板那样的程度。等出了泛型再考虑自己动手实现一个队列吧

### 完整代码
```Go
package leetCode

func SnakeAndLadders(board [][]int) int {
	n := len(board)
	target := n * n
	pos := [][]int{
		{1, 0},
	}
	visited := make(map[int]bool, target+1)
	visited[1] = true
	for len(pos) > 0 {
		//判断是否到达终点
		if pos[0][0] == target {
			return pos[0][1]
		}
		currPos := pos[0]
		//删除第一个元素
		pos = pos[1:]
		for i := 1; i <= 6; i++ {
			nextCode := currPos[0] + i
			if nextCode > target {
				break
			}
			//判断是不是梯子
			r, c := computePos(n, currPos[0]+i)
			if board[r][c] != -1 {
				nextCode = board[r][c]
			}
			if nextCode == target {
				return currPos[1] + 1
			}
			if _, ok := visited[nextCode]; ok == false {
				pos = append(pos, []int{nextCode, currPos[1] + 1})
				//标记已经访问过
				visited[nextCode] = true
			}
			//}
		}

	}

	return -1
}

//根据编号计算位置
func computePos(n int, code int) (r, c int) {
	r = n - (code+n-1)/n
	if (n-r)%2 == 0 {
		c = n - (code - (n-r-1)*n)
	} else {
		c = (code - (n-r-1)*n) - 1
	}

	return r, c
}
```
### 调用
实现了之后，就需要外部能调用，调试时也比较好打断点。最简单当然时直接再同一个包下加个main函数了，但是为了练习下如何引用自己写包的，在这里添加另一个`main`包，在里面的main函数中进行调用。

使用`go mod`作为包管理器，只需要修改`go.mod`文件，添加依赖即可。由于`leetCode`包并没有发布，所以需要指定本地路径：
```
module muzhy/main

go 1.16

require muzhy/leetCode v0.0.0

replace muzhy/leetCode => ../leetCode

```
其中`../leetCode`为本地的相对路径

```Go
// main.go

package main

import (
	"fmt"
	"leetCode"
)

func main() {

	board := [][]int{
		{-1, -1, -1, -1, -1, -1},
		{-1, -1, -1, -1, -1, -1},
		{-1, -1, -1, -1, -1, -1},
		{-1, 35, -1, -1, 13, -1},
		{-1, -1, -1, -1, -1, -1},
		{-1, 15, -1, -1, -1, -1},
	}
	fmt.Println(leetCode.SnakeAndLadders(board))
}
```
这样就可以调用了

## 测试
Go自带了测试框架，直接使用该测试框架就可以进行测试了，还是比较方便的
在这里只进行了单元测试。
```Go 
package leetCode

import "testing"

func TestSnakesAdnLadder(t *testing.T) {

	cases := []struct {
		board [][]int
		want  int
	}{
		{
			[][]int{
				{-1, -1, -1, -1, -1, -1},
				{-1, -1, -1, -1, -1, -1},
				{-1, -1, -1, -1, -1, -1},
				{-1, 35, -1, -1, 13, -1},
				{-1, -1, -1, -1, -1, -1},
				{-1, 15, -1, -1, -1, -1},
			}, 4,
		},
		{
			[][]int{
				{-1, -1, -1},
				{-1, 9, 8},
				{-1, 8, 9},
			}, 1,
		}, {
			[][]int{
				{1, 1, -1},
				{1, 1, 1},
				{-1, 1, 1},
			}, -1,
		},
		{
			[][]int{
				{-1, -1, -1, 46, 47, -1, -1, -1},
				{51, -1, -1, 63, -1, 31, 21, -1},
				{-1, -1, 26, -1, -1, 38, -1, -1},
				{-1, -1, 11, -1, 14, 23, 56, 57},
				{11, -1, -1, -1, 49, 36, -1, 48},
				{-1, -1, -1, 33, 56, -1, 57, 21},
				{-1, -1, -1, -1, -1, -1, 2, -1},
				{-1, -1, -1, 8, 3, -1, 6, 56},
			}, 4,
		},
	}

	for _, c := range cases {
		step := SnakeAndLadders(c.board)
		if step != c.want {
			t.Errorf("board is %v, result is %v expect %v \n", c.board, step, c.want)
		}
	}

}
```
可以把提交出错的用例添加到单元测试中，在提交之前只要跑一下测试`go test`，就可以保证修改不会导致之前的用例失败了。

## 总结
这样一个比较简单的项目算完成了，包含了不同包的调用，添加了单元测试。当然了，大型的项目的包更多，测试也更加复杂。但是，最基本的项目结构也是有了
可以看到还是比较简单的，`go mod`与`CMake`感觉要点像，但是要简单得多，也可能是还没用来组织大型项目的关系。这里也只用很小的一部分Go的特性，而Go最重要的并发并不能体现出来，后续在考虑找个能体现并发性的Demo吧、