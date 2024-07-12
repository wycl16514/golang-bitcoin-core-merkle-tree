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

8, the following chunk of data are hash values of all transaction, its length is 32 * value from step 7:

0aba412a0d1480e370173072c9562becffe87aa661c1e4a6dbc305d38ec5dc088a7cf92e6458aca7b32edae818f9c2c98c37e06bf72ae0ce80649a386
55ee1e27d34d9421d940b16732f24b94023e9d572a7f9ab8023434a4feb532d2adfc8c2c2158785d1bd04eb99df2e86c54bc13e139862897217400def5d72c280222c4cbaee7261831
e1550dbb8fa82853e9fe506fc5fda3f7b919d8fe74b6282f92763cef8e625f977af7c8619c32a369b832bc2d051ecd9c73c51e76370ceabd4f25097c256597fa898d404ed53425de608
ac6bfe426f6e2bb457f1c554866eb69dcb8d6bf6f880e9a59b3cd053e6c7060eeacaacf4dac6697dac20e4bd3f38a2ea2543d1ab7953e3430790a9f81e1c67f5b58c825acf46bd02848
384eebe9af917274cdfbb1a28a5d58a23a17977defode10d644258d9c54f886d47d293a411cb622610

9, the final 4 bytes is named flag bits: 3b55635

The first 6 fields are the same as getheader command, the last 4 fields are use for proof-of-inclusion. The value from step 7 is the number of hash values in the list as we metioned in
previous sector,

