# 红黑树那点事儿


> 前置知识：二叉查找树、B树，本文不多做介绍。

## 0. 写在前面

在观察和思考红黑树时，一定要牢记：红黑树是一颗平衡二叉查找树，我们所赋予红黑树的一切行为，都是为了使其能够保持平衡性。

红黑树是由`2-3-4树`（4阶B树）等价过来的（当然也有由`2-3树`等价来的左倾或右倾红黑树），而这层等价关系，是我们用以维持红黑树平衡性的根本。

![等价关系.jpeg](https://i.loli.net/2021/05/06/h6DA5UabVoznLRC.jpg)

在红黑树中，我们以黑色来标识对应B树里的`2-结点`，而红色结点表示其`parent`到自己的链接是一条红链接（向上融合后对应B树中的`3-结点`，若兄弟结点也是红色则一起向上融合对应`4-结点`）

接下来我们来看看红黑树的五大性质：

- 结点为红色或黑色
- 根为黑色
- NULL叶子结点均为黑色
- 每个叶子结点到根的路径上不能有两个连续的红色结点
- 完美黑色平衡

通过前面展示的等价关系，要推导出这几条性质不成问题。

## 1. 旋转以及颜色翻转的本质

首先我们要明确一个概念：旋转是作用于红链接的操作。那么，旋转操作到底给我们的红黑树带来了怎样的影响呢？

### Ⅰ. 旋转

旋转分为左旋（`left-rotate`）和右旋（`right-rotate`）：

- 左旋：将右红链接转换为左红链接
- 右旋：将左红链接转换为右红链接

![rotate.jpg](https://i.loli.net/2021/05/06/PGNnhry5dsOc2D8.jpg)

可以看到：在经过旋转操作后，我们可以调节某一侧黑色结点的平衡，而对应的`2-3-4树`结点并未发生任何变化。

### Ⅱ. 颜色翻转（flip-color）

颜色翻转其实对应的是`2-3-4树`中`4-结点`需要分裂时的情况：一个新的`KEY`想要加入`4-结点`，会导致原有的`4-结点`分裂，原来中间的`KEY`会提到上层进行融合。

![flipColors.jpg](https://i.loli.net/2021/05/06/AbkSlQLrFh2OIV1.jpg)

当我们插入`d`时，会形成一个临时`5-结点`，这时原先的`4-结点`将分裂，将`b`提上去进行融合（标为红色），当然，若此时`b`已经成为根节点，则进一步标为黑色。可以看到，我们通过简单的颜色翻转即可达到`2-3-4树`中结点分裂的效果。

## 2. 平衡性维持

在我们按照二叉查找树的规则做完插入或删除操作之后，就要通过观察对应`2-3-4树`中的变化，对我们的红黑树进行等价的操作从而恢复对应关系（即维持红黑树的平衡性）。

### Ⅰ. 插入后的平衡性维持

新的结点作为红色叶子结点插入到树中，无非为以下4种情况：

#### a. 新增一个2-结点

![2node.jpeg](https://i.loli.net/2021/05/06/Zdvp5jHaSq8JYEc.jpg)

如果新增的`2-结点`将作为根节点，则将其进一步标为黑色。

#### b. 与一个2-结点合并成为3-结点

![3node.jpeg](https://i.loli.net/2021/05/06/JFkmLxbgrQTlit8.jpg)

即新结点的`parent`结点为黑色，无需调整。

#### c. 与一个3-结点合并成为4-结点

![4node.jpeg](https://i.loli.net/2021/05/06/bJQ1nCwpxGOEVHI.jpg)

- 图中黑色虚线方框内是插入新节点时其`parent`结点以及`grandparent`结点可能的情形
- 橙色三角标所指表示我们新插入的结点的位置

可能会出现连续两条红链接的情况，这时需要通过旋转操作进行调整（为上图下半部分的形式）。

#### d. 与一个4-结点进行合并， 4-结点需要分裂（颜色翻转）

![4node-split.jpeg](https://i.loli.net/2021/05/06/jwYiLDSHAP2huCB.jpg)

### Ⅱ. 删除后的平衡性维持

删除操作同`BST`大体相同，不过在我们的代码实现里，被删除的永远是一个替代结点。因为红黑树由2-3-4树等价而来，我们要充分对其性质进行利用。

那么替代结点究竟如何得到呢？我们先查找到指定删除的结点`d`，如果该结点有两个孩子，则替代结点为其前驱或后继结点；如果该结点只有一个孩子，该孩子就是替代结点；若`d`没有孩子（即`d`为根结点），则替代结点就是其本身。

因为`2-3-4`树的完美平衡性质，我们可以得到一个结论：真正被删除的元素一定是`2-3-4树`中的叶子节点中的`KEY`，也即在红黑树中删除的（替代结点）一定是叶子节点或者叶子结点上面的双亲结点（最多只有一个孩子）。

进一步得出以下推论：若替代结点的左孩子不为空，则替代结点一定是一个前驱结点；反之，是一个后继结点。删除后用其子节点替代自己的位置。

而这又对应了`2-3-4树`中叶子结点的状况：

- `2-结点`：删除后兄弟够借，经过双亲借；兄弟不够，啃老来凑
- 非`2-结点`：自己足够，直接删除

而后我们的删除策略就变成了（以用前驱结点替代为例）：

- 若删除的替代结点有孩子，且替代结点为黑色，将孩子变为黑色来替代自己
- 删除的是一个叶子节点，且这个替代结点为黑色，在删除前先进行平衡修复（避免提前删除导致信息丢失）

接下来讨论上述第二种情况：需要平衡修复的情况。

#### a. 兄弟够借

向兄弟结点借之前得先知道兄弟手里够不够吧，所以需要找到真正的兄弟结点，因为若当前替代结点的兄弟是红色结点的话，对应2-3-4树，在红黑树中根本不是真正的兄弟结点。如下图第二种情况（图中`p`表示`parent结点`哦）：

![find-bro.jpeg](https://i.loli.net/2021/05/06/ovUeGrXcxKSFwu5.jpg)

这时我们只需对替代结点的`parent`结点做一次旋转操作即可（出现红链接了）,如下所示：

![find-real-bro.jpeg](https://i.loli.net/2021/05/06/jgHPDvSGsUh86Ip.jpg)

找到真正的兄弟结点之后，如果兄弟结点的孩子中有红色结点，就对应了`2-3-4树`中非`2-结点`可借的情况。

既然兄弟够借，那么对应`2-3-4树`中，就是兄弟中`KEY`最接近自己的上去顶替`parent`，`parent`下来顶替被删的结点。

为了实现起来简单，这里有一个小trick：通过旋转将兄弟与兄弟孩子结点达到某种关系后，再旋转`parent`结点。这样做可以达到在对应`2-3-4树`中如果有兄弟有两个孩子就从兄弟那里借两个来的效果，而且可以大大的简化我们的代码。

例如被删结点是其`parent`的左孩子，找到其真正的右兄弟后，如果右兄弟有两个红色子节点，直接旋转`parent`，这样，兄弟结点中只会留下右孩子（较大的`KEY`），兄弟结点上升代替`parent`，`parent`与兄弟左孩子下来代替删除的结点，左孩子没有的话也没关系；另一种需要提前调整的情况是，右兄弟只有一个左孩子可借，这时候我们对右兄弟进行右旋后再对`parent`进行左旋即可。

然而我们前面说：旋转操作是针对红链接的。然而我们在这里其实很幸运地跳出了这个规则的束缚,只需做小许额外的步骤：

- 对`parent`旋转之后， 兄弟顶上去替代`parent`的位置， 兄弟被染成了`parent`原来的颜色  ——  上去顶替,没毛病

- 但`parent`变成了红色，不过因为`parent`是下来顶替被删的黑色结点，故需将其染成黑色

- 最后，将兄弟结点之前的右孩子标为黑色 —— 只剩下它了，作为一个`2-结点`

若被删结点是其`parent`的右孩子，把上面的逻辑反过来左右对调即可。

如果真兄弟结点的两个孩子均为黑色，就对应了`2-结点`、不可借的情况，我们将在下面继续进行讨论。

#### b. 兄弟不够

兄弟无法借过来，对应`2-3-4树`中，兄弟就得找`parent`合并。所以直接将兄弟结点变成红色，交由上层解决。若`parent`也是红色，就把`parent`变成黑色。若`parent`本身就是黑色，就得让`parent`再找他自己的兄弟结点借。

![Delete.jpeg](https://i.loli.net/2021/05/08/ElLRvKUJpHVMZmk.jpg)             

## 3. 代码实现

```go
package rbt

const (
	RED   = true
	BLACK = false
)

type (
	// KeyType is the type with CompareTo behavior.
	KeyType interface {
		CompareTo(c interface{}) int
	}

	// KeyTypeInt is the int that implements the KeyType interface.
	KeyTypeInt int

	// node is the red-black tree node
	node struct {
		Key    KeyType
		Val    interface{}
		Color  bool
		Left   *node
		Right  *node
		Parent *node
	}

	// RBT is the red-black tree
	RBT struct {
		root *node
		size int
	}
)

// CompareTo implementation for type KeyTypeInt
func (k KeyTypeInt) CompareTo(c interface{}) int {
	return int(k) - int(c.(KeyTypeInt))
}

// isRed returns whether a node is in red color, nil is black
func isRed(n *node) bool {
	if n == nil {
		return BLACK
	}
	return n.Color
}

// NewRBT returns a red-black tree
func NewRBT() *RBT {
	return &RBT{}
}

// treeMin find the minimum node in n's sub-tree
// func (n *node) treeMin() *node {
// 	for n.Left != nil {
// 		n = n.Left
// 	}
// 	return n
// }

// treeMax find the maximum node in n's sub-tree
func (n *node) treeMax() *node {
	for n.Right != nil {
		n = n.Right
	}
	return n
}

// predecessor returns n's predecessor node
func (n *node) predecessor() *node {
	if n == nil {
		return nil
	}

	if n.Left != nil {
		return n.Left.treeMax()
	} else {
		p := n.Parent
		lc := n
		for p != nil && lc == p.Left {
			lc = p
			p = p.Parent
		}
		return p
	}
}

// successor returns n's successor node
// func (n *node) successor() *node {
// 	if n == nil {
// 		return nil
// 	}

// 	if n.Right != nil {
// 		return n.Right.treeMin()
// 	} else {
// 		p := n.Parent
// 		lc := n
// 		for p != nil && lc == p.Right {
// 			lc = p
// 			p = p.Parent
// 		}
// 		return p
// 	}
// }

// LeftRotate left rotate the node n, acting on a red link
//
//    p               p
//    |               |
//    n               m
//   / \             / \
//  nl  m    ==>    n  mr
//     / \         / \
//    ml mr       nl ml
//
func (t *RBT) leftRotate(n *node) {
	m := n.Right
	n.Right = m.Left
	if m.Left != nil {
		m.Left.Parent = n
	}
	m.Parent = n.Parent
	if n.Parent == nil {
		t.root = m
	} else if n == n.Parent.Left {
		n.Parent.Left = m
	} else {
		n.Parent.Right = m
	}
	m.Left = n
	n.Parent = m
	// repaint color
	m.Color = n.Color
	n.Color = RED
}

// RightRotate right rotate the node n, acting on a red link
//
//     p                p
//     |                |
//     n                m
//    / \              / \
//   m  nr    ==>     ml  n
//  / \                  / \
// ml mr                mr nr
//
func (t *RBT) rightRotate(n *node) {
	m := n.Left
	n.Left = m.Right
	if m.Right != nil {
		m.Right.Parent = n
	}
	m.Parent = n.Parent
	if n.Parent == nil {
		t.root = m
	} else if n == n.Parent.Left {
		n.Parent.Left = m
	} else {
		n.Parent.Right = m
	}

	m.Right = n
	n.Parent = m
	// repaint color
	m.Color = n.Color
	n.Color = RED
}

func (t *RBT) flipColors(n *node) {
	n.Color = RED
	n.Left.Color = BLACK
	n.Right.Color = BLACK
}

// Search returns the node by the given key if it exists
func (t *RBT) Search(key KeyType) *node {
	root := t.root
	for root != nil {
		cmp := key.CompareTo(root.Key)
		if cmp < 0 {
			root = root.Left
		} else if cmp > 0 {
			root = root.Right
		} else {
			return root
		}
	}
	return root
}

// Insert inserts a key with associated data(val) to a place in the red-black
// tree by compare the keys, and if the key exists, update it with the new val.
func (t *RBT) Insert(key KeyType, val interface{}) {
	if t.root == nil {
		t.root = &node{Key: key, Val: val}
		t.size = 1
		return
	}

	// find parent node to attach
	root := t.root
	var p *node
	for root != nil {
		p = root
		cmp := key.CompareTo(root.Key)
		if cmp < 0 {
			root = root.Left
		} else if cmp > 0 {
			root = root.Right
		} else {
			root.Val = val
			return
		}
	}

	newnode := &node{Key: key,
		Val:    val,
		Color:  RED,
		Parent: p,
	}

	cmp := key.CompareTo(p.Key)
	if cmp < 0 {
		p.Left = newnode
	} else {
		p.Right = newnode
	}

	t.insertFix(newnode)
	t.size++
}

func (t *RBT) insertFix(n *node) {
	var u *node // n's uncle node
	for n.Parent != nil && isRed(n.Parent) {
		if n.Parent == n.Parent.Parent.Left {
			u = n.Parent.Parent.Right
			if u != nil && isRed(u) {
				n = n.Parent.Parent
				t.flipColors(n)
			} else {
				if n == n.Parent.Right {
					n = n.Parent
					t.leftRotate(n)
				}
				t.rightRotate(n.Parent.Parent)
			}
		} else {
			u = n.Parent.Parent.Left
			if u != nil && isRed(u) {
				n = n.Parent.Parent
				t.flipColors(n)
			} else {
				if n == n.Parent.Left {
					n = n.Parent
					t.rightRotate(n)
				}
				t.leftRotate(n.Parent.Parent)
			}
		}
	}
	// root should be black node at the end
	t.root.Color = BLACK
}

// Remove delete a node by the given key if it exits
func (t *RBT) Remove(key KeyType) {
	d := t.Search(key)

	if d == nil {
		return // not found
	}

	// find replacement node
	// in our code, if finally the rpl hold a child node, the rpl must be a
	// predecessor or successor
	rpl := d
	if rpl.Left != nil && rpl.Right != nil {
		// rpl = rpl.successor()
		rpl = rpl.predecessor()
	} else {
		if rpl.Left != nil {
			rpl = rpl.Left
		} else if rpl.Right != nil {
			rpl = rpl.Right
		}
	}
	// delete replacement node
	if rpl != t.root {
		if d != rpl { // d is not a leaf node
			d.Key = rpl.Key
			d.Val = rpl.Val
		}

		if rpl.Left != nil { // rpl is a predecessor
			if !isRed(rpl) {
				rpl.Left.Color = BLACK
			}

			rpl.Left.Parent = rpl.Parent
			if rpl == rpl.Parent.Left {
				rpl.Parent.Left = rpl.Left
			} else {
				rpl.Parent.Right = rpl.Left
			}
			// unlink rpl
			rpl.Parent = nil
			rpl.Left = nil
		} else {
			// rpl is a leaf node
			// fix
			if !isRed(rpl) {
				t.removeFix(rpl)
			}
			// then delete
			if rpl == rpl.Parent.Left {
				rpl.Parent.Left = nil
			} else {
				rpl.Parent.Right = nil
			}
			// unlink rpl
			rpl.Parent = nil
		}
	} else { // single node tree
		t.root = nil
	}
	t.size--
}

// fixAfterRemove do fix if the deleted node is in black color
func (t *RBT) removeFix(n *node) {
	for n != t.root && !isRed(n) {
		if n == n.Parent.Left {
			rBro := n.Parent.Right
			// find real brother node
			if isRed(rBro) {
				t.leftRotate(n.Parent)
				rBro = n.Parent.Right
			}
			if !isRed(rBro.Left) && !isRed(rBro.Right) { // can't borrow
				rBro.Color = RED
				n = n.Parent
			} else { // can borrow
				// 3-node
				if !isRed(rBro.Right) {
					t.rightRotate(rBro)
					rBro = n.Parent.Right
				}
				t.leftRotate(n.Parent)
				// since we pull down n's parent to replace n and
				// we borrow two n's bro's childern, we need repaint
				n.Parent.Color = BLACK
				rBro.Right.Color = BLACK
				n = t.root
			}
		} else {
			lBro := n.Parent.Left

			if isRed(lBro) {
				t.rightRotate(n.Parent)
				lBro = n.Parent.Left
			}
			if !isRed(lBro.Left) && !isRed(lBro.Right) {
				lBro.Color = RED
				n = n.Parent
			} else {
				if !isRed(lBro.Left) {
					t.leftRotate(lBro)
					lBro = n.Parent.Left
				}
				t.rightRotate(n.Parent)
				n.Parent.Color = BLACK
				lBro.Left.Color = BLACK
				n = t.root
			}
		}
	}
	n.Color = BLACK
}
```

## 4. Do something fun：红黑树可视化

用[graphviz](https://github.com/mzohreva/GoGraphviz)库（需自行安装`dot`工具包）给我们的红黑树实现了`Visualize`方法：

```go
func (t *RBT) Visualize() {
	G := visuzlizeTree(t.root)
	G.GenerateImage("dot", "rbt.svg", "svg")
}

func visuzlizeTree(root *node) *graphviz.Graph {
	G := &graphviz.Graph{}
	addSubTree(root, G)
	G.DefaultNodeAttribute(graphviz.Shape, graphviz.ShapeCircle)
	G.DefaultNodeAttribute(graphviz.FontName, "JetBrainsMono Nerd Font")
	G.GraphAttribute(graphviz.NodeSep, "0.3")
	G.DefaultNodeAttribute(graphviz.Style, graphviz.StyleFilled)
	G.DefaultNodeAttribute(graphviz.FillColor, "#B7BBBC")
	// G.MakeDirected()
	return G
}

func addSubTree(root *node, G *graphviz.Graph) int {
	if root == nil {
		null := G.AddNode("")
		G.NodeAttribute(null, graphviz.Shape, graphviz.ShapePoint)
		return null
	}
	rootnode := G.AddNode(fmt.Sprint(root.Key))
	if root.isRed() {
		G.NodeAttribute(rootnode, graphviz.Style, graphviz.StyleFilled)
		G.NodeAttribute(rootnode, graphviz.FillColor, "#F8BCC2")
	}
	leftnode := addSubTree(root.Left, G)
	rightnode := addSubTree(root.Right, G)
	G.AddEdge(rootnode, leftnode, "")
	G.AddEdge(rootnode, rightnode, "")
	return rootnode
}
```

调用生成`rbt.svg`：

![rbt.png](https://i.loli.net/2021/05/06/Pe5RWiYyrMhl1Gj.png)


## 5. 总结

本文沿着红黑树同`2-3-4树`的关系脉络来讨论了红黑树里的插入及删除操作，其可能涉及的情况十分繁多，本文亦不能做到面面俱到。手绘的图也只展示了部分情况，力图通过简单的梳理来理解红黑树平衡性的本质。假如如果你此前对于`B树`已有很深入的理解的话，相信理解红黑树也不会花费太多时间。


