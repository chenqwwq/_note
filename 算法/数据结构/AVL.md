

```go
package main

import (
	"fmt"
	"strconv"
)

/**
 * @author chen
 * @date: 2021/10/2 11:17 下午
 * @description:
 * AVL 树就是可以自平衡的二叉查找树，查找和平衡的时间复杂度都为O(logN)
 * AVL 需要平衡的情况分为 LL，RR，LR，RL 四种，
 * LL 情况如下：
 *  [5]        [5]					[4]
 *   /   ===>  /    == rotate =>    / \
 * [4]       [4]				  [3] [5]
 *          /
 *        [3]
 * 文字表述上就是失衡节点(5)下挂到左子节点(4)的右节点上，并且将其右节点改为自己的右节点，RR 可以类推。
 * <p>
 * 如果是 LR 就是先 LL 转化为 RR 的形态，在进行 RR 的操作，RL 可以类推
 * <p>
 * 旋转注意节点高度的变化
 */
// node AVL 树节点
type node struct {
	left, right *node
	val         int
	height      int
}

// AVLTree 接口
type AVLTree interface {
	Add(val int) *node
	Del(avl int) *node
	IsBalance() bool
	Max() *node
	Min() *node
	Find(target int) *node
	Contains(target int) *node
}

// AVL 树
type AVL struct {
	root          *node
	balanceFactor int // 平衡因子，能接受的左右子树的高度差
}

func getTestTree() *AVL {
	n := Constructor()
	n.Add(5)
	n.Add(3)
	n.Add(7)
	n.Add(2)
	n.Add(4)
	n.Add(6)
	n.Add(8)
	n.Add(1)
	n.Add(9)
	return n
}

func (avl *AVL) Add(val int) *node {
	// create new node
	newNode := newNode(val)
	if avl.root == nil {
		avl.root = newNode
		return newNode
	}
	// add to root
	avl.root = avl.root.add(newNode)
	return newNode
}

func (avl *AVL) Del(val int) *node {
	avl.root = avl.root.del(val)
	return avl.root
}

func (avl *AVL) Contains(target int) *node {
	return avl.root.contains(target)
}

func (avl *AVL) Find(target int) *node {
	return avl.root.find(target)
}

func (avl *AVL) Max() *node {
	return avl.root.maxNode()
}

func (avl *AVL) Min() *node {
	return avl.root.minNode()
}

func Constructor() *AVL {
	return &AVL{nil, 1}
}

func (avl *AVL) String() string {
	return avl.root.format()
}

func (avl *AVL) IsBalance() bool {
	return avl.root.isBalance()
}

// 内部方法

func (this *node) contains(target int) *node {
	if this == nil {
		return nil
	}

	if this.val == target {
		return this
	} else if this.val < target {
		return this.right.contains(target)
	} else {
		return this.left.contains(target)
	}

}

// del 删除节点  返回删除后的子树根节点
func (this *node) del(target int) *node {
	if this == nil {
		return nil
	}
	// 需要删除的不是当前节点,但是删除之后高度可能发生变化
	ret := this
	if target < this.val {
		this.left = this.left.del(target)
	} else if target > this.val {
		this.right = this.right.del(target)
	} else if this.val == target {
		// 需要删除的就是当前节点
		if this.left != nil && this.right != nil {
			// 如果左右子树都在,就找左子树的最大值或者右子树的最小值
			// 这里选择左子树的最大值
			// right 存在，所以 maxNode 肯定存在
			maxNode := this.left.maxNode()
			maxNode.left = this.left.del(maxNode.val)
			maxNode.right = this.right
			ret = maxNode
		} else if this.left != nil {
			ret = this.left
		} else if this.right != nil {
			ret = this.right
		} else {
			ret = nil
		}
		// 还有如果当前节点为子节点的时候
	}
	if ret == nil {
		return ret
	}
	ret.height = max(ret.left.getHeight(), ret.right.getHeight()) + 1
	// 判断是否需要重新平衡
	return ret.balance()
}

// balance 平衡
func (this *node) balance() *node {
	if this.isBalance() {
		return this
	}
	if this.getBalanceFactor() > 0 {
		if this.left.getBalanceFactor() > 0 {
			return this.rotateRR()
		} else {
			return this.rotateLR()
		}
	} else {
		if this.right.getBalanceFactor() < 0 {
			return this.rotateLL()
		} else {
			return this.rotateRL()
		}
	}
}

// getBalanceFactor 获取平衡因子
// 左边节点高度减去右边节点高度的数值
func (this *node) getBalanceFactor() int {
	if this == nil {
		return 0
	}
	return this.left.getHeight() - this.right.getHeight()
}

func (this *node) find(target int) *node {
	if this == nil {
		return nil
	}

	if this.val == target {
		return this
	}
	if this.val > target {
		return this.left.find(target)
	}
	return this.right.find(target)
}

func (this *node) maxNode() *node {
	if this == nil || this.right == nil {
		return this
	}
	return this.right.maxNode()
}

func (this *node) minNode() *node {
	if this == nil || this.left == nil {
		return this
	}
	return this.left.minNode()
}

func (this *node) add(newNode *node) *node {
	if this == nil {
		return newNode
	}
	if this.val == newNode.val {
		return this
	}
	if newNode.val < this.val {
		this.left = this.left.add(newNode)
	} else {
		this.right = this.right.add(newNode)
	}

	this.height = max(this.right.getHeight(), this.left.getHeight()) + 1
	return this.balance()
}

func (this *node) format() string {
	if this == nil {
		return "#"
	}
	return strconv.Itoa(this.val) + this.left.format() + this.right.format()
}

func newNode(val int) *node {
	return &node{
		val:    val,
		height: 1,
	}
}

func (this *node) isBalance() bool {
	if this == nil {
		return true
	}
	return abs(this.left.getHeight()-this.right.getHeight()) <= 1
}

func (this *node) getHeight() int {
	if this == nil {
		return 0
	}
	return this.height
}

func abs(a int) int {
	if a < 0 {
		return -a
	}
	return a
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

// 旋转方法
// LL 和 RR 可能颠倒

// rotateRR 表示的是左左的失去平衡
// 理解中的需要左旋的情况
//       5(root)
//      / \
//     4   t1              4
//    / \	  --右旋->	 /   \
//   3   t2		        3     5
//  / \                / \   / \
// t3  t4             t3 t4 t2  t1
//
func (this *node) rotateRR() *node {
	// modify ptr
	left := this.left
	this.left = left.right
	left.right = this

	// update node's height
	// important: 个人在这里有一个误区
	// 高度表示的是当前节点下的子节点的层数，同个节点的左右节点的高度可能是不同的
	this.height = max(this.left.getHeight(), this.right.getHeight()) + 1
	left.height = max(left.left.getHeight(), left.right.getHeight()) + 1
	return left
}

// rotateLL
func (this *node) rotateLL() *node {
	right := this.right
	this.right = right.left
	right.left = this
	this.height = max(this.left.getHeight(), this.right.getHeight()) + 1
	right.height = max(right.left.getHeight(), right.right.getHeight()) + 1
	return right
}

// rotateLR
func (this *node) rotateLR() *node {
	this.left = this.left.rotateLL()
	return this.rotateRR()
}

// rotateRL
func (this *node) rotateRL() *node {
	this.right = this.right.rotateRR()
	return this.rotateLL()
}

```

