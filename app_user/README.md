### 问题

容器和宿主机共享同一 `linux`内核。在不使用 `user namespace`时，宿主机 `root uid`和容器 `root uid`一致 `（root:0）`
只用容器作用户管理的话，如果容器渗透到宿主机，容器 `root`等同于宿主机 `root`

### 理想情形

容器内用户，无宿主机 `root`权限，

### 摸索了一套可行方案

核心思路是用 `user namespace`方法使容器 `root uid（root:0）`在宿主机上的暴露非 `root:0`

### 参考

https://docs.docker.com/engine/security/userns-remap/
https://www.jujens.eu/posts/en/2017/Jul/02/docker-userns-remap/

### 具体方法：

宿主机添加 `docker_user`用户及组

```shell
sudo useradd docker_user
```

宿主机添加用户 `user1`，`user2`，并把他们拉进 `sudo`组、`docker_user`组里

```shell
sudo useradd -G docker_user,sudo user1
sudo useradd -G docker_user,sudo user2
```

添加 `user1`、`user2`密码

```shell
sudo chpasswd
```

```
user1:passwdu1
user2:passwdu2
```

添加 `/etc/docker/daemon.json`文件，在里面写入

```shell
sudo vim /etc/docker/daemon.json
```

```
{
  "userns-remap": "docker_user"
}
```

`id user1`或查看 `/etc/passwd`获得 `user1`、`user2`的 `uid`、`primary gid`
将新加入用户的 `uid`、`primary gid`写入 `/etc/subuid`及 `/etc/subgid`

```shell
id user1
id user2
```

```shell
sudo vim /etc/subuid
```

```
user1:1002:1
user2:1003:1
```

```shell
sudo vim /etc/subgid
```

```
user1:1002:1
user2:1003:1
```

重启docker服务

```shell
sudo systemctl restart docker
```

切换user1使用者

```shell
su user1
sudo docker compose build user1 && sudo docker compose up user1
```

### 特点

- 每个实际用户对应一个容器
- 每个容器对应宿主机一个虚拟用户，容器由宿主机每个虚拟用户拉取
- 容器里所有用户（包括 `root:0`），权限不高于运行该容器的宿主机虚拟用户
- 实际用户知道宿主机上自己用户的密码吗？
  实际用户无权限访问宿主机上自己的虚拟用户，仅宿主机管理员知道宿主机虚拟用户密码，实际用户仅拥有自己对应容器的访问权限及自己容器 `root`密码
- 为什么要把宿主机每个虚拟用户拉进 `sudo`组？安全吗？
  宿主机虚拟用户无 `sudo`权限无法运行容器。
  这个 `sudo`权限是安全的，用户即使从容器内 `（root:0）`渗透到宿主机，`uid`也是宿主机虚拟用户 `（user1）`的 `uid`，虽然虚拟用户在 `sudo`组里，但仍需 `root`密码才能执行宿主机 `sudo`命令
- 使用 `user namespace`技术与不使用时，容器渗透到宿主机？
  不使用 `user namespace`时，技术高超的开发人员可以从容器内 `root:0`渗透到宿主机 `root:0`
  使用 `user namespace`时，技术高超的开发人员可以从容器内 `root:0`渗透到宿主机运行该容器的虚拟用户 `uid`，比如 `user1:1001`，拥有 `user1:1001`的权限

### 注意

`/etc/subuid`与 `/etc/subgid`，任何一个文件内部如发生 `id`分配范围重叠将会存在安全隐患

容器首次运行，需宿主机管理员进入容器开启ssh
