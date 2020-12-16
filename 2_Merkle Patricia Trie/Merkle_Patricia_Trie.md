# Merkle Patricia Trie

Merkle Patricia Trie(MPT) 是一种 cryptographically authenticated 的数据结构，融合了 Merkle 树和前缀树的结构优点，提供了 O(logn) 的操作复杂度，在以太坊中被用来组织管理账户数据，生成交易集合哈希。

## 前言

我们先了解一下 Merkle 树和前缀树的结构特点

- 前缀树(trie)，用于保存关联数组，其 key 的内容为字符串，被编码在根节点到该节点的路径中。其查找效率为O(m), m为 key 的长度。但是前缀树存在一个主要的限制：如果只存放一个键值对效率低下
- Merkle 树是用于比特币网络中的一种数据结构，用于归纳一个区块中的所有交易，同时生成整个交易集合的数字指纹。

## 结构

MPT 的一个节点是以下四种之一：

### NULL 节点

即空节点

### Branch 节点

A 17-item node [ v0 ... v15, vt ]

```go
type fullNode struct {
    Children [17]node // Actual trie node data to encode/decode (needs custom encoder)
    flags    nodeFlag // nodeFlag contains caching-related metadata about a node.
}

type nodeFlag struct {
    hash  hashNode // cached hash of the node (may be nil)
    dirty bool     // whether the node has changes that must be written to the database
}
```

与前缀树相同，MPT同样是把key-value数据项的key编码在树的路径中，但是key的每一个字节值的范围太大（0-127），因此在以太坊中，在进行树操作之前，首先会进行一个key编码的转换，将一个字节的高低四位内容分拆成两个字节存储。通过编码转换，key 的每一位的值范围都在[0, 15]内。因此，一个分支节点的孩子至多只有16个。以太坊通过这种方式，减小了每个分支节点的容量，但是在一定程度上增加了树高。最后一个 children 元素是用来存储自身的内容

另外，每个分支节点会有一个附带的字段 nodeFlag，用来记录一下内容：

- Hash of node：若该字段不为空，则当需要进行哈希计算时，可以跳过计算过程而直接使用上次计算的结果（当节点变脏时，该字段被置空）
- Dirty flag：当一个节点被修改时，该标志位被置为1

### Leaf & Extension 结点

- leaf: A 2-item node [ encodedPath, value ]
- extension: A 2-item node [ encodedPath, key ]

leaf 和 extension 结点共享一套定义。在以太坊中，通过在Key中加入特殊的标志来区分两种类型的节点

```go
type shortNode struct {
	Key   []byte
	Val   node
    flags nodeFlag
}
```

当MPT试图插入一个节点，插入过程中发现目前没有与该节点Key拥有相同前缀的路径。此时MPT把剩余的Key存储在叶子／扩展节点的Key字段中，实现了编码路径的压缩

此外Val字段用来存储叶子／扩展节点的内容，对于叶子节点来说，该字段存储的是一个数据项的内容；而对于扩展节点来说，该字段可以是以下两种内容

- Val字段存储的是其孩子节点在数据库中存储的索引值（其实该索引值也是孩子节点的哈希值）
- Val字段存储的是其孩子节点的引用

## 操作

Ethereum MPT 常用函数接口定义如下：

```go
func (t *Trie) Get(key []byte) []byte
func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error)
func (t *Trie) Delete(key []byte)
func (t *Trie) Update(key, value []byte)
func (t *Trie) Commit(onleaf LeafCallback) (root common.Hash, err error)
```

### Get

Get 操作返回 trie 中保存的key值

从根节点开始搜寻与搜索路径内容一致的路径
- 若当前节点为叶子节点，存储的内容是数据项的内容，且搜索路径的内容与叶子节点的key一致，则表示找到该节点；反之则表示该节点在树中不存在
- 若当前节点为扩展节点，且存储的内容是哈希索引，则利用哈希索引从数据库中加载该节点，再将搜索路径作为参数，对新解析出来的节点递归地调用查找函数
- 若当前节点为扩展节点，存储的内容是另外一个节点的引用，且当前节点的key是搜索路径的前缀，则将搜索路径减去当前节点的key，将剩余的搜索路径作为参数，对其子节点递归地调用查找函数；若当前节点的key不是搜索路径的前缀，表示该节点在树中不存在
- 若当前节点为分支节点，若搜索路径为空，则返回分支节点的存储内容；反之利用搜索路径的第一个字节选择分支节点的孩子节点，将剩余的搜索路径作为参数递归地调用查找函数

### Insert

- 根据Get中描述的查找步骤，首先找到与新插入节点拥有最长相同路径前缀的节点，记为Node
- 若该Node为分支节点
  - 剩余的搜索路径不为空，则将新节点作为一个叶子节点插入到对应的孩子列表中
  - 剩余的搜索路径为空（完全匹配），则将新节点的内容存储在分支节点的第17个孩子节点项中（Value）
- 若该节点为叶子／扩展节点
  - 剩余的搜索路径与当前节点的key一致，则把当前节点Val更新即可
  - 剩余的搜索路径与当前节点的key不完全一致，则将叶子／扩展节点的孩子节点替换成分支节点，将新节点与当前节点key的共同前缀作为当前节点的key，将新节点与当前节点的孩子节点作为两个孩子插入到分支节点的孩子列表中，同时当前节点转换成了一个扩展节点（若新节点与当前节点没有共同前缀，则直接用生成的分支节点替换当前节点）
- 若插入成功，则将被修改节点的dirty标志置为true，hash标志置空（之前的结果已经不可能用），且将节点的诞生标记更新为现在

### Delete

- 根据 Get中描述的查找步骤，找到与需要插入的节点拥有最长相同路径前缀的节点，记为Node
- 若Node为叶子／扩展节点
  - 若剩余的搜索路径与node的Key完全一致，则将整个node删除
  - 若剩余的搜索路径与node的key不匹配，则表示需要删除的节点不存于树中，删除失败
  - 若node的key是剩余搜索路径的前缀，则对该节点的Val做递归的删除调用
- 若Node为分支节点
  - 删除孩子列表中相应下标标志的节点
  - 删除结束，若Node的孩子个数只剩下一个，那么将分支节点替换成一个叶子／扩展节点
- 若删除成功，则将被修改节点的dirty标志置为true，hash标志置空（之前的结果已经不可能用），且将节点的诞生标记更新为现在

### Update

更新操作就是Insert与Delete的结合。当用户调用Update函数时，若value不为空，则隐式地转为调用Insert；若value为空，则隐式地转为调用Delete

### Commit

Commit函数提供将内存中的MPT数据持久化到数据库的功能，在commit完成后，所有变脏的树节点会重新进行哈希计算，并且将新内容写入数据库；最终新的根节点哈希将被作为MPT的最新状态被返回

## 作用

MPT 的作用主要如下：

- 存储任意长度的key-value键值对数据
- 提供了一种快速计算所维护数据集哈希标识的机制
- 提供了快速状态回滚的机制
- 提供了一种称为默克尔证明的证明方法，进行轻节点的扩展，实现简单支付验证

## References

- [Merkle Patricia Tree 详解](https://ethfans.org/toya/articles/588)
- [Patricia Tree](https://eth.wiki/fundamentals/patricia-tree)
