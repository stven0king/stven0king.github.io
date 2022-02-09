---
title: 【亲测】Shell日期时间和时间戳相互转化
date: 2022-01-16 16:10:58
tags: [时间戳]
categories: shell
description:  "先说一下为什么写这篇文章，因为目前没有一篇文章能让我在Mac上成功执行的脚本。"
---

`date -d` 在Mac上提示以下错误：
```log
date: option requires an argument -- d
usage: date [-jnRu] [-d dst] [-r seconds] [-t west] [-v[+|-]val[ymwdHMS]] ...
            [-f fmt date | [[[mm]dd]HH]MM[[cc]yy][.ss]] [+format]
```

以下时间戳都是以秒为单位

- 自定义日期时间转时间戳
```shell
#/bin/sh
help="?"
if [ $# != 1 ] ; then
  echo "参数为空，输入的时间格式为：2022-01-16 15:26:11"
  exit 0
elif [[ $1 = "?" ]]; then
  echo "输入的时间格式为：2022-01-16 15:26:11"
  exit 0
fi
echo "北京时间："$1
echo "时间戳：" $(date -j -f "%Y-%m-%d %H:%M:%S" "$1" +%s)
```

- 时间戳转日期
```shell
#/bin/sh
echo "北京时间："$(date -r $1)
echo "时间戳："$1
```


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！