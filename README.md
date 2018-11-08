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
	r, _ := http.ReadRequest(bufio.NewReader(strings.NewReader("GET /product?code=ZZ99 HTTP/1.0\r\n\r\n")))
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
BenchmarkHandleProduct-4   	    5000	   3177473 ns/op	    4977 B/op	      63 allocs/op
PASS
ok  	github.com/w-bt/benchmark	16.610s
```
This command produces 3 new files:
  - binary test (benchmark.test)
  - cpu profile (prof.cpu)
  - memory profile (prof.mem)

Based on the data above, benchmarking is done by +-10s. Detailed result:
  - total execution: 5000 times
  - duration each operation: 3177473 ns/op
  - each iteration costs: 4977 bytes with 63 allocations

### CPU Profiling
```sh
$ go tool pprof benchmark.test prof.cpu
File: benchmark.test
Type: cpu
Time: Nov 9, 2018 at 3:10am (WIB)
Duration: 16.42s, Total samples = 16.40s (99.88%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 15.96s, 97.32% of 16.40s total
Dropped 74 nodes (cum <= 0.08s)
Showing top 10 nodes out of 16
      flat  flat%   sum%        cum   cum%
     8.01s 48.84% 48.84%      8.01s 48.84%  memeqbody
     4.74s 28.90% 77.74%     16.10s 98.17%  github.com/w-bt/benchmark.findProduct
     3.03s 18.48% 96.22%      3.20s 19.51%  runtime.mapiternext
     0.15s  0.91% 97.13%      0.15s  0.91%  runtime.memequal
     0.03s  0.18% 97.32%      0.12s  0.73%  runtime.scanobject
         0     0% 97.32%     16.23s 98.96%  github.com/w-bt/benchmark.BenchmarkHandleProduct
         0     0% 97.32%     16.22s 98.90%  github.com/w-bt/benchmark.handleProduct
         0     0% 97.32%      0.09s  0.55%  regexp.Compile
         0     0% 97.32%      0.10s  0.61%  regexp.MatchString
         0     0% 97.32%      0.09s  0.55%  regexp.compile
(pprof) top5
Showing nodes accounting for 15.96s, 97.32% of 16.40s total
Dropped 74 nodes (cum <= 0.08s)
Showing top 5 nodes out of 16
      flat  flat%   sum%        cum   cum%
     8.01s 48.84% 48.84%      8.01s 48.84%  memeqbody
     4.74s 28.90% 77.74%     16.10s 98.17%  github.com/w-bt/benchmark.findProduct
     3.03s 18.48% 96.22%      3.20s 19.51%  runtime.mapiternext
     0.15s  0.91% 97.13%      0.15s  0.91%  runtime.memequal
     0.03s  0.18% 97.32%      0.12s  0.73%  runtime.scanobject
(pprof) top --cum
Showing nodes accounting for 15.93s, 97.13% of 16.40s total
Dropped 74 nodes (cum <= 0.08s)
Showing top 10 nodes out of 16
      flat  flat%   sum%        cum   cum%
         0     0%     0%     16.23s 98.96%  github.com/w-bt/benchmark.BenchmarkHandleProduct
         0     0%     0%     16.23s 98.96%  testing.(*B).launch
         0     0%     0%     16.23s 98.96%  testing.(*B).runN
         0     0%     0%     16.22s 98.90%  github.com/w-bt/benchmark.handleProduct
     4.74s 28.90% 28.90%     16.10s 98.17%  github.com/w-bt/benchmark.findProduct
     8.01s 48.84% 77.74%      8.01s 48.84%  memeqbody
     3.03s 18.48% 96.22%      3.20s 19.51%  runtime.mapiternext
         0     0% 96.22%      0.16s  0.98%  runtime.systemstack
     0.15s  0.91% 97.13%      0.15s  0.91%  runtime.memequal
         0     0% 97.13%      0.12s  0.73%  runtime.gcBgMarkWorker
(pprof)
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
Total: 16.40s
ROUTINE ======================== github.com/w-bt/benchmark.handleProduct in /home/nakama/Code/go/src/github.com/w-bt/benchmark/main.go
         0     16.22s (flat, cum) 98.90% of Total
         .          .     18:	log.Fatal(http.ListenAndServe("127.0.0.1:1234", nil))
         .          .     19:}
         .          .     20:
         .          .     21:func handleProduct(w http.ResponseWriter, r *http.Request) {
         .          .     22:	code := r.FormValue("code")
         .      100ms     23:	if match, _ := regexp.MatchString(`^[A-Z]{2}[0-9]{2}$`, code); !match {
         .          .     24:		http.Error(w, "code is invalid", http.StatusBadRequest)
         .          .     25:		return
         .          .     26:	}
         .          .     27:
         .     16.10s     28:	result := findProduct(products, code)
         .          .     29:
         .          .     30:	if result.Code == "" {
         .          .     31:		http.Error(w, "Data Not Found", http.StatusBadRequest)
         .          .     32:		return
         .          .     33:	}
         .          .     34:
         .       10ms     35:	w.Header().Set("Content-Type", "text/html; charset=utf-8")
         .       10ms     36:	w.Write([]byte(`<font size="10">Product Code : ` + result.Code + ` Name :` + result.Name + `</font>`))
         .          .     37:}
         .          .     38:
         .          .     39:func findProduct(Products map[string]*Product, code string) Product {
         .          .     40:	for _, item := range Products {
         .          .     41:		if code == (*item).Code {
(pprof) list findProduct
Total: 16.40s
ROUTINE ======================== github.com/w-bt/benchmark.findProduct in /home/nakama/Code/go/src/github.com/w-bt/benchmark/main.go
     4.74s     16.10s (flat, cum) 98.17% of Total
         .          .     35:	w.Header().Set("Content-Type", "text/html; charset=utf-8")
         .          .     36:	w.Write([]byte(`<font size="10">Product Code : ` + result.Code + ` Name :` + result.Name + `</font>`))
         .          .     37:}
         .          .     38:
         .          .     39:func findProduct(Products map[string]*Product, code string) Product {
     230ms      3.43s     40:	for _, item := range Products {
     4.51s     12.67s     41:		if code == (*item).Code {
         .          .     42:			return *item
         .          .     43:		}
         .          .     44:	}
         .          .     45:
         .          .     46:	return Product{}
(pprof)
```

To see in UI form, use `web`

![cpu profile](./pprof003.svg)

### Memory Profiling

Similar with CPU Profiling, execute this command
```sh
go tool pprof --alloc_space benchmark.test prof.mem
File: benchmark.test
Type: alloc_space
Time: Nov 9, 2018 at 3:11am (WIB)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top --cum
Showing nodes accounting for 27.39MB, 48.86% of 56.07MB total
Showing top 10 nodes out of 61
      flat  flat%   sum%        cum   cum%
         0     0%     0%    32.61MB 58.17%  runtime.main
   26.39MB 47.07% 47.07%    30.89MB 55.10%  github.com/w-bt/benchmark.GenerateProduct
         0     0% 47.07%    21.50MB 38.35%  github.com/w-bt/benchmark.BenchmarkHandleProduct
         0     0% 47.07%    21.50MB 38.35%  testing.(*B).launch
         0     0% 47.07%    21.50MB 38.35%  testing.(*B).runN
       1MB  1.78% 48.86%    20.50MB 36.57%  github.com/w-bt/benchmark.handleProduct
         0     0% 48.86%    19.76MB 35.25%  github.com/w-bt/benchmark.TestMain
         0     0% 48.86%    19.76MB 35.25%  main.main
         0     0% 48.86%       16MB 28.54%  regexp.MatchString
         0     0% 48.86%    14.50MB 25.86%  regexp.Compile
(pprof) list handleProduct
Total: 56.07MB
ROUTINE ======================== github.com/w-bt/benchmark.handleProduct in /home/nakama/Code/go/src/github.com/w-bt/benchmark/main.go
       1MB    20.50MB (flat, cum) 36.57% of Total
         .          .     18:	log.Fatal(http.ListenAndServe("127.0.0.1:1234", nil))
         .          .     19:}
         .          .     20:
         .          .     21:func handleProduct(w http.ResponseWriter, r *http.Request) {
         .          .     22:	code := r.FormValue("code")
         .       16MB     23:	if match, _ := regexp.MatchString(`^[A-Z]{2}[0-9]{2}$`, code); !match {
         .          .     24:		http.Error(w, "code is invalid", http.StatusBadRequest)
         .          .     25:		return
         .          .     26:	}
         .          .     27:
         .          .     28:	result := findProduct(products, code)
         .          .     29:
         .          .     30:	if result.Code == "" {
         .          .     31:		http.Error(w, "Data Not Found", http.StatusBadRequest)
         .          .     32:		return
         .          .     33:	}
         .          .     34:
         .        2MB     35:	w.Header().Set("Content-Type", "text/html; charset=utf-8")
       1MB     2.50MB     36:	w.Write([]byte(`<font size="10">Product Code : ` + result.Code + ` Name :` + result.Name + `</font>`))
         .          .     37:}
         .          .     38:
         .          .     39:func findProduct(Products map[string]*Product, code string) Product {
         .          .     40:	for _, item := range Products {
         .          .     41:		if code == (*item).Code {
(pprof) list MatchString
Total: 56.07MB
ROUTINE ======================== regexp.(*Regexp).MatchString in /usr/local/go/src/regexp/regexp.go
         0     1.50MB (flat, cum)  2.68% of Total
         .          .    436:}
         .          .    437:
         .          .    438:// MatchString reports whether the string s
         .          .    439:// contains any match of the regular expression re.
         .          .    440:func (re *Regexp) MatchString(s string) bool {
         .     1.50MB    441:	return re.doMatch(nil, nil, s)
         .          .    442:}
         .          .    443:
         .          .    444:// Match reports whether the byte slice b
         .          .    445:// contains any match of the regular expression re.
         .          .    446:func (re *Regexp) Match(b []byte) bool {
ROUTINE ======================== regexp.MatchString in /usr/local/go/src/regexp/regexp.go
         0       16MB (flat, cum) 28.54% of Total
         .          .    460:
         .          .    461:// MatchString reports whether the string s
         .          .    462:// contains any match of the regular expression pattern.
         .          .    463:// More complicated queries need to use Compile and the full Regexp interface.
         .          .    464:func MatchString(pattern string, s string) (matched bool, err error) {
         .    14.50MB    465:	re, err := Compile(pattern)
         .          .    466:	if err != nil {
         .          .    467:		return false, err
         .          .    468:	}
         .     1.50MB    469:	return re.MatchString(s), nil
         .          .    470:}
         .          .    471:
         .          .    472:// MatchString reports whether the byte slice b
         .          .    473:// contains any match of the regular expression pattern.
         .          .    474:// More complicated queries need to use Compile and the full Regexp interface.
(pprof) web handleProduct
```
![mem profile](./pprof004.svg)

### Another Way

Use Web Version!!!! Open `localhost:8081`

```sh
$ go tool pprof -http=":8081" [binary] [profile]
```
