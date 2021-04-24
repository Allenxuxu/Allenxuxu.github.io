---
title: "二叉树的遍历模版（递归，迭代）"
date: 2020-04-26T11:22:38+08:00
draft: false
tags: ["Go", "algorithm"]
---

![image-20210424121612528](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/image-20210424121612528.png)

> 图片来自 leetcode

- 深度优先遍历（dfs）
  - 前序遍历
  - 中序遍历
  - 后序遍历
- 广度优先遍历（bfs）

```go
type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}
```

## 深度优先遍历

### 递归

递归版本，代码比较简单，只需改变 append 数据的位置即可。

#### 前序遍历

```go
func preorderTraversal(root *TreeNode) []int {
	var ret []int
	helper(root, &ret)
	return ret
}

func helper(root *TreeNode, data *[]int) {
	if root == nil {
		return
	}
	*data = append(*data, root.Val)
	helper(root.Left, data)
	helper(root.Right, data)
}
```

#### 中序遍历

```go
func inorderTraversal(root *TreeNode) []int {
	var ret []int
	helper(root, &ret)
	return ret
}

func helper(root *TreeNode, data *[]int) {
	if root == nil {
		return
	}
	helper(root.Left, data)
	*data = append(*data, root.Val)
	helper(root.Right, data)
}
```

#### 后序遍历

```go
func postorderTraversal(root *TreeNode) []int {
	var ret []int
	helperPostOrder(root, &ret)
	return ret
}

func helperPostOrder(root *TreeNode, data *[]int) {
	if root == nil {
		return
	}
	helperPostOrder(root.Left, data)
	helperPostOrder(root.Right, data)
	*data = append(*data, root.Val)
}
```

### 迭代

迭代版本，稍微复杂，需要模拟函数调用栈，需要使用 stack，只需要改变压栈的位置即可，代码模版性较好，便于记忆。

在文末附录有基于 golang 标准库的 list 实现的 stack 。

#### 前序遍历

```go
func preorderTraversal(root *TreeNode) []int {
	if root == nil {
		return nil
	}

	var ret []int
	s := stack.New()
	s.Push(root)

	for s.Len() > 0 {
		c := s.Pop()

		if c != nil {
			node := c.(*TreeNode)
			if node.Right != nil { //右节点先压栈，最后处理
				s.Push(node.Right)
			}
			if node.Left != nil {
				s.Push(node.Left)
			}

			s.Push(node) //当前节点重新压栈（留着以后处理），因为先序遍历所以最后压栈
			s.Push(nil)  //在当前节点之前加入一个空节点表示已经访问过了
		} else { // 当前 c == nil , 说明这个节点已经访问过了
			node := s.Pop().(*TreeNode) // node 是上面 s.Push(node) 中的那个 node
			ret = append(ret, node.Val)
		}
	}

	return ret
}
```

#### 中序遍历

```go
func inorderTraversal(root *TreeNode) []int {
	if root == nil {
		return nil
	}

	var ret []int
	s := stack.New()
	s.Push(root)

	for s.Len() > 0 {
		c := s.Pop()

		if c != nil {
			node := c.(*TreeNode)

			if node.Right != nil {
				s.Push(node.Right) //右节点先压栈，最后处理
			}
			s.Push(node)
			s.Push(nil)
			if node.Left != nil {
				s.Push(node.Left)
			}
		} else {
			node := s.Pop().(*TreeNode)
			ret = append(ret, node.Val)
		}
	}

	return ret
}
```

#### 后序遍历

```go
func postorderTraversal(root *TreeNode) []int {
	if root == nil {
		return nil
	}

	var ret []int
	s := stack.New()
	s.Push(root)

	for s.Len() > 0 {
		c := s.Pop()

		if c != nil {
			node := c.(*TreeNode)

			s.Push(node)
			s.Push(nil)

			if node.Right != nil {
				s.Push(node.Right)
			}
			if node.Left != nil {
				s.Push(node.Left)
			}
		} else {
			node := s.Pop().(*TreeNode)
			ret = append(ret, node.Val)
		}
	}

	return ret
}
```

## 广度优先遍历

广度优先遍历需要使用 queue，文末附录有 queue 的简单实现。

```go
func levelOrder(root *TreeNode) [][]int {
	if root == nil {
		return nil
	}

	var ret [][]int
	q := queue.New(10)
	q.Push(root)

	for q.Len() > 0 {
		size := q.Len()
		tmp := make([]int, 0, size)
		for i := 0; i < size; i++ {
			node := q.Pop()
			tmp = append(tmp, node.Val)

			if node.Left != nil {
				q.Push(node.Left)
			}
			if node.Right != nil {
				q.Push(node.Right)
			}
		}
		ret = append(ret, tmp)
	}

	return ret
}
```

## 附录

### stack 实现

```go
package stack

import "container/list"

type Stack interface {
	Push(v interface{})
	Pop() interface{}
	Len() int
}

type stack struct {
	list *list.List
}

func New() Stack {
	return &stack{
		list: list.New(),
	}
}
func (s *stack) Push(v interface{}) {
	s.list.PushBack(v)
}

func (s *stack) Pop() interface{} {
	v := s.list.Back()
	if v == nil {
		return nil
	}

	return s.list.Remove(v)
}

func (s *stack) Len() int {
	return s.list.Len()
}
```

### queue实现

```go
type queue struct {
	array []*TreeNode
}

func New(size int) *queue {
	return &queue{
		array: make([]*TreeNode, 0, size),
	}
}

func (q *queue) Push(i *TreeNode) {
	q.array = append(q.array, i)
}
func (q *queue) Pop() *TreeNode {
	if q.Len() == 0 {
		return nil
	}

	e := q.array[0]
	q.array = q.array[1:]

	return e
}
func (q *queue) Len() int {
	return len(q.array)
}
```

