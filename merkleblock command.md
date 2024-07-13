We have built up merkle tree in previous section, in this section we will see how to use the tree to verify proof-of-inclusion in bitcoin blockchain. As we have shown in previous section,
When we get a list of hashes, we can build up the merkle tree and get the root value. The question is how we get those list of hash values, actually we can send a "getdata" command with
given transaction hash to a full node, then it will response with a "mekleblock" command, and a list of hash values will be contained in the body of command.

First we check an example of merkleblock command, following is the binary data of the body of merkleblock command:

```go
00000020df3b053dc46f162a9b00c7f0d5124e2676d47bbe7c5d0793a500000000000000ef445fef2ed495c275892206ca533e7411907971013ab83e3b47bded692d14d4dc7c835b
67d8001ac157e670bfOd00000aba412a0d1480e370173072c9562becffe87aa661c1e4a6dbc305d38ec5dc088a7cf92e6458aca7b32edae818f9c2c98c37e06bf72ae0ce80649a386
55ee1e27d34d9421d940b16732f24b94023e9d572a7f9ab8023434a4feb532d2adfc8c2c2158785d1bd04eb99df2e86c54bc13e139862897217400def5d72c280222c4cbaee7261831
e1550dbb8fa82853e9fe506fc5fda3f7b919d8fe74b6282f92763cef8e625f977af7c8619c32a369b832bc2d051ecd9c73c51e76370ceabd4f25097c256597fa898d404ed53425de608
ac6bfe426f6e2bb457f1c554866eb69dcb8d6bf6f880e9a59b3cd053e6c7060eeacaacf4dac6697dac20e4bd3f38a2ea2543d1ab7953e3430790a9f81e1c67f5b58c825acf46bd02848
384eebe9af917274cdfbb1a28a5d58a23a17977defode10d644258d9c54f886d47d293a411cb6226103b55635
```

Let's put the chunk of binary data aboved into fields:

1, the first 4 bytes in little endian format is version number: 0000002

2, following 32 bytes in little endian format is id of previous block: 
0df3b053dc46f162a9b00c7f0d5124e2676d47bbe7c5d0793a500000000000000

3, the following 32 bytes in little endian format is value of merkle root:
ef445fef2ed495c275892206ca533e7411907971013ab83e3b47bded692d14d4

4, following 4 bytes in little endian format is timestamp: dc7c835b

5, the following 4 bytes is named bits: 67d8001a

6, the following 4 bytes is named nonce: c157e670

7, the following 4 bytes in little endian format is number of total transaction: bfOd0000

8, the following 1 bytes is variant int, the number of hashes: 0a

9, the following chunk of data are hash values of all transaction, its length is 32 * value from step 7:

ba412a0d1480e370173072c9562becffe87aa661c1e4a6dbc305d38ec5dc088a7cf92e6458aca7b32edae818f9c2c98c37e06bf72ae0ce80649a386
55ee1e27d34d9421d940b16732f24b94023e9d572a7f9ab8023434a4feb532d2adfc8c2c2158785d1bd04eb99df2e86c54bc13e139862897217400def5d72c280222c4cbaee7261831
e1550dbb8fa82853e9fe506fc5fda3f7b919d8fe74b6282f92763cef8e625f977af7c8619c32a369b832bc2d051ecd9c73c51e76370ceabd4f25097c256597fa898d404ed53425de608
ac6bfe426f6e2bb457f1c554866eb69dcb8d6bf6f880e9a59b3cd053e6c7060eeacaacf4dac6697dac20e4bd3f38a2ea2543d1ab7953e3430790a9f81e1c67f5b58c825acf46bd02848
384eebe9af917274cdfbb1a28a5d58a23a17977defode10d644258d9c54f886d47d293a411cb622610

9, the final 4 bytes is named flag bits: 3b55635

