All kinds of blockchain suffer a common weakness that is shortage of disk volumn, computing power and bandwidth, bitcoin blockchain is not exception.This cause problems when you want to check the data 
integrety, you may need to wait for a long time to download the neccesary data for a simple checking. For example when we using bitcoin for trading, you paid you goods or services with bitcoins, the 
paying record need to append to the bitcoin chain. 

In order to verify that your transaction is record on the chain you may need to sync the whole chain down to your device, this may need weeks and of course unacceptable. Therefore we need some algorithm
to quickly verify that the given info is already on the chain. To make sure some data in the a block and exist on the chain is called proof-of-inclusion, we will going to make clear about how it works

Merkle tree is a kind of data structure used to verify the integrety of a group of objects, in blockchain it is used to verify transactions 
in block, the objects in the group are ordered, then we can use following steps to construct merkle tree:

1, Hash each object in the group using given hash function

2, If there is only one object to be hashed, then it is after the hash the process completed

3, if there are odd number of objects, then after hashing them all, we copy the last hash result and add it to the hashed list
then we have even number of hash values in the list

4, we select a pair of hash results in order, hash the pair and use the result as their parent, this step will half the number of items
in the second layer

5, goto step 2

This is somehow like a reverse of a binary tree, we begin from the lowest level, then select two leaves and construct their parent until
we go to the root. Following is an example of merkle tree:


![image](https://github.com/wycl16514/golang-bitcoin-core-merkle-tree/assets/7506958/4f258c11-4cc2-41e6-b8c7-6e937f9276ef)

The process is just like divide and conqure, if there are to many objects, hash them together is impossible, then we can devide the group
into two, and try to hash each half group then combine the two hash result to produce the top hash, if the sub group is still too large,
then we continue to divide them until we get only one object, then we collect the lower level hash to produce the up lever hash.

We can abstract the process as following:
H: hash function
P: parent hash
L: left hash
R: right hash

Then P=H(L||R), || means connect R to the end of L.

If someone want to proof L is include in P, then he can provide R and P, then we compute L, then connect L and R to compute the hash and 
check the result is P or not. Let's do some code instead of only talking, since computer need code to run, create a folder named merkle tree and create a file with name merkle_tree.go and have following 
code, the first method we write is how to connect give two hash together and compute their parent hash, this is the step 4 aboved:
```go
package merkletree

import (
	ecc "elliptic_curve"
)

func MerkleParent(hash1 []byte, hash2 []byte) []byte {
	buf := make([]byte, 0)
	buf = append(buf, hash1...)
	buf = append(buf, hash2...)
	return ecc.Hash256(string(buf))
}

}
```
Now let's have some test in main.go:
```go
package main

import (
	"encoding/hex"
	"fmt"
	merkle "merkletree"
)

func main() {
	hash1, _ := hex.DecodeString("c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5")
	hash2, _ := hex.DecodeString("c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5")
	parentHash := merkle.MerkleParent(hash1, hash2)
	fmt.Printf("parent hash:%x\n", parentHash)
}
```
Running the code we can get the following result:
```go
parent hash:8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd
```

Now let's see how to do step 3, if there are even number of hashes, we get put two in pair and computing there parent hash using MerkleParent we have done aboved, but if we have odd number, then we need
to duplicate the last one to make it has even number of hashes, following it is the code :
```go
func MerkleParentLevel(hashes [][]byte) [][]byte {
	/*
		if there are even number hashes, we put two as pair and compute their merkle parent,
		if there are odd number, we duplicate the last one, and put two in pair to compute
		merkle parent
	*/
	if len(hashes) == 1 {
		panic("Can't take parent level with onl y 1 item")
	}

	if len(hashes)%2 == 1 {
		//odd number, dpulicate the last one
		hashes = append(hashes, hashes[len(hashes)-1])
	}

	parentLevel := make([][]byte, 0)
	for i := 0; i < len(hashes); i += 2 {
		parent := MerkleParent(hashes[i], hashes[i+1])
		parentLevel = append(parentLevel, parent)
	}

	return parentLevel
}
```
Then we can goto main.go to test aboved code:
```go
hashes := make([][]byte, 0)
	hash, _ := hex.DecodeString("c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5")
	hashes = append(hashes, hash)
	hash, _ = hex.DecodeString("c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5")
	hashes = append(hashes, hash)
	hash, _ = hex.DecodeString("f391da6ecfeed1814efae39e7fcb3838ae0b02c02ae7d0a5848a66947c0727b0")
	hashes = append(hashes, hash)
	hash, _ = hex.DecodeString("3d238a92a94532b946c90e19c49351c763696cff3db400485b813aecb8a13181")
	hashes = append(hashes, hash)
	hash, _ = hex.DecodeString("10092f2633be5f3ce349bf9ddbde36caa3dd10dfa0ec8106bce23acbff637dae")
	hashes = append(hashes, hash)
	parentLevel := merkle.MerkleParentLevel(hashes)
	for _, item := range parentLevel {
		fmt.Printf("parent item: %x\n", item)
	}

```
Run the aboved code we can get the following result:
```go
parent item: 8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd
parent item: 7f4e6f9e224e20fda0ae4c44114237f97cd35aca38d83081c9bfd41feb907800
parent item: 3ecf6115380c77e8aae56660f5634982ee897351ba906a6837d15ebc3a225df0
```

Finally let's see how to get the merkle root, as you can see aboved, each time we compute ParentLevel, the number of hashes is halfed, we repeat MerkleParent on previous result continuously until there 
is only one hash then we done, following is the code:
```go
func MerkleRoot(hashes [][]byte) []byte {
	curLevel := hashes
	for len(curLevel) > 1 {
		curLevel = MerkleParentLevel(curLevel)
	}

	return curLevel[0]
}
```
Then in main.go we can test the code as following:
```go
merkleRootHashes := make([][]byte, 0)
	hash, _ = hex.DecodeString("c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("f391da6ecfeed1814efae39e7fcb3838ae0b02c02ae7d0a5848a66947c0727b0")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("3d238a92a94532b946c90e19c49351c763696cff3db400485b813aecb8a13181")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("10092f2633be5f3ce349bf9ddbde36caa3dd10dfa0ec8106bce23acbff637dae")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("7d37b3d54fa6a64869084bfd2e831309118b9e833610e6228adacdbd1b4ba161")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("8118a77e542892fe15ae3fc771a4abfd2f5d5d5997544c3487ac36b5c85170fc")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("dff6879848c2c9b62fe652720b8df5272093acfaa45a43cdb3696fe2466a3877")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("b825c0745f46ac58f7d3759e6dc535a1fec7820377f24d4c2c6ad2cc55c0cb59")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("95513952a04bd8992721e9b7e2937f1c04ba31e0469fbe615a78197f68f52b7c")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("2e6d722e5e4dbdf2447ddecc9f7dabb8e299bae921c99ad5b0184cd9eb8e5908")
	merkleRootHashes = append(merkleRootHashes, hash)
	hash, _ = hex.DecodeString("b13a750047bc0bdceb2473e5fe488c2596d7a7124b4e716fdd29b046ef99bbf0")
	merkleRootHashes = append(merkleRootHashes, hash)
	merkleRoot := merkle.MerkleRoot(merkleRootHashes)
	fmt.Printf("merkle root is :%x\n", merkleRoot)
```
Run the code and we get the following result:
```go
merkle root is :acbcab8bcc1af95d8d563b77d24c3d19b18f1486383d75a5085c4e86c86beed6
```
Using merkle tree we can achive proof of inclusion, for example you have a list of objects , and you want to check your peer has the same set of objects, then you compute the merkle root on L and 
let the peer compute the merkle root, then you compare two roots, if they are the same, then it is proof your peer also have the same list of objects. In application to the bitcoin, if we want to know
whether a batch of transactions have included in the block or not, we can compute the merkle root instead of getting all transactions from the block and checking them one by one.

As shown by the following image:
![image](https://github.com/wycl16514/golang-bitcoin-core-merkle-tree/assets/7506958/8e79a944-601e-4845-be1e-00c4246c6d2f)

If we have two transactions H(k), H(N), and we want to make sure whether these two transaction have already inlcuded in a block, then the bicoin full node can only return hashes that are represent by the
blue box and we can compute the merkle root for a check in the following step:

1, H(KL) = MerkleParent(H(k), H(L))

2, H(MN) = MerkleParent(H(M), H(N))

3, H(IJKL) = MerkleParent(H(IJ), H(KL))

4, H(MNOP) = MerkleParent(H(MN), H(OP))

5, H(IJKLMNOP) = MerkleParent(H(IJKL), H(MNOP))

6, H(ABCDEFGHIJKLMNOP) = MerkleParent(H(ABCDEFGH), H(IJKLMNOP))

As we can see, the merkle root is a kind of compression algorithm, we need only part of the information can we get to the conclusion. But we still have problems for aboved computation, How do we know we
need to pair H(k), H(L) to get the merkle parent, how do we know we need to pair H(NM), H(OP) to get merkle parent? We need some info th deduce such info.

The info we need is the position of those blue boxes, and we need to define the "position" of the box in the binary tree structure. The "position" of a node in a tree related to how
we "traval" the tree, if you have backgroud of basic data structure and algorithm you will know there is a kind of data structure called "graph", the tree aboved is binary tree and it is a kind of graph.

For graph which contains "nodes and edges", if given one node, we can go to other nodes by using the edges coming from the given node, then we have two kinds of ways to "traval" the graph, breath first 
and depth first:

![image](https://github.com/wycl16514/golang-bitcoin-core-merkle-tree/assets/7506958/09a58967-2323-4838-b1e8-643d119703e9)

As you can see from the image aboved, for breath first traval, we visit the node layer by layer, we first at the root node 0, then we visit all nodes below it that are node 1, 2, 3, then we goto layer 2,
visit all nodes there that are nodes 4,5,6,7. The order for each node which they are visited is their "position". The depth first traval will a little bit complex, when we in a given node, we first check
if we can goto the lower layer from left edge, if we can then we go down to the lower layer. If we can't go down to the lowe layer from the left edge(there is not left edge or we have alread been there),
then we try whether we can go down by using the right edge, if we can then we go down to the lowe layer by using the right edge. Otherwise we go up to the parent node and do the same again, the order for
the node that is being visited is the "position" of that node, you can see nodes will have different order or "position" for these two traval ways.

When the full node return hashes for the blue boxes, it will also return their order under the depth first traval, then we will use the order and the given hash value to reconstruct the merkle root.
Let's go through the whole process step by step:

the first step we need to achive is given a list of objects, we need to convert it to tree like structure:

![merkle tree](https://github.com/user-attachments/assets/07596327-13af-4577-be56-1f40a64bcee3)


As you can see from aboved image, we have 8 nodes in list, if we want to construct a merkle tree from these 8 nodes, we can set these 8 nodes as the leaves, and pair two as a group then "grow"
its parent, therefore for 8 nodes, we can "grow out" 4 parents in the second layer, the same process can repeat again and again until we have only one node. Since the number of nodes is half 
when it comes up to one layer, given N nodes, we can have most int(lg(N)+1) layers

Let's write a function for this, given a number N, we return a two dimension array, the first dimension with number to the layer, and the second dimension with nodes for each layer:
```go
func ConstructTree(n int32) [][][]byte {
	/*
		each element is a hash string with 32 bytes, that's why we need three dimensional
		slice
	*/
	maxDepth := math.Ceil(math.Log2(float64(n))) + 1
	merkleTree := make([][][]byte, 0)
	for depth := 0; depth < int(maxDepth); depth += 1 {
		/*
			in layer t, the number of nodes in the layer should be 2^t,
			for example given list of 27 items, the depth of the tree is ceil(lg(27)) == 5,
			then the whole tree will contains 32 nodes
		*/
		layer := make([][]byte, 0)
		nodesInLayer := int(math.Pow(2.0, float64(depth)))
		for i := 0; i < nodesInLayer; i++ {
			layer = append(layer, []byte{})
		}
		merkleTree = append(merkleTree, layer)
	}

	return merkleTree
}
```

Then we can run the method like following:
```go
func main() {
	merkleTree := merkle.ConstructTree(27)
	for _, level := range merkleTree {
		fmt.Printf("%d\n", level)
	}
}
```
The aboved code will give following output:
[[]]
[[] []]
[[] [] [] []]
[[] [] [] [] [] [] [] []]
[[] [] [] [] [] [] [] [] [] [] [] [] [] [] [] []]

When we convert 27 objects into tree like structure, the first layer will contains one element, the second contains 2, and so on. Now let's coding a merkle tree as following:
```go
type MerkleTree struct {
	total int
	nodes [][][]byte
	//combine currentDepth and currentIndex we point to a given node
	currentDepth int32
	currentIndex int32
	maxDepth     int32
}

func NewMerkleTree(hashes [][]byte) *MerkleTree {
	merkleTree := &MerkleTree{
		total:        len(hashes),
		currentDepth: 0,
		currentIndex: 0,
		maxDepth:     int32(math.Ceil(math.Log2(float64(len(hashes))))),
	}

	merkleTree.nodes = ConstructTree(int32(merkleTree.total))
	//set up the lowest layer
	for idx, hash := range hashes {
		merkleTree.nodes[merkleTree.maxDepth][idx] = hash
	}
	//set up nodes in up layer
	for len(merkleTree.Root()) == 0 {
		if merkleTree.IsLeaf() {
			merkleTree.Up()
		} else {
			/*
				in ordre to compute the hash of current node, we need to get the value of its
				left child and right child, if the left child is nil, then we compute the value
				of its left child, if the right child is nil, then we compute the value of right
				child, when both childs have their value, then we can compute the value of
				current node, this is just like the post-order traval of binary tree
			*/
			leftHash := merkleTree.GetLeftNode()
			rightHash := merkleTree.GetRightNode()
			if len(leftHash) == 0 {
				merkleTree.Left()
			} else if len(rightHash) == 0 {
				merkleTree.Right()
			} else {
				//both left and right childs are ready, set the current node
				merkleTree.SetCurrentNode(MerkleParent(leftHash, rightHash))
				merkleTree.Up()
			}
		}
	}
	return merkleTree
}

func (m *MerkleTree) String() string {
	result := make([]string, 0)
	for depth, level := range m.nodes {
		items := make([]string, 0)
		for index, h := range level {
			short := "nil"
			if len(h) != 0 {
				//only print out first 8 digits of the hash value
				short = fmt.Sprintf("%x...", h[:4])
			}
			if depth == int(m.currentDepth) && index == int(m.currentIndex) {
				//current node is being pointed to,then we just show 6 digits with two *
				items = append(items, fmt.Sprintf("*%x*", h[:3]))
			} else {
				items = append(items, short)
			}
		}

		result = append(result, strings.Join(items, ","))
	}

	return strings.Join(result, "\n")
}

func (m *MerkleTree) Up() {
	//point to current node's parent
	if m.currentDepth > 0 {
		m.currentDepth -= 1
	}
	m.currentIndex /= 2
}

func (m *MerkleTree) Left() {
	m.currentDepth += 1
	m.currentIndex *= 2
}

func (m *MerkleTree) Right() {
	m.currentDepth += 1
	m.currentIndex = m.currentIndex*2 + 1
}

func (m *MerkleTree) Root() []byte {
	return m.nodes[0][0]
}

func (m *MerkleTree) SetCurrentNode(value []byte) {
	m.nodes[m.currentDepth][m.currentIndex] = value
}

func (m *MerkleTree) GetCurrentNode() []byte {
	return m.nodes[m.currentDepth][m.currentIndex]
}

func (m *MerkleTree) GetLeftNode() []byte {
	return m.nodes[m.currentDepth+1][m.currentIndex*2]
}

func (m *MerkleTree) GetRightNode() []byte {
	return m.nodes[m.currentDepth+1][m.currentIndex*2+1]
}

func (m *MerkleTree) IsLeaf() bool {
	return m.currentDepth == m.maxDepth
}

func (m *MerkleTree) RightExist() bool {
	/*
		If the number of leaves is not power of 2, then some nodes may not
		have right child
	*/
	return len(m.nodes[m.currentDepth+1]) > int(m.currentIndex)*2+1
}

```
Then we can test aboved code in main.go as following:
```go
merkleTreeHashes := make([][]byte, 0)
	hash, _ := hex.DecodeString("9745f7173ef14ee4155722d1cbf13304339fd00d900b759c6f9d58579b5765fb")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("5573c8ede34936c29cdfdfe743f7f5fdfbd4f54ba0705259e62f39917065cb9b")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("82a02ecbb6623b4274dfcab82b336dc017a27136e08521091e443e62582e8f05")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("507ccae5ed9b340363a0e6d765af148be9cb1c8766ccc922f83e4ae681658308")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("a7a4aec28e7162e1e9ef33dfa30f0bc0526e6cf4b11a576f6c5de58593898330")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("bb6267664bd833fd9fc82582853ab144fece26b7a8a5bf328f8a059445b59add")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("ea6d7ac1ee77fbacee58fc717b990c4fcccf1b19af43103c090f601677fd8836")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("457743861de496c429912558a106b810b0507975a49773228aa788df40730d41")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("7688029288efc9e9a0011c960a6ed9e5466581abf3e3a6c26ee317461add619a")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("b1ae7f15836cb2286cdd4e2c37bf9bb7da0a2846d06867a429f654b2e7f383c9")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("9b74f89fa3f93e71ff2c241f32945d877281a6a50a6bf94adac002980aafe5ab")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("b3a92b5b255019bdaf754875633c2de9fec2ab03e6b8ce669d07cb5b18804638")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("b5c0b915312b9bdaedd2b86aa2d0f8feffc73a2d37668fd9010179261e25e263")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("c9d52c5cb1e557b92c84c52e7c4bfbce859408bedffc8a5560fd6e35e10b8800")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("c555bc5fc3bc096df0a0c9532f07640bfb76bfe4fc1ace214b8b228a1297a4c2")
	merkleTreeHashes = append(merkleTreeHashes, hash)
	hash, _ = hex.DecodeString("f9dbfafc3af3400954975da24eb325e326960a25b87fffe23eef3e7ed2fb610e")
	merkleTreeHashes = append(merkleTreeHashes, hash)

	tree := merkle.NewMerkleTree(merkleTreeHashes)
	fmt.Printf("%s\n", tree)

```
Running the aboved code we have the following result:
```go
*597c4b*
6382df3f...,87cf8fa3...
3ba6c080...,8e894862...,7ab01bb6...,3df760ac...
272945ec...,9a38d037...,4a64abd9...,ec7c95e1...,3b67006c...,850683df...,d40d268b...,8636b7a3...
9745f717...,5573c8ed...,82a02ecb...,507ccae5...,a7a4aec2...,bb626766...,ea6d7ac1...,45774386...,76880292...,b1ae7f15...,9b74f89f...,b3a92b5b...,b5c0b915...,c9d52c5c...,c555bc5f...,f9dbfafc...
```
