# Termux 的 Golang 环境
提供一个 `Termux`的`Golang` 环境  

用于构建能运行在 `Termux` 的 `Go` 应用  

暂只支持 `aarch64` 架构

## 说明

[termux-docker Github](https://github.com/termux/termux-docker/) 上说: 在 x86（64） 主机上运行 AArch64 容器，需要通过 binfmt_misc 设置 QEMU 仿真器  

```shell
docker run --rm --privileged aptman/qus -s -- -p aarch64 arm
```

而且 AArch64 和 ARM 容器仅在特权模式下才能正常工作

```shell
docker run -it --privileged termux/termux-docker:aarch64
```

这意味着,把 `termux/termux-docker:aarch64` 当作基础镜像会有问题

具体表现就是因权限问题, dnsmasq启动失败导致网络问题, 不能使用pkg安装环境

其实 `docker run --privileged` 参数, `docker buildx` 是支持的, 文档在这 [buildx/build/#allow](https://docs.docker.com/reference/cli/docker/buildx/build/#allow)
还有这 [dockerfile/#run---security](https://docs.docker.com/reference/dockerfile/#run---security)

经过测试, 实际上安装环境的时候还是没网

Dockerfile
```shell 
# syntax=docker/dockerfile:1-labs
FROM termux/termux-docker:aarch64
USER system
RUN --network=host --security=insecure pkg install -y golang git make

CMD ["/data/data/com.termux/files/usr/bin/login"]

```

构建命令
```shell

docker buildx create --use --name termux-go-env-builder --platform linux/arm64 --buildkitd-flags '--oci-worker-net host --allow-insecure-entitlement security.insecure --allow-insecure-entitlement network.host'

docker buildx build --allow network.host --allow security.insecure --platform linux/arm64 -t termux-docker-golang .

```


因原始镜像的问题(可能), 暂时没有办法通过 `Dockerfile` 的方式构建镜像

最终 只能通过 `run` 的方式生成容器 再 commit 出 镜像 这种方式来构建出 `Termux`的`Golang` 环境的镜像

```shell
docker run --name termux-docker-golang --privileged termux/termux-docker:aarch64 bash -c 'pkg install -y golang git make'
docker commit --change='CMD ["/data/data/com.termux/files/usr/bin/login"]' termux-docker-golang termux-docker-golang:latest
```


## 使用:

以 `trzsz-ssh` 为例子

```shell
# clone 和 构建
docker run --name tmp_container --privileged fa1seut0pia/termux-docker-golang:aarch64 bash -c 'git clone https://github.com/trzsz/trzsz-ssh.git && cd trzsz-ssh && make'

# 手动导出二进制文件
docker cp tmp_container:/data/data/com.termux/files/home/trzsz-ssh/bin/tssh .
```