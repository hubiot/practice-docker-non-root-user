<h1>docker non root user</h1>
コンテナのユーザーとホストのユーザーを合わせる。

[Visual Studio Code リモート開発を使用して開発コンテナを作成する](https://code.visualstudio.com/docs/devcontainers/create-dev-container#_extend-your-docker-compose-file-for-development)

[Docker や VSCode \+ Remote\-Container のパーミッション問題に立ち向かう](https://zenn.dev/forrep/articles/8c0304ad420c8e)

[Docker コンテナ内での作業ユーザー作成方法](https://zenn.dev/aidemy/articles/3b34214b9aabf4)

[devcontainer でホスト側のファイル権限が root になる問題の対処 \- ツー](https://celeron1ghz.hatenablog.com/entry/2024/02/25/110807)

- [フォルダ構成](#フォルダ構成)
- [ホストとコンテナのユーザー ID を調べる](#ホストとコンテナのユーザー-id-を調べる)
- [Dockerfie](#dockerfie)
- [docker-compose.yml](#docker-composeyml)
- [vscode よりコンテナ起動](#vscode-よりコンテナ起動)

# フォルダ構成

```
/home/atoru/workplace/prg/test2
|--.devcontainer
|  |--devcontainer.json
|  |--docker-compose.yml
|  |--web
|  |  |--Dockerfile
```

# ホストとコンテナのユーザー ID を調べる

ホストの id を調べる

```
web $ id
uid=1000(atoru) gid=1000(atoru) groups=1000(atoru),4(adm),20(dialout),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),117(netdev),1001(docker)
web $
```

uid=1000 gid=1000 であることがわかる。

コンテナを作成して、user id ,user group を調べる。

Dockerife

```
FROM node:lts-buster
RUN apt-get update
```

```
root@d1c4fe09e874:/etc# cat passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
::::::::
node:x:1000:1000::/home/node:/bin/bash
```

node:lts-buster には、すでに user id:1000 group id:1000 のユーザーが作られている。

サンプルには、groupadd -g 1000 app_user と user id = 1000 のユーザーを追加する例が多いが、node:lts-buster ではすでに、node ユーザーが id:1000 を使っているので、新たなユーザーを追加せず、node ユーザー の id:1000 を使用する。

# Dockerfie

Dockerfile

```
FROM node:lts-buster

RUN apt-get update
#    apt-get install vim -y && \
#    groupadd -g 1000 app_user && \
#    useradd -m -s /bin/bash -u 1000 -g 1000 app_user
#RUN mkdir -p /opt/app && chown -R app_user:app_user /opt/app
RUN mkdir -p /opt/app && chown -R 1000:1000 /opt/app
WORKDIR /opt/app

#USER app_user
USER 1000
```

ポイントは、WORKDIR でフォルダを作成するのではなく、ユーザー権限のフォルダを先に作って、そのフォルダに移動する。コマンで、フォルダを作成すると、root 権限となってしまう。

```
RUN mkdir -p /opt/app && chown -R 1000:1000 /opt/app
WORKDIR /opt/app
```

コンテナを起動し、コンテ内でファイルを作成し、ホストで権限を確認する。

```
web $ docker run -it bfeb bash
node@bfdc7a628f07:/opt/app$ touch a.txt
node@bfdc7a628f07:/opt/app$ ls
a.txt
node@bfdc7a628f07:/opt/app$
```

ホストで、ユーザー権限となっていることが確認できる。

```
test2 $ ls -la
合計 16
drwxr-xr-x  3 atoru atoru 4096 11月 19 14:40 .
drwxr-xr-x 51 atoru atoru 4096 11月 18 21:39 ..
drwxr-xr-x  3 atoru atoru 4096 11月 19 14:47 .devcontainer
-rw-r--r--  1 atoru atoru 2643 11月 19 15:09 README.md
-rw-r--r--  1 atoru atoru    0 11月 19 14:17 a.txt
```

# docker-compose.yml

docker-compose.yml

```
services:
  test:
    build:
      dockerfile: ./web/Dockerfile
    stdin_open: true
    tty: true
    working_dir: '/opt/app'
    volumes:
      - ..:/opt/app
        # - ./app:/opt/app
        #- ~/.aws/:/root/.aws/
    user: "1000:1000"
```

ポイントは、docker-compose.yml で新たなフォルダを作成しないこと。`./app:/opt/app`と新たなフォルダを作成すると、root 権限になってしまう。
試しに、先にホストで`./app`を作成しておけば、ホストのユーザー権限となった。

# vscode よりコンテナ起動

devcontainer.json

```
{
  "name": "Amplify Vue3",
  "dockerComposeFile": "docker-compose.yml",
  "service": "test",
  "workspaceFolder": "/opt/app",
  // "remoteUser": "atoru",
  "updateRemoteUserUID": false,
}
```

`"updateRemoteUserUID": false,`とすること。

コンテナでファイル作成

```
node@aafdc671b6b6:/opt/app$ touch b.txt
node@aafdc671b6b6:/opt/app$
node@aafdc671b6b6:/opt/app$
```

ホストで権限を確認

```
test2 $ ls -la
合計 20
drwxr-xr-x  3 atoru atoru 4096 11月 19 15:24 .
drwxr-xr-x 51 atoru atoru 4096 11月 18 21:39 ..
drwxr-xr-x  3 atoru atoru 4096 11月 19 15:23 .devcontainer
-rw-r--r--  1 atoru atoru 5012 11月 19 15:20 README.md
-rw-r--r--  1 atoru atoru    0 11月 19 14:17 a.txt
-rw-r--r--  1 atoru atoru    0 11月 19 15:24 b.txt
```

コンテナで作成したファイルの権限がホストのユーザーとなっている。これで OK。
