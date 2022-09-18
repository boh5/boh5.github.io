---
title: 数据结构和算法学习笔记（五）前缀树和贪心算法
date: 2022-09-15T17:00:00+08:00
lastmod: 2022-09-18T22:43:00+08:00
resources:
- name: featured-image
  src: images/features/zh-cn.png
categories:
  - 学习笔记
  - 数据结构和算法
tags:
  - 数据结构
  - 算法
  - 前缀树
  - 贪心算法
---

## 1. 前缀树

> [维基百科 - Trie](https://zh.wikipedia.org/zh-cn/Trie)
> 
> 在计算机科学中，trie，又称前缀树或字典树，是一种有序树，用于保存关联数组，其中的键通常是字符串。与二叉查找树不同，**键不是直接保存在节点中**，而是由节点在树中的位置决定。一个节点的所有子孙都有**相同的前缀**，也就是这个节点对应的字符串，而根节点对应空字符串。一般情况下，不是所有的节点都有对应的值，只有叶子节点和部分内部节点所对应的键才有相关的值。

**Golang实现**：

```go
package trie

import (
	"errors"
	"math"
)

type Node struct {
	Pass  int     // 添加元素时，node 走过几次
	End   int     // 有几个元素是以当前 node 最为结尾的
	Nexts []*Node // 下级 node 切片, 如果字符太多，可以用 map 代替
}

var (
	ErrNotContain = errors.New("not in trie")
)

func NewNode() *Node {
	return &Node{
		Pass:  0,
		End:   0,
		Nexts: make([]*Node, math.MaxUint8),
	}
}

type Tree struct {
	Root *Node // Root 节点表示空字符串，如过 Root.End == 1，那表示加入了一个空字符串
}

func NewTree() *Tree {
	return &Tree{Root: NewNode()}
}

// Insert 将单词插入前缀树
func (t *Tree) Insert(word string) error {
	chars := []byte(word)

	node := t.Root
	node.Pass++
	for _, c := range chars {
		index := int(c) // 0 位置表示 'a', 1 位置表示 'b'...
		if node.Nexts[index] == nil {
			node.Nexts[index] = NewNode() // 没有就新建
		}
		node = node.Nexts[index]
		node.Pass++ // 路过这个 node, Pass++
	}
	node.End++ // word 放入完毕，最后一个 node 的 End++
	return nil
}

// lastNode 返回单词在树中的最后一个 node
func (t *Tree) lastNode(word string) (*Node, error) {
	chars := []byte(word)

	node := t.Root
	for _, c := range chars {
		index := int(c)
		if node.Nexts[index] == nil {
			return nil, ErrNotContain
		}
		node = node.Nexts[index]
	}
	return node, nil
}

// Search 返回单词在前缀树中插入了几次
func (t *Tree) Search(word string) int {
	lastNode, err := t.lastNode(word)
	if err != nil {
		return 0
	}
	return lastNode.End
}

// PrefixNumber 返回有几个以 pre 作前缀的字符串，注意一个字符串可能加入多次
func (t *Tree) PrefixNumber(pre string) int {
	lastNode, err := t.lastNode(pre)
	if err != nil {
		return 0
	}
	return lastNode.Pass
}

// Delete 从前缀树中删除单词
func (t *Tree) Delete(word string) {
	_, err := t.lastNode(word)
	if err != nil {
		return
	}
	chars := []byte(word)

	node := t.Root
	node.Pass--
	for _, c := range chars {
		index := int(c)
		if node.Nexts[index].Pass--; node.Nexts[index].Pass == 0 {
			node.Nexts[index] = nil
			return
		}
		node = node.Nexts[index]
	}
	node.End--
}
```

**Test**:

```go
package trie_test

import (
	"math/rand"
	"strings"
	"testing"
	"time"

	"github.com/stretchr/testify/require"

	"github.com/boh5/dsal/trie"
)

const alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func init() {
	rand.Seed(int64(time.Now().Nanosecond()))
}
func insertTree(t *testing.T, tree *trie.Tree, words []string, numInTree int) {
	require.NotNil(t, tree)
	for i, w := range words {
		err := tree.Insert(w)
		require.NoError(t, err)
		require.Equal(t, numInTree+i+1, tree.Root.Pass)
	}
	require.Equal(t, 0, tree.Root.End)
}

func TestTree_Insert(t *testing.T) {
	t.Parallel()

	n := 10
	words := randomUniqueWords(n, 10)
	tree := trie.NewTree()

	insertTree(t, tree, words, 0)
	insertTree(t, tree, words, n)
}

func TestTree_Search(t *testing.T) {
	t.Parallel()

	n := 10
	words := randomUniqueWords(n, 10)
	tree := trie.NewTree()

	insertTree(t, tree, words, 0)

	for _, word := range words {
		i := tree.Search(word)
		require.Equal(t, 1, i)
	}

	wordsNotInTree := []string{"NOT", "IN", "Tree"}
	for _, word := range wordsNotInTree {
		i := tree.Search(word)
		require.Equal(t, 0, i)
	}

	repeatWords := words[:5]
	insertTree(t, tree, repeatWords, n)
	for _, word := range repeatWords {
		i := tree.Search(word)
		require.Equal(t, 2, i)
	}
}

func TestTree_PrefixNumber(t *testing.T) {
	t.Parallel()

	words := []string{"abc", "abcd", "abcde", "abcdef"}
	n := len(words)
	tree := trie.NewTree()

	insertTree(t, tree, words, 0)

	for i, word := range words {
		ans := tree.PrefixNumber(word)
		require.Equal(t, n-i, ans)
	}

	notContainPrefixes := []string{"123", "eee", "xyz"}
	for _, word := range notContainPrefixes {
		ans := tree.PrefixNumber(word)
		require.Equal(t, 0, ans)
	}
}

func TestTree_Delete(t *testing.T) {
	t.Parallel()

	n := 10
	words := randomUniqueWords(n, 10)
	tree := trie.NewTree()

	insertTree(t, tree, words, 0)

	// Delete not exists
	tree.Delete("NE")
	for _, word := range words {
		i := tree.Search(word)
		require.Equal(t, 1, i)
	}

	// Delete repeat
	insertTree(t, tree, words[:1], n)
	tree.Delete(words[0])
	require.Equal(t, 1, tree.Search(words[0]))

	// Delete
	for _, word := range words {
		tree.Delete(word)
		i := tree.Search(word)
		require.Equal(t, 0, i)
	}

	// Last status
	require.Equal(t, 0, tree.Root.Pass)
	for _, node := range tree.Root.Nexts {
		require.Nil(t, node)
	}

}

func randomString(n int) string {
	var sb strings.Builder
	for i := 0; i < n; i++ {
		sb.WriteByte(alphabet[rand.Intn(len(alphabet))])
	}
	return sb.String()
}

func randomUniqueWords(n, wordLen int) []string {
	words := make([]string, n)
	seen := make(map[string]any)
	for i := 0; i < n; i++ {
		var str string
		for {
			str = randomString(wordLen)
			if _, ok := seen[str]; !ok {
				break
			}
		}
		seen[str] = nil
		words[i] = str
	}
	return words
}
```

## 2. 贪心算法

> 局部最优 -> 整体最优

**解题套路**

1. 实现一个不依靠贪心策略的解法X，可以用最暴力的尝试
2. 脑补出贪心策略A、贪心策略B、贪心策略C...
3. 用解法X和对数器，去验证每一个贪心策略，用实验的方式得知哪个贪心策略正确
4. 不要去纠结贪心策略的证明

**算法题**:

- [ ][1547. 切棍子的最小成本](https://leetcode.cn/problems/minimum-cost-to-cut-a-stick/)
