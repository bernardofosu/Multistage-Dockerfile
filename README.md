## Single stage build

```sh
cd ~/multistage-dockerfile

docker build -t single-stage -f Dockerfile-ubuntu .
```
docker build → builds a new image.
- -t single-stage → -t = tag. It gives the built image a name (single-stage), so you can run it later with docker run single-stage.
- You can also include version or registry, e.g. -t myapp:1.0 or -t myrepo/myapp:latest.
- -f Dockerfile-ubuntu → tells Docker which Dockerfile to use (default is Dockerfile).

. → the build context (current directory). Docker sends this context to the daemon to execute the build.

### Lets run our Application
```sh
docker run --name single-stage -d -p 8080:8080 single-stage

docker ps
```

## Muilti stage build

```sh
cd ~/multistage-dockerfile

docker build -t multi-stage -f Dockerfile .
```

### Lets run our Application
```sh
docker run --name multi-stage -d -p 8080:8080 multi-stage
```

```sh
docker ps -a                          # you'll see the stopped container 0bdcc04c68f4

docker rm 0bdcc04c68f4                # remove the stopped container
docker rmi 09993d826c2e               # now the image can be deleted
# (or force both in one shot: docker rm -f 0bdcc04c68f4 && docker rmi -f 09993d826c2e)

docker container prune      # removes all stopped containers (interactive prompt)
docker image prune          # removes dangling (<none>) images
# or everything not used by any container:
# docker image prune -a
```



# Java_app 
Hello world Java application.


### Devops training 

``` testing ```


##### Testing 
``` Dvelopment ```
