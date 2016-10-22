---
layout: post
title: go net/http Client使用
category: 技术
tags: Go
keywords:
description:
---

go net/http Client使用总结

###Client数据结构

```go

// A Client is an HTTP client. Its zero value (DefaultClient) is a
// usable client that uses DefaultTransport.
//
// The Client's Transport typically has internal state (cached TCP
// connections), so Clients should be reused instead of created as
// needed. Clients are safe for concurrent use by multiple goroutines.
//
// A Client is higher-level than a RoundTripper (such as Transport)
// and additionally handles HTTP details such as cookies and
// redirects.
type Client struct {
	// Transport specifies the mechanism by which individual
	// HTTP requests are made.
	// If nil, DefaultTransport is used.
	Transport RoundTripper

	// CheckRedirect specifies the policy for handling redirects.
	// If CheckRedirect is not nil, the client calls it before
	// following an HTTP redirect. The arguments req and via are
	// the upcoming request and the requests made already, oldest
	// first. If CheckRedirect returns an error, the Client's Get
	// method returns both the previous Response (with its Body
	// closed) and CheckRedirect's error (wrapped in a url.Error)
	// instead of issuing the Request req.
	// As a special case, if CheckRedirect returns ErrUseLastResponse,
	// then the most recent response is returned with its body
	// unclosed, along with a nil error.
	//
	// If CheckRedirect is nil, the Client uses its default policy,
	// which is to stop after 10 consecutive requests.
	CheckRedirect func(req *Request, via []*Request) error

	// Jar specifies the cookie jar.
	// If Jar is nil, cookies are not sent in requests and ignored
	// in responses.
	Jar CookieJar

	// Timeout specifies a time limit for requests made by this
	// Client. The timeout includes connection time, any
	// redirects, and reading the response body. The timer remains
	// running after Get, Head, Post, or Do return and will
	// interrupt reading of the Response.Body.
	//
	// A Timeout of zero means no timeout.
	//
	// The Client cancels requests to the underlying Transport
	// using the Request.Cancel mechanism. Requests passed
	// to Client.Do may still set Request.Cancel; both will
	// cancel the request.
	//
	// For compatibility, the Client will also use the deprecated
	// CancelRequest method on Transport if found. New
	// RoundTripper implementations should use Request.Cancel
	// instead of implementing CancelRequest.
	Timeout time.Duration
}

```

注意`Transport`数据结构，对应如下：

