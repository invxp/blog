---
title: "[开源小工具]IIDX歌单自定义编辑器"
date: 2022-09-29T18:55:52+08:00

tags: ["go", "iidx"]

author: "阿猫"
showToc: false
TocOpen: false
hidemeta: false
comments: true
description: ""

disableShare: true
ShowReadingTime: true
ShowBreadCrumbs: true

ShowPostNavLinks: true

ShowWordCount: true
ShowRssButtonInSectionTermList: true

cover:
    image: ""
    alt: ""
    caption: ""
---
### 快速开始
```shell
$ go get github.com/invxp/iidxfavlist
$ cd iidxfavlist
$ go build cmd/iidxfavlist
$ ./iidxfavlist
```

---

### beatmaniaIIDX 歌单编辑器
1. 能够自定义编辑歌单
2. 能够方便的模糊查找歌单与游戏系统库的歌曲
3. 本应用依赖playlister插件
![](/blog/game.png)

---

### 使用教程
![](/blog/menu.png)

#### 请确保放到beatmaniaIIDX游戏目录下
#### 输入对应的命令即可
![](/blog/edit.png)
* e: 增加/修改/删除歌单
* l: 查询歌单
* r: 重命名歌单/修改歌单模式(SP/DP)
* s: 搜索游戏歌曲库.如:'s {id}/{artist}/{songname}' 除ID外支持模糊查询
* f: 搜索当前歌单库.如:'f {id}/{artist}/{songname}' 除ID外支持模糊查询
* b: 返回上一级菜单(部分功能具备)
* q: 退出应用
