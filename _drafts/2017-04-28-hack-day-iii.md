# Berlin Hack Day III

Biweekly on Fridays some of "the cool guys" (and some of "the others") pool our 10% time and have what we call a Hack Day.

This is not a hackathon, or competition, or themed event - we just use that time all on the same day, sometimes in small groups, and share what we did at the end over some drinks and snacks. No pressure; it's all good.

What do we use our 10% time for? Self-improvement, implementing stuff that might not otherwise happen, learning, knowledge-sharing, _fun_... Here's what we got up to on the most recent.

## Starting this blog!

_Silke, Luis, Max, Jim_

We've kicked around the idea for a while of starting a dev blog to showcase the awesome ways we do things (both technically and process-wise), so here it is! _see how awesome we are_

It is implemented in Github Pages, because one of us already had experience with it and has been a learning experience in how to work with that. We also set up a Trello board to keep track of the process of running this blog and an internal slack channel to discuss it, and wrote up an introductory post.

The name will go unexplained. For now.

## [Internal communication project]

_Tanja, Dario, Alex, Mariano_

TODO - Investigate improving communication between the Berlin and London offices ?

## Improve the recruitment process

_Arian, Isaiah, Sam_

TODO



## Build something in node.js

_Felix, Max_

TODO

## Checkout Elixir

_Lukas_

TODO

## Play with golang

_Phillip_

Going for a small trip
===

since we are writing a couple of services  we must sometimes validate why a specific service is slower than expected and therefor I was looking into a language which provides a toolset to deal with profiling --in this case go lang.

This is just a short insight of my experiences plazing with it therefor this article contains a lot subjective opinions.


For the start we write a not so well written service:

```go

package main

import (
	"encoding/json"
	"flag"
	"fmt"
	"log"
	"net/http"
	"regexp"
	"strconv"
)

func main() {
	host, port := ParseArgs()
	address := fmt.Sprintf("%s:%d", host, port)
	log.Printf("start service %s", address)
	http.HandleFunc("/", handleRoot)
	log.Fatal(http.ListenAndServe(address, nil))
}

type author struct {
	ID   uint64 `json:"id"`
	Name string `json:"name"`
}

// ParseArgs will return the given hostname as well as port, if host is not provided the service will not start
func ParseArgs() (string, int) {
	remoteAddr := flag.String("host", "", "REQUIRED; address to listen to, must be reachable via tcp.")
	port := flag.Int("port", 8080, "the port to listen to.")

	flag.Parse()
	if *remoteAddr == "" {
		flag.PrintDefaults()
		log.Fatalln("host is missing.")
	}
	return *remoteAddr, *port
}

var called uint64 = 0
var knownAuthors []author = []author{}

func handleRoot(w http.ResponseWriter, r *http.Request) {
	respondWithJson := func(d interface{}) {
		b, e := json.Marshal(d)
		if e != nil {
			log.Printf("got error while marshalling %v", d)
			w.WriteHeader(http.StatusInternalServerError)
			return
		}
		_, e = w.Write(b)
		if e != nil {
			log.Printf("got error while marshalling %v", d)
			w.WriteHeader(http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "application/json")
	}
	id := r.URL.Path[1:]
	if match, _ := regexp.MatchString(`^\d*$`, id); !match {
		http.Error(w, "Optional id is invalid", http.StatusBadRequest)
		return
	}
	called++
	switch id {
	case "":
		respondWithJson(knownAuthors)
	default:
		log.Printf("got called %d times", called)
		u, _ := strconv.ParseUint(id, 10, 64)
		a := author{called, fmt.Sprintf("f%d l%d", u, u)}
		knownAuthors = append(knownAuthors, a)
		log.Printf("%+v", &a)
		respondWithJson(a)
	}
}
````

and a test like

```go

package main

