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


## Muilti stage build

```sh
cd ~/multistage-dockerfile

docker build -t multi-stage -f Dockerfile .
```

### Lets run our Application
```sh
docker run -d -p 8080:8080 multi-stage
```




# Java_app 
Hello world Java application.


### Devops training 

``` testing ```


##### Testing 
``` Dvelopment ```
