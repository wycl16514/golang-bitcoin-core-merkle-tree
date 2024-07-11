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



