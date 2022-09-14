---
title: "Who Am I???"
date: 2022-09-03T11:34:53+02:00
draft: false
---

I am ahmed

I am a programmer
here is how code works:
```go
package main

import (
	"log"
	"os"
	"strconv"
	"sync"
)

var balance int = 10

func buy(price int, tx string, mu *sync.Mutex) bool {
	mu.Lock()
	defer mu.Unlock()

	if balance < price {
		log.Printf("%s, refused due to insufficient funds", tx)

		return false
	}

	balance -= price
	log.Printf("%s, succeeded", tx)

	return true
}

func main() {
	enforceSdkSign, err := strconv.ParseBool(os.Getenv("ENFORCE_SDK_SIGN"))
	if err != nil {
		enforceSdkSign = false
	}
	log.Printf("ENFORCE_SDK_SIGN %t", enforceSdkSign)

}

```

Now, good bye