+++ 
draft = false
date = 2021-12-31T13:49:55+01:00
title = "Graceful HTTP Server Shutdown in Go"
description = ""
slug = "golang_graceful_shutdown"
authors = ["Konstantin"]
tags = ["golang"]
categories = ["software-engineering"]
externalLink = ""
series = [""]
+++


Suppose we have a longer running operation in one of our http handler, perhaps a important transaction and if we just quit the application the transaction will be interrupted and some important data might be lost. How can we handle that? The solution is quite simple and well known - *graceful shutdown* and i am going to show you how to implement it in go.

Let's get started. At frist let's implement a simple http server which responses with a _bar_ if we are requesting a _/foo_

```go
http.HandleFunc("/foo", func(w http.ResponseWriter, r *http.Request) {
		log.Printf("begin transaction")
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("bar"))
		time.Sleep(5 * time.Second)
		log.Printf("finish transaction")
})

srv := http.Server{Addr: ":8080"}
```

you see we just block the handler for 5 seconds to simulate a transaction. Now we can start our server in a goroutine since `ListenAndServe` call is blocking.

```go
go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatal(err)
    }
}()
```

Note that we don't consider `http.ErrServerClosed` as an error, it's a default response if the server regularly closes.

Well, the server is running, now we need to get notified if somebody wants to stop our application.

```go
done := make(chan os.Signal, 1)
signal.Notify(done, syscall.SIGINT, syscall.SIGTERM)

<-done
```

After we received a SIGINT or SIGTERM signal we proceed with the actual graceful shutdown

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
    log.Fatal("something went wrong %s", err.Error())
}

log.Printf("done")
```

That's basically it, if we try to close the  application (Ctrl-C) middle in the transaction, it won't be interrupted (at least for 10 seconds, see the context timeout) and we are going to see `finish transaction` console message right before application quits.