```go
// Transport is an implementation of RoundTripper that supports HTTP,
// HTTPS, and HTTP proxies (for either HTTP or HTTPS with CONNECT).
//
// By default, Transport caches connections for future re-use.
// This may leave many open connections when accessing many hosts.
// This behavior can be managed using Transport's CloseIdleConnections method
// and the MaxIdleConnsPerHost and DisableKeepAlives fields.
//
// Transports should be reused instead of created as needed.
// Transports are safe for concurrent use by multiple goroutines.
//
// A Transport is a low-level primitive for making HTTP and HTTPS requests.
// For high-level functionality, such as cookies and redirects, see Client.
//
// Transport uses HTTP/1.1 for HTTP URLs and either HTTP/1.1 or HTTP/2
// for HTTPS URLs, depending on whether the server supports HTTP/2.
// See the package docs for more about HTTP/2.
type Transport struct {
	idleMu     sync.Mutex
	wantIdle   bool                                // user has requested to close all idle conns
	idleConn   map[connectMethodKey][]*persistConn // most recently used at end
	idleConnCh map[connectMethodKey]chan *persistConn
	idleLRU    connLRU

	reqMu       sync.Mutex
	reqCanceler map[*Request]func()

	altMu    sync.RWMutex
	altProto map[string]RoundTripper // nil or map of URI scheme => RoundTripper

	// Proxy specifies a function to return a proxy for a given
	// Request. If the function returns a non-nil error, the
	// request is aborted with the provided error.
	// If Proxy is nil or returns a nil *URL, no proxy is used.
	Proxy func(*Request) (*url.URL, error)

	// DialContext specifies the dial function for creating unencrypted TCP connections.
	// If DialContext is nil (and the deprecated Dial below is also nil),
	// then the transport dials using package net.
	DialContext func(ctx context.Context, network, addr string) (net.Conn, error)

	// Dial specifies the dial function for creating unencrypted TCP connections.
	//
	// Deprecated: Use DialContext instead, which allows the transport
	// to cancel dials as soon as they are no longer needed.
	// If both are set, DialContext takes priority.
	Dial func(network, addr string) (net.Conn, error)

	// DialTLS specifies an optional dial function for creating
	// TLS connections for non-proxied HTTPS requests.
	//
	// If DialTLS is nil, Dial and TLSClientConfig are used.
	//
	// If DialTLS is set, the Dial hook is not used for HTTPS
	// requests and the TLSClientConfig and TLSHandshakeTimeout
	// are ignored. The returned net.Conn is assumed to already be
	// past the TLS handshake.
	DialTLS func(network, addr string) (net.Conn, error)

	// TLSClientConfig specifies the TLS configuration to use with
	// tls.Client. If nil, the default configuration is used.
	TLSClientConfig *tls.Config

	// TLSHandshakeTimeout specifies the maximum amount of time waiting to
	// wait for a TLS handshake. Zero means no timeout.
	TLSHandshakeTimeout time.Duration

	// DisableKeepAlives, if true, prevents re-use of TCP connections
	// between different HTTP requests.
	DisableKeepAlives bool

	// DisableCompression, if true, prevents the Transport from
	// requesting compression with an "Accept-Encoding: gzip"
	// request header when the Request contains no existing
	// Accept-Encoding value. If the Transport requests gzip on
	// its own and gets a gzipped response, it's transparently
	// decoded in the Response.Body. However, if the user
	// explicitly requested gzip it is not automatically
	// uncompressed.
	DisableCompression bool

	// MaxIdleConns controls the maximum number of idle (keep-alive)
	// connections across all hosts. Zero means no limit.
	MaxIdleConns int

	// MaxIdleConnsPerHost, if non-zero, controls the maximum idle
	// (keep-alive) connections to keep per-host. If zero,
	// DefaultMaxIdleConnsPerHost is used.
	MaxIdleConnsPerHost int

	// IdleConnTimeout is the maximum amount of time an idle
	// (keep-alive) connection will remain idle before closing
	// itself.
	// Zero means no limit.
	IdleConnTimeout time.Duration

	// ResponseHeaderTimeout, if non-zero, specifies the amount of
	// time to wait for a server's response headers after fully
	// writing the request (including its body, if any). This
	// time does not include the time to read the response body.
	ResponseHeaderTimeout time.Duration

	// ExpectContinueTimeout, if non-zero, specifies the amount of
	// time to wait for a server's first response headers after fully
	// writing the request headers if the request has an
	// "Expect: 100-continue" header. Zero means no timeout.
	// This time does not include the time to send the request header.
	ExpectContinueTimeout time.Duration

	// TLSNextProto specifies how the Transport switches to an
	// alternate protocol (such as HTTP/2) after a TLS NPN/ALPN
	// protocol negotiation. If Transport dials an TLS connection
	// with a non-empty protocol name and TLSNextProto contains a
	// map entry for that key (such as "h2"), then the func is
	// called with the request's authority (such as "example.com"
	// or "example.com:1234") and the TLS connection. The function
	// must return a RoundTripper that then handles the request.
	// If TLSNextProto is nil, HTTP/2 support is enabled automatically.
	TLSNextProto map[string]func(authority string, c *tls.Conn) RoundTripper

	// MaxResponseHeaderBytes specifies a limit on how many
	// response bytes are allowed in the server's response
	// header.
	//
	// Zero means to use a default limit.
	MaxResponseHeaderBytes int64

	// nextProtoOnce guards initialization of TLSNextProto and
	// h2transport (via onceSetNextProtoDefaults)
	nextProtoOnce sync.Once
	h2transport   *http2Transport // non-nil if http2 wired up

	// TODO: tunable on max per-host TCP dials in flight (Issue 13957)
}

```
实际上是一个**连接池**，所以在使用的过程中通常是初始化**一个**`Transport`和`Client`，在**每个**处理函数中都共用该连接池，例如如下Demo：

