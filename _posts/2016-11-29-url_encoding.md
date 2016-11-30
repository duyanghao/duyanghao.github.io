---
layout: post
title: url encoding
date: 2016-11-29 18:10:31
category: 技术
tags: Linux Network
excerpt: golang url encoding……
---

## HTML URL Encoding Reference

URL encoding converts characters into a format that can be transmitted over the Internet.

## URL - Uniform Resource Locator

Web browsers request pages from web servers by using a URL.

The URL is the address of a web page, like: http://www.baidu.com.

## URL Encoding (Percent Encoding)

URLs can only be sent over the Internet using the [ASCII character-set.](http://www.w3schools.com/charsets/ref_html_ascii.asp)

Since URLs often contain characters outside the ASCII set, the URL has to be converted into a valid ASCII format.

URL encoding replaces unsafe ASCII characters with a "%" followed by two hexadecimal digits.

URLs cannot contain spaces. URL encoding normally replaces a space with a plus (+) sign or with %20.

## Golang url encoding

```go
package main

import (
    "fmt"
    "net/url"
)

func main() {

    var Url *url.URL
    Url, err := url.Parse("http://www.example.com")
    if err != nil {
        panic("boom")
    }

    Url.Path += "/some/path/or/other_with_funny_characters?_or_not/"
    parameters := url.Values{}
    parameters.Add("hello", "42")
    parameters.Add("hello", "54")
    parameters.Add("vegetable", "potato")
    Url.RawQuery = parameters.Encode()

    fmt.Printf("Encoded URL is %q\n", Url.String())
}
```

```sh
Encoded URL is "http://www.example.com/some/path/or/other_with_funny_characters%3F_or_not/?vegetable=potato&hello=42&hello=54"
```

## Refs

* [URL编码](http://www.ruanyifeng.com/blog/2010/02/url_encoding.html)

* [Encode / decode URLs](http://stackoverflow.com/questions/13820280/encode-decode-urls)

* [HTML URL Encoding Reference](http://www.w3schools.com/tags/ref_urlencode.asp)


