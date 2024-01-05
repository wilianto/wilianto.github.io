---
layout: post
title:  "Big Endian vs Little Endian"
date:   2024-01-05 00:06:36 +0800
categories: byte, golang
permalink: big-endian-vs-little-endian
---

I get a bug today when encoding some message from my IoT device. After reading their doc, turns out they send message in little endian order instead of big endian order.

I write a simple go code to simulate the different between big endian and little endian. 
Big endian sends most significant bytes from right to left.
Little endian sends most least bytes from right to left.

```golang
package main

import (
	"encoding/binary"
	"encoding/hex"
	"fmt"
)

func main() {
	n := 325

	little := make([]byte, 2)
	binary.LittleEndian.PutUint16(little, uint16(n))
	fmt.Println("little hex", hex.EncodeToString(little))
	fmt.Printf("little binary %08b\n", little)

	fmt.Println("===========")

	big := make([]byte, 2)
	binary.BigEndian.PutUint16(big, uint16(n))
	fmt.Println("big hex", hex.EncodeToString(big))
	fmt.Printf("big binary %08b\n", big)
}
```

Output
```
little hex 4501
little binary [01000101 00000001]
===========
big hex 0145
big binary [00000001 01000101]
```

From what I understand there is no hard rule on using either one. But big endian is more common to be used in the network protocol. Meanwhile little endian is more dominant in modern computer architectures processors (x86, ARM), IoT device & embedded system.