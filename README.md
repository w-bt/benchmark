# Benchmark and Optimizing Code

Sharing Session Benchmark and Optimizing Code on 09 November 2018 - Tokopedia Tower 31st Floor

### Requirements

  - Golang 1.11 or newer
  - Any editor (vscode or atom)
  
### Test Case

```golang
package main

import (
	"log"
	"net/http"
	"regexp"
)

var products map[string]*Product

func init() {
	GenerateProduct()
}

func main() {
	log.Printf("Starting on port 1234")
	http.HandleFunc("/product", handleProduct)
	log.Fatal(http.ListenAndServe("127.0.0.1:1234", nil))
}

func handleProduct(w http.ResponseWriter, r *http.Request) {
	code := r.FormValue("code")
	if match, _ := regexp.MatchString(`^[A-Z]{2}[0-9]{2}$`, code); !match {
		http.Error(w, "code is invalid", http.StatusBadRequest)
		return
	}

	result := findProduct(products, code)

	if result.Code == "" {
		http.Error(w, "Data Not Found", http.StatusBadRequest)
		return
	}

	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	w.Write([]byte(`<font size="10">Product Code : ` + result.Code + ` Name :` + result.Name + `</font>`))
}

func findProduct(Products map[string]*Product, code string) Product {
	for _, item := range Products {
		if code == (*item).Code {
			return *item
		}
	}

	return Product{}
}
```
### Data

