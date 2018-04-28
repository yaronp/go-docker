# go-docker
Dockerfile for Go code that will be compiled in container

* Code has to be compiled in a container, to boost the chances my build will be reproducible.
* Use dep for fetching the dependencies in case the vendor folder is not committed alongside with the code. 

NOTE: if vendor/ is in the .gitignore, it should be in the .dockerignore too.

* The final image should be as small as possible. 
  - Go applications compile to a single binary. 
    We can have images as small as our compiled binary by leveraging the special FROM scratch base image in a multi-stage build.
    
**This will work out of the box. But if you plan to use sub-packages, remember to replace “github.com/username/repo” on line 8 with the actual URI of your repository, in order to provide the compiler with a working import path.**

One tricky part is on line 12: the go build step. What is that gibberish?

* CGO_ENABLED=0 is an environment variable that tells to the compiler to disable the support for linking C code. As a result, the resulting binary will not be able to depend on the C system libraries. The point is that in a scratch container, there are no system libraries. If we omit this directive, the Docker build will terminate successfully, but the resulting container will crash with funny errors.

```standard_init_linux.go:195: exec user process caused “no such file or directory”```

installsuffix is probably not necessary in a Docker build, but it looks like it’s a pretty complicated matter. This option is meant to change the name of the folder used to stock the pkg files, by adding a custom suffix. If I get it, specifying a suffix is useful in order to force the compiler to rebuild everything and not rely on the contents of the current pkg folder. Basically, it has a similar meaning to the -a switch.

On line 14, FROM scratch tells Docker to start over, using a new empty base image. The first stage, FROM golang, will be discarded. The final Docker image will only contain this one, which doesn’t even carry an operating system.

* COPY --from=builder is the switch for selecting one previous stage to copy a file from. At line 1, we have named the first stage with AS builder. We can now use that name to pick up /app and put it in this new empty stage.

Note that in the last line, the arguments to ENTRYPOINT are given as a JSON array: if instead a string had been passed, Docker would have invoked sh as a tokenizer. And… you have guessed it: there is no sh in a scratch container.

```docker: Error response from daemon: OCI runtime create failed: container_linux.go:296: starting container process caused “exec: \”/bin/sh\”: stat /bin/sh: no such file or directory”: unknown.```
