# Go In Docker
Demonstration of Go API in docker.

## Simple API
Lets first write a simple api in go so that we can run that in docker container.
[main.go](https://github.com/shoeb240/Go-In-Docker/blob/master/main.go)
```cgo
package main

import (
    "fmt"
    "github.com/gorilla/mux"
    "log"
    "net/http"
)

func main(){
    fmt.Println("Waiting for request to serve...")

    myRouter := mux.NewRouter()
    myRouter.HandleFunc("/users", getUsers).Methods("GET")

    log.Fatal(http.ListenAndServe(":8081", myRouter))
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "I am responding to your API call")
}

```


This API will listen to at port 8081. So after successfully running this app if we browse localhost:8081/users, it will show us the response
```cgo
I am responding to your API call
```

To check how to run go api check [this](https://github.com/shoeb240/first-go-api)

## Dockerfile
We will create a dockerfile using base image golang:alpine. 
```dockerfile
FROM golang:alpine AS builder
ADD . /go/src/
RUN apk add git
RUN go get github.com/gorilla/mux
WORKDIR /go/src
RUN go build -o main .
CMD ["./main"]
```

Line 2 copies everything from current directory to $GOPATH of the container.

Line 3 installs git in the container. We will need git to import Gorilla Mux from git repo.

Line 4 downloads Gorilla Mux package using go get command

Line 5 selects containers /go/src as working directory.

Line 6 compiles the application and creates executable file at the same directory.

Line 7 runs the application which means it will be waiting for any API request. It shows:


Lets build image from this dockerfile
```text
$ docker build -t go_in_docker:1.0 .
Sending build context to Docker daemon  54.78kB
Step 1/8 : FROM golang:alpine AS builder
 ---> f56365ec0638
Step 2/8 : ADD . /go/src/
 ---> 2ea27dbbd7b0
Step 3/8 : RUN apk add git
 ---> Running in a746074d58b1
fetch http://dl-cdn.alpinelinux.org/alpine/v3.8/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.8/community/x86_64/APKINDEX.tar.gz
(1/6) Installing nghttp2-libs (1.32.0-r0)
(2/6) Installing libssh2 (1.8.0-r3)
(3/6) Installing libcurl (7.61.1-r1)
(4/6) Installing expat (2.2.5-r0)
(5/6) Installing pcre2 (10.31-r0)
(6/6) Installing git (2.18.1-r0)
Executing busybox-1.28.4-r2.trigger
OK: 19 MiB in 20 packages
Removing intermediate container a746074d58b1
 ---> f4dbc055eada
Step 4/8 : RUN git config --global core.autocrlf false
 ---> Running in 546f339b3bdb
Removing intermediate container 546f339b3bdb
 ---> d770ce96b729
Step 5/8 : RUN go get github.com/gorilla/mux
 ---> Running in 072d6316559b
Removing intermediate container 072d6316559b
 ---> 0eefad1c83d4
Step 6/8 : WORKDIR /go/src
 ---> Running in 7b2e235837b7
Removing intermediate container 7b2e235837b7
 ---> 2c890a083b58
Step 7/8 : RUN go build -o main .
 ---> Running in f6b13ec4524f
Removing intermediate container f6b13ec4524f
 ---> f654545d9c48
Step 8/8 : CMD ["./main"]
 ---> Running in 8121b3a6da0e
Removing intermediate container 8121b3a6da0e
 ---> a433b4463ccf
Successfully built a433b4463ccf
Successfully tagged go_in_docker:1.0
```

Check docker images
```text
Shoeb-Mac:go-in-docker shoeb240$ docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
go_in_docker                               1.0                 a433b4463ccf        2 minutes ago       334MB
```

We are ready to create docker container from this new image. 
```text
$ docker run -p 8081:8081 go_in_docker:1.0
Waiting for request to serve...
```

So our api is running and waiting for request to serve.

Lets use curl or browser to cal the restful API
```text
$ curl localhost:8081/users
I am responding to your API call
```

Our app is running in docker container perfectly, but actually we do not need golang:alpine image anymore, which is heavy weight, after successful compilation and creation of executable file. So we can add another layer of docker build to throw what is unnecessary. We modify our dockefile:
[Dockerfile](https://github.com/shoeb240/Go-In-Docker/blob/master/Dockerfile)
```dockerfile
FROM golang:alpine AS builder
ADD . /go/src/
RUN apk add git
RUN git config --global core.autocrlf false
RUN go get github.com/gorilla/mux
WORKDIR /go/src
RUN go build -o main .

FROM alpine
COPY --from=builder /go/src/main /app/
WORKDIR /app
CMD ["./main"]
```

Our final image is built from alpine base image. Executable file is copied to new container. After we build image we see the image size is dropped.

```text
$ docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
go-in-docker_go                            latest              0573c97084f3        23 seconds ago      11.3MB
<none>                                     <none>              8aea8ef57d99        24 seconds ago      334MB
```

Our previous image `go_in_docker:1.0` was 334MB, now it is 11.3MB.

## Docker compose
I personally prefer docker-compose to run even single service docker container because you can create and remove docker container in a single command.

[docker-compose.yaml](https://github.com/shoeb240/Go-In-Docker/blob/master/docker-compose.yaml)
```text
go:
  container_name: DOCKER_GO_API
  build: .
  restart: always
  ports:
    - "8081:8081"
``` 

Run docker-compose
```text
$ docker-compose up --build
Building go
Step 1/8 : FROM golang:alpine AS builder
 ---> f56365ec0638
Step 2/8 : ADD . /go/src/
 ---> fb33d32c763f
Step 3/8 : RUN apk add git
 ---> Running in 939ef782a425
fetch http://dl-cdn.alpinelinux.org/alpine/v3.8/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.8/community/x86_64/APKINDEX.tar.gz
(1/6) Installing nghttp2-libs (1.32.0-r0)
(2/6) Installing libssh2 (1.8.0-r3)
(3/6) Installing libcurl (7.61.1-r1)
(4/6) Installing expat (2.2.5-r0)
(5/6) Installing pcre2 (10.31-r0)
(6/6) Installing git (2.18.1-r0)
Executing busybox-1.28.4-r2.trigger
OK: 19 MiB in 20 packages
Removing intermediate container 939ef782a425
 ---> 457d683225ab
Step 4/8 : RUN git config --global core.autocrlf false
 ---> Running in 581fa316bee4
Removing intermediate container 581fa316bee4
 ---> abe189cbd9c8
Step 5/8 : RUN go get github.com/gorilla/mux
 ---> Running in 4cdb1643293d
Removing intermediate container 4cdb1643293d
 ---> 42c386ba4d49
Step 6/8 : WORKDIR /go/src
 ---> Running in 5363fba4a4eb
Removing intermediate container 5363fba4a4eb
 ---> 9431e87b1451
Step 7/8 : RUN go build -o main .
 ---> Running in 0e3e83dec129
Removing intermediate container 0e3e83dec129
 ---> 854f94a6c38e
Step 8/8 : CMD ["./main"]
 ---> Running in d80e2fb0db4e
Removing intermediate container d80e2fb0db4e
 ---> 5444452a7077
Successfully built 5444452a7077
Successfully tagged go-in-docker_go:latest
Creating DOCKER_GO_API ... done
Attaching to DOCKER_GO_API
DOCKER_GO_API | Waiting for request to serve...
```
