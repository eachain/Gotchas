# 常用代码或技巧

## 1. 检查http请求，常用于tcp代理程序
```go
package main

import (
	"bufio"
	"bytes"
	"io"
	"io/ioutil"
	"net"
	"net/http"
	"os"
)

/*
Common stream module:
	input => output

DumpAndCheckHTTPRequest stream module:
	input => tee             output
				\--> buffer -->/
				 \--> check
*/
func DumpAndCheckHTTPRequest(w io.Writer, r io.Reader, valid func(*http.Request) bool) error {
	buf := &bytes.Buffer{}
	br := bufio.NewReader(io.TeeReader(r, buf))
	lr := &io.LimitedReader{R: buf}

	for {
		req, err := http.ReadRequest(br)
		if err != nil {
			return err
		}

		dst := w
		if !valid(req) {
			dst = ioutil.Discard
		}

		lr.N = int64(buf.Len() - br.Buffered())
		_, err = io.Copy(dst, lr)
		if err != nil {
			return err
		}
	}

	return nil
}

func main() {
	c1, c2 := net.Pipe()
	go DumpAndCheckHTTPRequest(os.Stdout, c2, func(req *http.Request) bool {
		return req.Method == http.MethodPost
	})

	req, _ := http.NewRequest(http.MethodPost,
		"http://www.baidu.com",
		bytes.NewBufferString("Hello world!\n"))
	req.Write(c1)
}
```

