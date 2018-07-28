---
language: en
title: Gzip
description: Example of how to implement gzip in Go
---

# Gzip

```go
package main

import (
	"time"
    	"net/http"
    	"io"
    	"compress/gzip"
    	"strings"
)

type GzipMiddleware struct {
	Next http.Handler
}

type gzipResponseWriter struct {
	http.ResponseWriter
	io.Writer
}

type gzipPusherResponseWriter struct {
	gzipResponseWriter
	http.Pusher
}

func (gm *GzipMiddleware) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if gm.Next == nil {
		gm.Next = http.DefaultServeMux
	}

	encodings := r.Header.Get("Accept-Encoding")
	if !strings.Contains(encodings, "gzip") {
		gm.Next.ServeHTTP(w, r)
		return
	}
	w.Header().Add("Content-Encoding", "gzip")
	gzipWriter := gzip.NewWriter(w)
	defer gzipWriter.Close()
	var rw http.ResponseWriter
	if pusher, ok := w.(http.Pusher); ok { // see if original writer implements server push
		rw = gzipPusherResponseWriter{
			gzipResponseWriter: gzipResponseWriter{
				ResponseWriter: w,
				Writer:         gzipWriter,
			},
			Pusher: pusher,
		}
	} else {
		rw = gzipResponseWriter{
			ResponseWriter: w,
			Writer:         gzipWriter,
		}
	}
	gm.Next.ServeHTTP(rw, r)
}

func (grw gzipResponseWriter) Write(data []byte) (int, error) {
	return grw.Writer.Write(data)
}

func Cache(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Vary", "Accept-Encoding")
        w.Header().Set("Cache-Control", "public, max-age=7776000")
        next.ServeHTTP(w, r)
    })
}

func main() {
	// main routes
	http.HandleFunc("/", home)
	http.HandleFunc("/favicon.ico", faviconHandler)

	// static files
	http.Handle("/public/",
		Cache(http.StripPrefix("/public/", http.FileServer(http.Dir("static/public")))))

	httpServer := &http.Server{
		Addr:           "localhost:8080",
		Handler:        new(GzipMiddleware),
		ReadTimeout:    time.Duration(60 * int64(time.Second)),
		WriteTimeout:   time.Duration(60 * int64(time.Second)),
		MaxHeaderBytes: 1 << 20, // 1MB identical as DefaultMaxHeaderBytes
	}

	httpServer.ListenAndServe()
}

func home(w http.ResponseWriter, r *http.Request) {
    // html content
}

func faviconHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Vary", "Accept-Encoding")
    w.Header().Set("Cache-Control", "public, max-age=7776000")
    http.ServeFile(w, r, "server/favicon.ico")
}
```