```go
package main

import (
    "bytes"
    "io/ioutil"
    "log"
	"net"
    "net/http"
    "time"
)

var (
    httpClient *http.Client
)

// init HTTPClient
func init() {
    httpClient = createHTTPClient()
}

const (
	MaxIdleConns int = 100
	MaxIdleConnsPerHost int = 100
	IdleConnTimeout int = 90
)

// createHTTPClient for connection re-use
func createHTTPClient() *http.Client {
    client := &http.Client{
        Transport: &http.Transport{
			Proxy: http.ProxyFromEnvironment,
			DialContext: (&net.Dialer{
        		Timeout:   30 * time.Second,
        		KeepAlive: 30 * time.Second,
      		}).DialContext,
			MaxIdleConns:        MaxIdleConns,
			MaxIdleConnsPerHost: MaxIdleConnsPerHost,
            IdleConnTimeout:	 IdleConnTimeout * time.Second,
        },
    }
    return client
}

func main() {
    var endPoint string = "https://localhost:8080/doSomething"

    req, err := http.NewRequest("POST", endPoint, bytes.NewBuffer([]byte("Post this data")))
    if err != nil {
        log.Fatalf("Error Occured. %+v", err)
    }
    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")

    // use httpClient to send request
    response, err := httpClient.Do(req)
    if err != nil && response == nil {
        log.Fatalf("Error sending request to API endpoint. %+v", err)
    } else {
        // Close the connection to reuse it
        defer response.Body.Close()

        // Let's check if the work actually is done
        // We have seen inconsistencies even when we get 200 OK response
        body, err := ioutil.ReadAll(response.Body)
        if err != nil {
            log.Fatalf("Couldn't parse response body. %+v", err)
        }

        log.Println("Response Body:", string(body))
    }

}

```

下面详细讲解`Transport`结构需要注意的几个参数设置：
* DisableKeepAlives
> // DisableKeepAlives, if true, prevents re-use of TCP connections
// between different HTTP requests.

表示是否开启http keepalive功能，也即是否重用连接，默认开启(false)。
* MaxIdleConns
>// MaxIdleConns controls the maximum number of idle (keep-alive)
	// connections across all hosts. Zero means no limit.

表示连接池对所有host的最大链接数量，host也即dest-ip，默认为无穷大（0），但是通常情况下为了性能考虑都要严格限制该数目（<font color="#8B0000">实际使用中通常利用压测 二分得到该参数的最佳近似值</font>）。
太大容易导致客户端和服务端的socket数量剧增，导致内存吃满，文件描述符不足等问题；太小则限制了连接池的socket数量，资源利用率较低。
* MaxIdleConnsPerHost
>// MaxIdleConnsPerHost, if non-zero, controls the maximum idle
	// (keep-alive) connections to keep per-host. If zero,
	// DefaultMaxIdleConnsPerHost is used.
	
表示连接池对每个host的最大链接数量，从字面意思也可以看出：
```
MaxIdleConnsPerHost <= MaxIdleConns
```
如果客户端只需要访问一个host，那么最好将`MaxIdleConnsPerHost`与`MaxIdleConns`设置为相同，这样逻辑更加清晰。
* IdleConnTimeout
>// IdleConnTimeout is the maximum amount of time an idle
	// (keep-alive) connection will remain idle before closing
	// itself.
	// Zero means no limit.
	
空闲timeout设置，也即socket在该时间内没有交互则自动关闭连接（注意：**该timeout起点是从每次空闲开始计时，若有交互则重置为0**）,该参数通常设置为分钟级别，例如：90秒。
* DialContext
>// DialContext specifies the dial function for creating unencrypted TCP connections.
	// If DialContext is nil (and the deprecated Dial below is also nil),
	// then the transport dials using package net.
	DialContext func(ctx context.Context, network, addr string) (net.Conn, error)
	
该函数用于创建http（非https）连接，通常需要关注`Timeout`和`KeepAlive`参数。前者表示建立Tcp链接超时时间；后者表示底层为了维持http keepalive状态 每隔多长时间发送Keep-Alive报文。`Timeout`通常设置为30s（网络环境良好），`KeepAlive`通常设置为30s(与IdleConnTimeout要对应)。


参考：
https://github.com/golang/go/issues/13957
https://golang.org/src/net/http/transport.go#L46
http://stackoverflow.com/questions/17948827/reusing-http-connections-in-golang
https://golang.org/pkg/net/http/#pkg-index 

