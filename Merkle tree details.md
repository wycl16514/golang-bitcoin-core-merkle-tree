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
code, the first method we write is how to connect give two hash together and compute their parent hash:
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

