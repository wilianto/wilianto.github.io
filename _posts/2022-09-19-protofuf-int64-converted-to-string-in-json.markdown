---
layout: post
title:  "Int64 Protobuf is converted to string in JSON, Why?"
date:   2022-09-19 00:06:36 +0800
categories: protobuf
---

Today I learn that int64 protobuf is converted to string when we encode it to JSON. This cause some issue on the javascript client side when we assume that the value we receive already in int64 instead of string.

Here is a snippet of the code. 

```protobuf
syntax = "proto3";

option go_package="wilianto.com/proto-int64/order";

message Money {
    string currency = 1;
    int64 amount = 2;
    int32 precision = 3;
}
```

```golang
package main

import (
	"fmt"

	"google.golang.org/protobuf/encoding/protojson"
	"wilianto.com/proto-int64/order"
)

func main() {
	money := order.Money{
		Currency:  "SGD",
		Precision: 2,
		Amount:    10050,
	}
	moneyJSON, _ := protojson.Marshal(&money)
	fmt.Println(string(moneyJSON))
	// output
	// {"currency":"SGD","amount":"10050","precision":2}
}
```

After reading again the protobuf to JSON mapping doc [here](https://developers.google.com/protocol-buffers/docs/proto3#json), it's expected that int64, fixed64 & uint64 are gonna be converted to decimal string. However, it does not work that way for int32, fixed32 & uint32.

<img src="/assets/images/protobuf_int64.png" alt="protobuf int64 doc" title="protobuf int64 doc">

## But why 64-bit number is serialized to string?

Based from what I understand, the issue is not from the protobuf. Instead the main reason is on the JSON protocol itself. 

From JSON [RFC-8259](https://datatracker.ietf.org/doc/html/rfc8259#section-6), we can learn that most of common software implement [IEEE 754 double precision floating point](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) to handle such a precision number. It works well to handle integers with range [-(2^53)+1, (2^53)-1] but not with the long PI number for example (3.141592653589793238462643383279). It may indicate potential interoperability problems.

So, to keep the precision between different system/software implementation, JSON with 64-bit precision should be written as string.

## Reference
- [https://groups.google.com/g/protobuf/c/4-BY-k-Lk-g/m/DhZRLBxDDAAJ](https://groups.google.com/g/protobuf/c/4-BY-k-Lk-g/m/DhZRLBxDDAAJ)
- [https://en.wikipedia.org/wiki/Double-precision_floating-point_format](https://en.wikipedia.org/wiki/Double-precision_floating-point_format)
- [https://datatracker.ietf.org/doc/html/rfc8259#section-6](https://datatracker.ietf.org/doc/html/rfc8259#section-6)