import (
	"bufio"
	"fmt"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func TestHandleRoot(t *testing.T) {

	testCases := []struct {
		Name   string
		Query  string
		Verify func(string) bool
	}{
		{"empty root", "/", func(b string) bool { return strings.Contains(b, "[]") }},
		{"with valid id", "/1", func(b string) bool { return strings.Contains(b, "{\"id\":2,\"name\":\"f1 l1\"}") }},
	}
	for _, tc := range testCases {
		rw := httptest.NewRecorder()
		q := fmt.Sprintf("GET %s HTTP/1.0\r\n\r\n", tc.Query)
		handleRoot(rw, req(t, q))
		if !tc.Verify(rw.Body.String())E {
			t.Errorf("%s: unexpected output: %s", tc.Name, rw.Body)
		}

	}

}

func req(t *testing.T, v string) *http.Request {
	req, err := http.ReadRequest(bufio.NewReader(strings.NewReader(v)))
	if err != nil {
		t.Fatal(err)
	}
	return req
}
```

detecting race conditions
===

Even though go has concurrency built-in to the language and automatically parallelizes code as necessary over any available CPUs you can write code with a data race if you're not careful enough.

To detect race conditions we can run our tests with ```-race``` flag, however since it is based on runtime evaluation we need to tweak our test a little:

```go
func TestHandleRoot(t *testing.T) {
	testCases := []struct {
		Name   string
		Query  string
		Verify func(string) bool
	}{
		{"empty root", "/", func(b string) bool { return strings.Contains(b, "[]") }},
		{"with valid id", "/1", func(b string) bool { return strings.Contains(b, "\"name\":\"f1 l1\"}") }},
	}
	for _, tc := range testCases {
		var wg sync.WaitGroup
		for i := 0; i < 2; i++ {
			wg.Add(1)
			go func() {
				defer wg.Done()
				rw := httptest.NewRecorder()
				q := fmt.Sprintf("GET %s HTTP/1.0\r\n\r\n", tc.Query)
				handleRoot(rw, req(t, q))
				if !tc.Verify(rw.Body.String()) {
					t.Errorf("%s: unexpected output: %s", tc.Name, rw.Body)
				}
			}()
		}
		wg.Wait()
	}
}
```

When executing ```go test -race``` you may experience something like:
```
WARNING: DATA RACE
Read at 0x0000009b3918 by goroutine 8:
  _/home/philipp/src/golang_demo.handleRoot()
      /home/philipp/src/golang_demo/main.go:63 +0x16f
  _/home/philipp/src/golang_demo.TestHandleRoot.func3()
      /home/philipp/src/golang_demo/main_test.go:31 +0x295

Previous write at 0x0000009b3918 by goroutine 7:
  _/home/philipp/src/golang_demo.handleRoot()
      /home/philipp/src/golang_demo/main.go:63 +0x18e
  _/home/philipp/src/golang_demo.TestHandleRoot.func3()
      /home/philipp/src/golang_demo/main_test.go:31 +0x295

Goroutine 8 (running) created at:
  _/home/philipp/src/golang_demo.TestHandleRoot()
      /home/philipp/src/golang_demo/main_test.go:36 +0x267
  testing.tRunner()
      /usr/lib/go/src/testing/testing.go:657 +0x107

Goroutine 7 (running) created at:
  _/home/philipp/src/golang_demo.TestHandleRoot()
      /home/philipp/src/golang_demo/main_test.go:36 +0x267
  testing.tRunner()
      /usr/lib/go/src/testing/testing.go:657 +0x107
==================
2017/04/28 21:50:57 got called 3 times
2017/04/28 21:50:57 got called 4 times
2017/04/28 21:50:57 &{ID:4 Name:f1 l1}
==================
WARNING: DATA RACE
Read at 0x00000098d610 by goroutine 10:
  _/home/philipp/src/golang_demo.handleRoot()
      /home/philipp/src/golang_demo/main.go:71 +0x524
  _/home/philipp/src/golang_demo.TestHandleRoot.func3()
      /home/philipp/src/golang_demo/main_test.go:31 +0x295

Previous write at 0x00000098d610 by goroutine 9:
  _/home/philipp/src/golang_demo.handleRoot()
      /home/philipp/src/golang_demo/main.go:71 +0x5da
  _/home/philipp/src/golang_demo.TestHandleRoot.func3()
      /home/philipp/src/golang_demo/main_test.go:31 +0x295

Goroutine 10 (running) created at:
  _/home/philipp/src/golang_demo.TestHandleRoot()
      /home/philipp/src/golang_demo/main_test.go:36 +0x267
  testing.tRunner()
      /usr/lib/go/src/testing/testing.go:657 +0x107

Goroutine 9 (running) created at:
  _/home/philipp/src/golang_demo.TestHandleRoot()
      /home/philipp/src/golang_demo/main_test.go:36 +0x267
  testing.tRunner()
      /usr/lib/go/src/testing/testing.go:657 +0x107
==================
==================
WARNING: DATA RACE
Read at 0x00c42000c860 by goroutine 10:
  runtime.growslice()
      /usr/lib/go/src/runtime/slice.go:82 +0x0
  _/home/philipp/src/golang_demo.handleRoot()
      /home/philipp/src/golang_demo/main.go:71 +0x791
  _/home/philipp/src/golang_demo.TestHandleRoot.func3()
      /home/philipp/src/golang_demo/main_test.go:31 +0x295

Previous write at 0x00c42000c860 by goroutine 9:
  _/home/philipp/src/golang_demo.handleRoot()
      /home/philipp/src/golang_demo/main.go:71 +0x587
  _/home/philipp/src/golang_demo.TestHandleRoot.func3()
      /home/philipp/src/golang_demo/main_test.go:31 +0x295

Goroutine 10 (running) created at:
  _/home/philipp/src/golang_demo.TestHandleRoot()
      /home/philipp/src/golang_demo/main_test.go:36 +0x267
  testing.tRunner()
      /usr/lib/go/src/testing/testing.go:657 +0x107

Goroutine 9 (finished) created at:
  _/home/philipp/src/golang_demo.TestHandleRoot()
      /home/philipp/src/golang_demo/main_test.go:36 +0x267
  testing.tRunner()
      /usr/lib/go/src/testing/testing.go:657 +0x107
==================
2017/04/28 21:50:57 &{ID:4 Name:f1 l1}
--- FAIL: TestHandleRoot (0.00s)
        testing.go:610: race detected during execution of test
FAIL
exit status 1
FAIL    _/home/philipp/src/golang_demo  0.010s
```

So we see that in our program we have two race conditions one is in line 63 and the other one is in 71.

We have at least three options to fix them:
- use channels
- use mutex
- use atomic

For the sake of demonstration we will use 
- mutex for the array
and
- atomic for the called counter
by changing knownAuthors to:
```go

var knownAuthors struct {
	sync.Mutex
	authors []author
}

func handleRoot(w http.ResponseWriter, r *http.Request) {
,,,
switch id {
	case "":
		if knownAuthors.authors == nil {
			respondWithJson([]author{})
		} else {
			respondWithJson(knownAuthors.authors)
		}
	default:
		log.Printf("got called %d times", called)
		u, _ := strconv.ParseUint(id, 10, 64)
		a := author{called, fmt.Sprintf("f%d l%d", u, u)}
		knownAuthors.Lock()
		defer knownAuthors.Unlock()
		knownAuthors.authors = append(knownAuthors.authors, a)
		log.Printf("%+v", &a)
		respondWithJson(a)
	}
}
```

and called to
```go

func handleRoot(w http.ResponseWriter, r *http.Request) {
...
	called := atomic.AddUint64(&called, 1)
 ...
}
```

CPU profiling
===

Besides the runtime profiling (```_ "net/http/pprof"```) there's also a more reliale and faster way to get the needed information. For that we need to extend our test with

```go
func benchWith(q string, b *testing.B) {
	b.ReportAllocs()
	r, err := http.ReadRequest(bufio.NewReader(strings.NewReader(fmt.Sprintf("GET %s HTTP/1.0\r\n\r\n", q))))
	if err != nil {
		b.Fatal(err)
	}
	for i := 0; i < b.N; i++ {
		rw := httptest.NewRecorder()
		handleRoot(rw, r)
	}

}

func BenchmarkRootWithID(b *testing.B) {
	benchWith("/1", b)
}

func BenchmarkRootWithoutID(b *testing.B) {
	benchWith("/", b)
}

```

when we run ```go test -v -run=^$ -bench=BenchmarkRootWithID -benchtime=2s -cpuprofile=prof.cpu -memprofile=prof.mem```
```
$ go tool pprof demo.test prof.cpu

Entering interactive mode (type "help" for commands)
(pprof) top
130ms of 150ms total (86.67%)
Showing top 10 nodes out of 44 (cum >= 10ms)
      flat  flat%   sum%        cum   cum%
      20ms 13.33% 13.33%       30ms 20.00%  regexp.makeOnePass
      20ms 13.33% 26.67%       30ms 20.00%  runtime.mallocgc
      20ms 13.33% 40.00%       20ms 13.33%  runtime.mapassign
      10ms  6.67% 46.67%       10ms  6.67%  fmt.(*pp).printValue
      10ms  6.67% 53.33%       10ms  6.67%  net/http.Header.Get
      10ms  6.67% 60.00%       10ms  6.67%  regexp/syntax.(*compiler).cat
      10ms  6.67% 66.67%       10ms  6.67%  regexp/syntax.Parse
      10ms  6.67% 73.33%       10ms  6.67%  runtime.entersyscall
      10ms  6.67% 80.00%       20ms 13.33%  runtime.growslice
      10ms  6.67% 86.67%       10ms  6.67%  runtime.heapBitsSetType
(pprof) top -cum
0 of 150000000ns total (    0%)
Showing top 10 nodes out of 44 (cum >= 40000000ns)
      flat  flat%   sum%        cum   cum%
         0     0%     0% 150000000ns   100%  runtime.goexit
         0     0%     0% 140000000ns 93.33%  _/home/philipp/src/golang_demo.BenchmarkRootWithID
         0     0%     0% 140000000ns 93.33%  _/home/philipp/src/golang_demo.benchWith
         0     0%     0% 140000000ns 93.33%  _/home/philipp/src/golang_demo.handleRoot
         0     0%     0% 140000000ns 93.33%  testing.(*B).launch
         0     0%     0% 140000000ns 93.33%  testing.(*B).runN
         0     0%     0% 80000000ns 53.33%  regexp.Compile
         0     0%     0% 80000000ns 53.33%  regexp.MatchString
         0     0%     0% 80000000ns 53.33%  regexp.compile
         0     0%     0% 40000000ns 26.67%  regexp/syntax.Compile
(pprof)



```

we see already that compiling the reg ex takes huge amount of time. We can also illustrate it via ```web```

![pprof001.svg](pprof001.svg)

however there's also [https://github.com/uber/go-torch] from uber so that we can get a more dynamic overview.
![torch.svg](torch.svg)

With that information we could refactor our application a bit and run it again:

```go

package main

import (
	"encoding/json"
	"flag"
	"fmt"
	"log"
	"net/http"
	"strconv"
	"sync"
	"sync/atomic"
)

func main() {
	host, port := ParseArgs()
	address := fmt.Sprintf("%s:%d", host, port)
	log.Printf("start service %s", address)
	http.HandleFunc("/", handleRoot)
	log.Fatal(http.ListenAndServe(address, nil))

}

type author struct {
	ID   uint64 `json:"id"`
	Name string `json:"name"`
}

// ParseArgs will return the given hostname as well as port, if host is not provided the service will not start
func ParseArgs() (string, int) {
	remoteAddr := flag.String("host", "", "REQUIRED; address to listen to, must be reachable via tcp.")
	port := flag.Int("port", 8080, "the port to listen to.")

	flag.Parse()
	if *remoteAddr == "" {
		flag.PrintDefaults()
		log.Fatalln("host is missing.")
	}
	return *remoteAddr, *port
}

var called uint64 = 0
var knownAuthors struct {
	sync.Mutex
	authors []author
}

func handleRoot(w http.ResponseWriter, r *http.Request) {
	respondWithJson := func(d interface{}) {
		b, e := json.Marshal(d)
		if e != nil {
			log.Printf("got error while marshalling %v", d)
			w.WriteHeader(http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		_, e = w.Write(b)
		if e != nil {
			log.Printf("got error while writing %v", b)
			w.WriteHeader(http.StatusInternalServerError)
			return
		}
	}
	id := r.URL.Path[1:]
	called := atomic.AddUint64(&called, 1)
	log.Printf("got called %d times", called)
	switch id {
	case "":
		if knownAuthors.authors == nil {
			respondWithJson([]author{})
		} else {
			respondWithJson(knownAuthors.authors)
		}
	default:
		u, err := strconv.ParseUint(id, 10, 64)
		if err != nil {
			http.Error(w, "Optional id is invalid", http.StatusBadRequest)
		} else {
			a := author{called, fmt.Sprintf("f%d l%d", u, u)}
			knownAuthors.Lock()
			defer knownAuthors.Unlock()
			knownAuthors.authors = append(knownAuthors.authors, a)
			respondWithJson(a)
		}
	}
}
```

Summary
===

Comming from a Java dominated World I really enjoyed working with go. I think it is easy to read and toolset is just amazing.
Some people would may miss generics but in my oppinion most cases could be handled via the concept of dynamic interface association. 



