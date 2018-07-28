---
language: en
title: SSL Server
description: Example of ssl server in Go
---

# SSL Server

```go
package main

import (
	"time"
	"net/http"

	"myProj/myFramework/server"
	"myProj/myFramework/routes"
	"myProj/myFramework/middlewares"
)

func main() {
	// main routes
	http.HandleFunc("/", routes.Home)
	http.HandleFunc("/favicon.ico", routes.FaviconHandler)

	// static files
	http.Handle("/public/",
		middlewares.Cache(http.StripPrefix("/public/", http.FileServer(http.Dir(server.Config.Static)))))

	httpServer := &http.Server{
		Addr:           server.Config.Addr,
		Handler:        http.HandlerFunc(redirectToHttps),
		ReadTimeout:    time.Duration(60 * int64(time.Second)),
		WriteTimeout:   time.Duration(60 * int64(time.Second)),
		MaxHeaderBytes: 1 << 20, // 1MB identical as DefaultMaxHeaderBytes
	}

	//starting up the server
	httpsServer := &http.Server{
		Addr:           server.Config.AddrHTTPS,
		Handler:        new(middlewares.GzipMiddleware),
		ReadTimeout:    time.Duration(60 * int64(time.Second)),
		WriteTimeout:   time.Duration(60 * int64(time.Second)),
		MaxHeaderBytes: 1 << 20,
	}

	go httpsServer.ListenAndServeTLS("server/fullchain.pem", "server/privkey.pem")
	httpServer.ListenAndServe()
}

func redirectToHttps(w http.ResponseWriter, r *http.Request) {
	// Redirect the incoming HTTP request.
	http.Redirect(w, r, "https://"+server.Config.AddrHTTPS+r.RequestURI, http.StatusMovedPermanently)
}

```