---
language: en
title: Cache files
description: Example of how to cache files
---

# Cache HTML files

```go
package main

import (
	"time"
	"net/http"
)

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

    http.ListenAndServe(":8080", nil)
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

Files like `css` and `js` that lives into `staic/public` folder are cached by the browser.
This files can be used into HTML with `public` prefix.