List of product with Code and Name. See [data.go](https://github.com/w-bt/benchmark/edit/master/data.go). You can assume this data is comming from database or something else.

### Run It
```sh
$ go build && ./benchmark
```
Open browser and hit `http://localhost:1234/product?code={code}`.

### Test
```sh
$ go test -cover -race -v
?   	github.com/w-bt/benchmark	[no test files]
```

### Add Unit Test and Retest
```golang
package main

import (
	"bufio"
	"net/http"
	"net/http/httptest"
	"os"
	"strings"
	"testing"
)

func TestMain(m *testing.M) {
	GenerateProduct()

	code := m.Run()

	os.Exit(code)
}

func TestHandleProduct(t *testing.T) {
	r, _ := http.NewRequest("GET", "/product?code=AA11", nil)
	w := httptest.NewRecorder()
	handleProduct(w, r)
	if !strings.Contains(w.Body.String(), "Product Code") {
		t.Error("An error was not expected")
	}
}
```
```sh
$ go test -cover -race -v
=== RUN   TestHandleProduct
--- PASS: TestHandleProduct (0.00s)
PASS
coverage: 73.3% of statements
ok  	github.com/w-bt/benchmark	2.048s
```

### GO Test Benchmark
```golang
func BenchmarkHandleProduct(b *testing.B) {
	b.ReportAllocs()
	r, _ := http.ReadRequest(bufio.NewReader(strings.NewReader("GET /product?code=AA11 HTTP/1.0\r\n\r\n")))
	for i := 0; i < b.N; i++ {
		rw := httptest.NewRecorder()
		handleProduct(rw, r)
	}
}
```
```sh
$ go test -v -run=^$ -bench=. -benchtime=10s -cpuprofile=prof.cpu -memprofile=prof.mem
goos: linux
goarch: amd64
pkg: github.com/w-bt/benchmark
BenchmarkHandleProduct-4   	    5000	   3220080 ns/op	    4979 B/op	      63 allocs/op
PASS
ok  	github.com/w-bt/benchmark	16.782s
```
This command produces 3 new files:
  - binary test (benchmark.test)
  - cpu profile (prof.cpu)
  - memory profile (prof.mem)

Based on the data above, benchmarking is done by +-10s. Detailed result:
  - total execution: 5000 times
  - duration each operation: 3220080 ns/op
  - each iteration costs: 4979 bytes with 63 allocations

### CPU Profiling
```sh
$ go tool pprof benchmark.test prof.cpu
File: benchmark.test
Type: cpu
Time: Nov 9, 2018 at 2:57am (WIB)
Duration: 16.62s, Total samples = 16.54s (99.53%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 16.21s, 98.00% of 16.54s total
Dropped 64 nodes (cum <= 0.08s)
Showing top 10 nodes out of 14
      flat  flat%   sum%        cum   cum%
     8.24s 49.82% 49.82%      8.24s 49.82%  memeqbody
     4.83s 29.20% 79.02%     16.29s 98.49%  github.com/w-bt/benchmark.findProduct
     2.87s 17.35% 96.37%      2.99s 18.08%  runtime.mapiternext
     0.23s  1.39% 97.76%      0.23s  1.39%  runtime.memequal
     0.04s  0.24% 98.00%      0.09s  0.54%  runtime.scanobject
         0     0% 98.00%     16.41s 99.21%  github.com/w-bt/benchmark.BenchmarkHandleProduct
         0     0% 98.00%     16.41s 99.21%  github.com/w-bt/benchmark.handleProduct
         0     0% 98.00%      0.10s   0.6%  regexp.MatchString
         0     0% 98.00%      0.09s  0.54%  runtime.gcBgMarkWorker
         0     0% 98.00%      0.09s  0.54%  runtime.gcBgMarkWorker.func2
(pprof) top5
Showing nodes accounting for 16.21s, 98.00% of 16.54s total
Dropped 64 nodes (cum <= 0.08s)
Showing top 5 nodes out of 14
      flat  flat%   sum%        cum   cum%
     8.24s 49.82% 49.82%      8.24s 49.82%  memeqbody
     4.83s 29.20% 79.02%     16.29s 98.49%  github.com/w-bt/benchmark.findProduct
     2.87s 17.35% 96.37%      2.99s 18.08%  runtime.mapiternext
     0.23s  1.39% 97.76%      0.23s  1.39%  runtime.memequal
     0.04s  0.24% 98.00%      0.09s  0.54%  runtime.scanobject
(pprof) top5 --cum
Showing nodes accounting for 4.83s, 29.20% of 16.54s total
Dropped 64 nodes (cum <= 0.08s)
Showing top 5 nodes out of 14
      flat  flat%   sum%        cum   cum%
         0     0%     0%     16.42s 99.27%  testing.(*B).runN
         0     0%     0%     16.41s 99.21%  github.com/w-bt/benchmark.BenchmarkHandleProduct
         0     0%     0%     16.41s 99.21%  github.com/w-bt/benchmark.handleProduct
         0     0%     0%     16.40s 99.15%  testing.(*B).launch
     4.83s 29.20% 29.20%     16.29s 98.49%  github.com/w-bt/benchmark.findProduct
```
Detailed informations:
  - flat: the duration of direct logic inside the function
  - flag%: flat percentage `(flat/total flats)*100`
  - sum%: sum of current flag% and previous flag%
  - cum: cumulative duration for the function
  - cum%: cumulative percentage `(cum/total cum)*100`

To see detail duration time for each line, execute this code:
```sh
(pprof) list handleProduct
Total: 16.54s
ROUTINE ======================== github.com/w-bt/benchmark.handleProduct in /home/nakama/Code/go/src/github.com/w-bt/benchmark/main.go
         0     16.41s (flat, cum) 99.21% of Total
         .          .     17:	http.HandleFunc("/product", handleProduct)
         .          .     18:	log.Fatal(http.ListenAndServe("127.0.0.1:1234", nil))
         .          .     19:}
         .          .     20:
         .          .     21:func handleProduct(w http.ResponseWriter, r *http.Request) {
         .       10ms     22:	code := r.FormValue("code")
         .      100ms     23:	if match, _ := regexp.MatchString(`^[A-Z]{2}[0-9]{2}$`, code); !match {
         .          .     24:		http.Error(w, "code is invalid", http.StatusBadRequest)
         .          .     25:		return
         .          .     26:	}
         .          .     27:
         .     16.29s     28:	result := findProduct(products, code)
         .          .     29:
         .          .     30:	if result.Code == "" {
         .          .     31:		http.Error(w, "Data Not Found", http.StatusBadRequest)
         .          .     32:		return
         .          .     33:	}
         .          .     34:
         .          .     35:	w.Header().Set("Content-Type", "text/html; charset=utf-8")
         .       10ms     36:	w.Write([]byte(`<font size="10">Product Code : ` + result.Code + ` Name :` + result.Name + `</font>`))
         .          .     37:}
         .          .     38:
         .          .     39:func findProduct(Products map[string]*Product, code string) Product {
         .          .     40:	for _, item := range Products {
         .          .     41:		if code == (*item).Code {
(pprof) list findProduct
Total: 16.54s
ROUTINE ======================== github.com/w-bt/benchmark.findProduct in /home/nakama/Code/go/src/github.com/w-bt/benchmark/main.go
     4.83s     16.29s (flat, cum) 98.49% of Total
         .          .     35:	w.Header().Set("Content-Type", "text/html; charset=utf-8")
         .          .     36:	w.Write([]byte(`<font size="10">Product Code : ` + result.Code + ` Name :` + result.Name + `</font>`))
         .          .     37:}
         .          .     38:
         .          .     39:func findProduct(Products map[string]*Product, code string) Product {
     300ms      3.29s     40:	for _, item := range Products {
     4.53s        13s     41:		if code == (*item).Code {
         .          .     42:			return *item
         .          .     43:		}
         .          .     44:	}
         .          .     45:
         .          .     46:	return Product{}
(pprof)
```