The first 6 fields are the same as getheader command, the last 4 fields are use for proof-of-inclusion. The value from step 7 is the number of hash values in the list as we metioned in
previous sector, Let's write the code for parsing a merkle block, create a new file named merkleblock.go, and have the following code:
```go
package merkletree

import (
	"bufio"
	"bytes"
	"fmt"
	"math/big"
	"strings"
	"transaction"
)

type MerkleBlock struct {
	version           *big.Int
	previousBlock     []byte
	merkleRoot        []byte
	timeStamp         *big.Int
	bits              []byte
	nonce             []byte
	totalTransactions *big.Int
	numHashes         *big.Int
	hashes            [][]byte
	flagBits          []byte
}

func ErrorPanic(err error, msg string) {
	if err != nil {
		panic(msg)
	}
}

func BytesToBitsField(bytes []byte) []string {
	flagBits := make([]string, 0)
	for _, byteVal := range bytes {
		flagBits = append(flagBits, fmt.Sprintf("%08b", byteVal))
	}

	return flagBits
}

func ParseMerkleBlock(payload []byte) *MerkleBlock {
	merkleBlock := &MerkleBlock{}
	reader := bytes.NewReader(payload)
	bufReader := bufio.NewReader(reader)
	version := make([]byte, 4)
	_, err := bufReader.Read(version)
	ErrorPanic(err, "MerkleBlock read version")
	merkleBlock.version = transaction.LittleEndianToBigInt(version, transaction.LITTLE_ENDIAN_4_BYTES)

	prevBlock := make([]byte, 32)
	_, err = bufReader.Read(prevBlock)
	ErrorPanic(err, "MerkleBlock read previous block")
	merkleBlock.previousBlock = transaction.ReverseByteSlice(prevBlock)

	merkleRoot := make([]byte, 32)
	_, err = bufReader.Read(merkleRoot)
	ErrorPanic(err, "MerkleBlock read merkle root")
	merkleBlock.merkleRoot = transaction.ReverseByteSlice(merkleRoot)

	timeStamp := make([]byte, 4)
	_, err = bufReader.Read(timeStamp)
	ErrorPanic(err, "MerkleBlock read time stamp")
	merkleBlock.timeStamp = transaction.LittleEndianToBigInt(timeStamp, transaction.LITTLE_ENDIAN_4_BYTES)

	bits := make([]byte, 4)
	_, err = bufReader.Read(bits)
	ErrorPanic(err, "MerkleBlock read bits")
	merkleBlock.bits = bits

	nonce := make([]byte, 4)
	_, err = bufReader.Read(nonce)
	ErrorPanic(err, "MerkleBlock read nonce")
	merkleBlock.nonce = nonce

	total := make([]byte, 4)
	_, err = bufReader.Read(total)
	ErrorPanic(err, "MerkleBloc read total")
	merkleBlock.totalTransactions = transaction.LittleEndianToBigInt(total, transaction.LITTLE_ENDIAN_4_BYTES)

	numHashes := transaction.ReadVarint(bufReader)
	merkleBlock.numHashes = numHashes

	hashes := make([][]byte, 0)
	for i := 0; i < int(numHashes.Int64()); i++ {
		hash := make([]byte, 32)
		_, err = bufReader.Read(hash)
		ErrorPanic(err, "MerkleBlock read hash")
		hashes = append(hashes, transaction.ReverseByteSlice(hash))
	}
	merkleBlock.hashes = hashes

	flagLen := transaction.ReadVarint(bufReader)
	flags := make([]byte, flagLen.Int64())
	_, err = bufReader.Read(flags)
	ErrorPanic(err, "MerkleBlock read flags")
	merkleBlock.flagBits = flags

	return merkleBlock
}

func (m *MerkleBlock) String() string {
	result := make([]string, 0)
	result = append(result, fmt.Sprintf("version: %x", m.version))
	result = append(result, fmt.Sprintf("previous block: %x", m.previousBlock))
	result = append(result, fmt.Sprintf("merkle root: %x", m.merkleRoot))
	bitsString := strings.Join(BytesToBitsField(m.bits), ",")
	result = append(result, fmt.Sprintf("bits: %s", bitsString))
	result = append(result, fmt.Sprintf("nonce:%x", m.nonce))
	result = append(result, fmt.Sprintf("total tx: %x", m.totalTransactions.Int64()))
	result = append(result, fmt.Sprintf("number of hashes:%d", m.numHashes.Int64()))
	for i := 0; i < int(m.numHashes.Int64()); i++ {
		result = append(result, fmt.Sprintf("%x,", m.hashes[i]))
	}

	result = append(result, fmt.Sprintf("flags: %x", m.flagBits))

	return strings.Join(result, "\n")
}

```
Now we can test the aboved code in main.go:
```go
package main

import (
	"encoding/hex"
	"fmt"
	merkle "merkletree"
)

func main() {
	payload, err := hex.DecodeString("00000020df3b053dc46f162a9b00c7f0d5124e2676d47bbe7c5d0793a500000000000000ef445fef2ed495c275892206ca533e7411907971013ab83e3b47bd0d692d14d4dc7c835b67d8001ac157e670bf0d00000aba412a0d1480e370173072c9562becffe87aa661c1e4a6dbc305d38ec5dc088a7cf92e6458aca7b32edae818f9c2c98c37e06bf72ae0ce80649a38655ee1e27d34d9421d940b16732f24b94023e9d572a7f9ab8023434a4feb532d2adfc8c2c2158785d1bd04eb99df2e86c54bc13e139862897217400def5d72c280222c4cbaee7261831e1550dbb8fa82853e9fe506fc5fda3f7b919d8fe74b6282f92763cef8e625f977af7c8619c32a369b832bc2d051ecd9c73c51e76370ceabd4f25097c256597fa898d404ed53425de608ac6bfe426f6e2bb457f1c554866eb69dcb8d6bf6f880e9a59b3cd053e6c7060eeacaacf4dac6697dac20e4bd3f38a2ea2543d1ab7953e3430790a9f81e1c67f5b58c825acf46bd02848384eebe9af917274cdfbb1a28a5d58a23a17977def0de10d644258d9c54f886d47d293a411cb6226103b55635")
	if err != nil {
		panic(err)
	}

	merkleBlock := merkle.ParseMerkleBlock(payload)
	fmt.Printf("%s\n", merkleBlock)
}
```
Running the aboved code we will have the following result:
```go
version: 20000000
previous block: 00000000000000a593075d7cbe7bd476264e12d5f0c7009b2a166fc43d053bdf
merkle root: d4142d690dbd473b3eb83a0171799011743e53ca06228975c295d42eef5f44ef
bits: 01100111,11011000,00000000,00011010
nonce:c157e670
total tx: 3519
number of hashes:10
8a08dcc58ed305c3dba6e4c161a67ae8ffec2b56c972301770e380140d2a41ba,
7de2e15e65389a6480cee02af76be0378cc9c2f918e8da2eb3a7ac58642ef97c,
c2c2c8df2a2d53eb4f4a432380abf9a772d5e92340b9242f73160b941d42d934,
ba4c2c2280c2725def0d401772896298133ec14bc5862edf99eb04bdd1858715,
ce6327f982624be78f9d917b3fda5ffc06e59f3e8582fab8db50151e836172ee,
9750f2d4abce7063e7513cc7d9ec51d0c22b839b362ac319867caf77f925e6f8,
8dcb9db66e8654c5f157b42b6e6f42fe6bac08e65d4253ed04d498a87f5956c2,
4325eaa2383fbde420ac7d69c6daf4accaea0e06c7e653d03c9ba5e980f8f66b,
4c2717f99abeee84838402bd46cf5a828cb5f5671c1ef8a9900743e35379abd1,
6122b61c413a297dd486f8549c8d2544d610def0de7779a1238ad5a5281abbdf,
flags: b55635

```


