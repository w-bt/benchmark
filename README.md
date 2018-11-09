# Benchmark and Optimizing Code

Sharing Session Benchmark and Optimizing Code on 09 November 2018 - Tokopedia Tower 31st Floor

### Requirements

  - Golang 1.11 or newer
  - Golang x tools (sudo apt install golang-golang-x-tools)
  - Any editor (vscode or atom)
  
# Test Case

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
	rw := httptest.NewRecorder()
	for i := 0; i < b.N; i++ {
		handleProduct(rw, r)
		reset(rw)
	}
}

func reset(rw *httptest.ResponseRecorder) {
	m := rw.HeaderMap
	for k := range m {
		delete(m, k)
	}
	body := rw.Body
	body.Reset()
	*rw = httptest.ResponseRecorder{
		Body:      body,
		HeaderMap: m,
	}
}
```
```sh
$ go test -v -run=^$ -bench=. -benchtime=10s -cpuprofile=prof.cpu -memprofile=prof.mem | tee prof1
goos: linux
goarch: amd64
pkg: github.com/w-bt/benchmark
BenchmarkHandleProduct-4   	    5000	   3188656 ns/op	    4403 B/op	      59 allocs/op
PASS
ok  	github.com/w-bt/benchmark	16.712s
```
This command produces 4 new files:
  - binary test (benchmark.test)
  - cpu profile (prof.cpu)
  - memory profile (prof.mem)
  - benchmark result (prof1)

Based on the data above, benchmarking is done by +-10s. Detailed result:
  - total execution: 5000 times
  - duration each operation: 3188656 ns/op
  - each iteration costs: 4403 bytes with 59 allocations

### CPU Profiling
```sh
$ go tool pprof benchmark.test prof.cpu
File: benchmark.test
Type: cpu
Time: Nov 9, 2018 at 8:34am (WIB)
Duration: 16.52s, Total samples = 16.42s (99.41%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 15.99s, 97.38% of 16.42s total
Dropped 67 nodes (cum <= 0.08s)
Showing top 10 nodes out of 16
      flat  flat%   sum%        cum   cum%
     8.13s 49.51% 49.51%      8.13s 49.51%  memeqbody
     5.09s 31.00% 80.51%     16.07s 97.87%  github.com/w-bt/benchmark.findProduct
     2.61s 15.90% 96.41%      2.72s 16.57%  runtime.mapiternext
     0.13s  0.79% 97.20%      0.13s  0.79%  runtime.memequal
     0.03s  0.18% 97.38%      0.10s  0.61%  runtime.scanobject
         0     0% 97.38%     16.24s 98.90%  github.com/w-bt/benchmark.BenchmarkHandleProduct
         0     0% 97.38%     16.23s 98.84%  github.com/w-bt/benchmark.handleProduct
         0     0% 97.38%      0.11s  0.67%  regexp.Compile
         0     0% 97.38%      0.11s  0.67%  regexp.MatchString
         0     0% 97.38%      0.11s  0.67%  regexp.compile
(pprof) top5
Showing nodes accounting for 15.99s, 97.38% of 16.42s total
Dropped 67 nodes (cum <= 0.08s)
Showing top 5 nodes out of 16
      flat  flat%   sum%        cum   cum%
     8.13s 49.51% 49.51%      8.13s 49.51%  memeqbody
     5.09s 31.00% 80.51%     16.07s 97.87%  github.com/w-bt/benchmark.findProduct
     2.61s 15.90% 96.41%      2.72s 16.57%  runtime.mapiternext
     0.13s  0.79% 97.20%      0.13s  0.79%  runtime.memequal
     0.03s  0.18% 97.38%      0.10s  0.61%  runtime.scanobject
(pprof) top --cum
Showing nodes accounting for 15.96s, 97.20% of 16.42s total
Dropped 67 nodes (cum <= 0.08s)
Showing top 10 nodes out of 16
      flat  flat%   sum%        cum   cum%
         0     0%     0%     16.24s 98.90%  github.com/w-bt/benchmark.BenchmarkHandleProduct
         0     0%     0%     16.24s 98.90%  testing.(*B).runN
         0     0%     0%     16.23s 98.84%  github.com/w-bt/benchmark.handleProduct
         0     0%     0%     16.23s 98.84%  testing.(*B).launch
     5.09s 31.00% 31.00%     16.07s 97.87%  github.com/w-bt/benchmark.findProduct
     8.13s 49.51% 80.51%      8.13s 49.51%  memeqbody
     2.61s 15.90% 96.41%      2.72s 16.57%  runtime.mapiternext
         0     0% 96.41%      0.14s  0.85%  runtime.systemstack
     0.13s  0.79% 97.20%      0.13s  0.79%  runtime.memequal
         0     0% 97.20%      0.11s  0.67%  regexp.Compile
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
Total: 16.42s
ROUTINE ======================== github.com/w-bt/benchmark.handleProduct in /home/nakama/Code/go/src/github.com/w-bt/benchmark/main.go
         0     16.23s (flat, cum) 98.84% of Total
         .          .     18:	log.Fatal(http.ListenAndServe("127.0.0.1:1234", nil))
         .          .     19:}
         .          .     20:
         .          .     21:func handleProduct(w http.ResponseWriter, r *http.Request) {
         .          .     22:	code := r.FormValue("code")
         .      110ms     23:	if match, _ := regexp.MatchString(`^[A-Z]{2}[0-9]{2}$`, code); !match {
         .          .     24:		http.Error(w, "code is invalid", http.StatusBadRequest)
         .          .     25:		return
         .          .     26:	}
         .          .     27:
         .     16.07s     28:	result := findProduct(products, code)
         .          .     29:
         .          .     30:	if result.Code == "" {
         .          .     31:		http.Error(w, "Data Not Found", http.StatusBadRequest)
         .          .     32:		return
         .          .     33:	}
         .          .     34:
         .       20ms     35:	w.Header().Set("Content-Type", "text/html; charset=utf-8")
         .       30ms     36:	w.Write([]byte(`<font size="10">Product Code : ` + result.Code + ` Name :` + result.Name + `</font>`))
         .          .     37:}
         .          .     38:
         .          .     39:func findProduct(Products map[string]*Product, code string) Product {
         .          .     40:	for _, item := range Products {
         .          .     41:		if code == (*item).Code {
(pprof) list findProduct
Total: 16.42s
ROUTINE ======================== github.com/w-bt/benchmark.findProduct in /home/nakama/Code/go/src/github.com/w-bt/benchmark/main.go
     5.09s     16.07s (flat, cum) 97.87% of Total
         .          .     35:	w.Header().Set("Content-Type", "text/html; charset=utf-8")
         .          .     36:	w.Write([]byte(`<font size="10">Product Code : ` + result.Code + ` Name :` + result.Name + `</font>`))
         .          .     37:}
         .          .     38:
         .          .     39:func findProduct(Products map[string]*Product, code string) Product {
     270ms      2.99s     40:	for _, item := range Products {
     4.82s     13.08s     41:		if code == (*item).Code {
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
Time: Nov 9, 2018 at 8:35am (WIB)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top --cum
Showing nodes accounting for 28.70MB, 51.37% of 55.88MB total
Showing top 10 nodes out of 55
      flat  flat%   sum%        cum   cum%
         0     0%     0%    30.86MB 55.22%  runtime.main
   25.70MB 46.00% 46.00%    29.70MB 53.15%  github.com/w-bt/benchmark.GenerateProduct
         0     0% 46.00%    23.50MB 42.06%  github.com/w-bt/benchmark.BenchmarkHandleProduct
    1.50MB  2.68% 48.68%    23.50MB 42.06%  github.com/w-bt/benchmark.handleProduct
         0     0% 48.68%    23.50MB 42.06%  testing.(*B).launch
         0     0% 48.68%    23.50MB 42.06%  testing.(*B).runN
         0     0% 48.68%    20.50MB 36.69%  regexp.MatchString
         0     0% 48.68%       18MB 32.22%  regexp.Compile
    1.50MB  2.68% 51.37%       18MB 32.22%  regexp.compile
         0     0% 51.37%    17.57MB 31.44%  github.com/w-bt/benchmark.TestMain
(pprof) list handleProduct
Total: 55.88MB
ROUTINE ======================== github.com/w-bt/benchmark.handleProduct in /home/nakama/Code/go/src/github.com/w-bt/benchmark/main.go
    1.50MB    23.50MB (flat, cum) 42.06% of Total
         .          .     18:	log.Fatal(http.ListenAndServe("127.0.0.1:1234", nil))
         .          .     19:}
         .          .     20:
         .          .     21:func handleProduct(w http.ResponseWriter, r *http.Request) {
         .          .     22:	code := r.FormValue("code")
         .    20.50MB     23:	if match, _ := regexp.MatchString(`^[A-Z]{2}[0-9]{2}$`, code); !match {
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
         .          .     35:	w.Header().Set("Content-Type", "text/html; charset=utf-8")
    1.50MB        3MB     36:	w.Write([]byte(`<font size="10">Product Code : ` + result.Code + ` Name :` + result.Name + `</font>`))
         .          .     37:}
         .          .     38:
         .          .     39:func findProduct(Products map[string]*Product, code string) Product {
         .          .     40:	for _, item := range Products {
         .          .     41:		if code == (*item).Code {
(pprof) list MatchString
Total: 55.88MB
ROUTINE ======================== regexp.(*Regexp).MatchString in /usr/local/go/src/regexp/regexp.go
         0     2.50MB (flat, cum)  4.47% of Total
         .          .    436:}
         .          .    437:
         .          .    438:// MatchString reports whether the string s
         .          .    439:// contains any match of the regular expression re.
         .          .    440:func (re *Regexp) MatchString(s string) bool {
         .     2.50MB    441:	return re.doMatch(nil, nil, s)
         .          .    442:}
         .          .    443:
         .          .    444:// Match reports whether the byte slice b
         .          .    445:// contains any match of the regular expression re.
         .          .    446:func (re *Regexp) Match(b []byte) bool {
ROUTINE ======================== regexp.MatchString in /usr/local/go/src/regexp/regexp.go
         0    20.50MB (flat, cum) 36.69% of Total
         .          .    460:
         .          .    461:// MatchString reports whether the string s
         .          .    462:// contains any match of the regular expression pattern.
         .          .    463:// More complicated queries need to use Compile and the full Regexp interface.
         .          .    464:func MatchString(pattern string, s string) (matched bool, err error) {
         .       18MB    465:	re, err := Compile(pattern)
         .          .    466:	if err != nil {
         .          .    467:		return false, err
         .          .    468:	}
         .     2.50MB    469:	return re.MatchString(s), nil
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

# Optimization

### Update findProduct()

```golang
func findProduct(Products map[string]*Product, code string) Product {
	if v, ok := Products[code]; ok {
		return *v
	}

	return Product{}
}
```
```sh
$ go test -v -run=^$ -bench=. -benchtime=10s -cpuprofile=prof.cpu -memprofile=prof.mem | tee prof2
goos: linux
goarch: amd64
pkg: github.com/w-bt/benchmark
BenchmarkHandleProduct-4   	 2000000	      8918 ns/op	    4400 B/op	      59 allocs/op
PASS
ok  	github.com/w-bt/benchmark	27.951s
```
```sh
$ benchcmp prof1 prof2
benchmark                    old ns/op     new ns/op     delta
BenchmarkHandleProduct-4     3188656       8918          -99.72%

benchmark                    old allocs     new allocs     delta
BenchmarkHandleProduct-4     59             59             +0.00%

benchmark                    old bytes     new bytes     delta
BenchmarkHandleProduct-4     4403          4400          -0.07%
```

### Update Regex

Compile regex first
```golang
	codeRegex = regexp.MustCompile(`^[A-Z]{2}[0-9]{2}$`)
	// . . .
	if match := codeRegex.MatchString(code); !match {
		// . . .
	}
```

```sh
$ goarch: amd64
pkg: github.com/w-bt/benchmark
BenchmarkHandleProduct-4   	10000000	      1400 ns/op	     560 B/op	       6 allocs/op
PASS
ok  	github.com/w-bt/benchmark	15.815s
```
```sh
$ benchcmp prof2 prof3
benchmark                    old ns/op     new ns/op     delta
BenchmarkHandleProduct-4     8918          1400          -84.30%

benchmark                    old allocs     new allocs     delta
BenchmarkHandleProduct-4     59             6              -89.83%

benchmark                    old bytes     new bytes     delta
BenchmarkHandleProduct-4     4400          560           -87.27%
```
### Update Concate String
```golang
var bufPool = sync.Pool{
	New: func() interface{} {
		return new(bytes.Buffer)
	},
}
// . . .
buf := bufPool.Get().(*bytes.Buffer)
defer bufPool.Put(buf)
buf.Reset()
buf.WriteString(`<font size="10">Product Code : `)
buf.WriteString(result.Code)
buf.WriteString(` Name :`)
buf.WriteString(result.Name)
buf.WriteString(`</font>`)
w.Write(buf.Bytes())
```
```sh
$ go test -v -run=^$ -bench=. -benchtime=10s -cpuprofile=prof.cpu -memprofile=prof.mem | tee prof4
goos: linux
goarch: amd64
pkg: github.com/w-bt/benchmark
BenchmarkHandleProduct-4   	10000000	      1347 ns/op	     432 B/op	       4 allocs/op
PASS
ok  	github.com/w-bt/benchmark	15.212s
```
```sh
$ benchcmp prof3 prof4
benchmark                    old ns/op     new ns/op     delta
BenchmarkHandleProduct-4     1400          1347          -3.79%

benchmark                    old allocs     new allocs     delta
BenchmarkHandleProduct-4     6              4              -33.33%

benchmark                    old bytes     new bytes     delta
BenchmarkHandleProduct-4     560           432           -22.86%
```
