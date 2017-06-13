---
layout: post
title: bash批量重命名文件
abstract: 使用bash对同模式多文件进行重命名
category: technical
permalink: bash-rename-file-batch
author: 木逸辰
tags: [tech, bash, regex]
---

### {{ page.title }}

`bash`操作文件是比较常用的操作，尤其对文件重命名这种，算是非常普遍了。最近下载一些影视资源后，有些文件名特别长，因为包含了许多广告啊等等额外信息，于是想给删掉，为了一劳永逸，于是写个`bash`程序来处理。

首先文件目录是这样的：
![original name](/assets/images/2017-06-13-bash-rename-original.png)

需要把名称改成“霹雳兵燹之问鼎天下0x.mkv”这种格式。直接用字符串匹配就行了：

    #列出文件名，循环重命名
    for fn in `ls`; do
        #先删除不需要的前缀
        tmp=${fn##[www.budaixi.cn][剧集][}
        mv $fn $tmp
        #再删除不需要的后缀
        tmp2=${tmp/][720P]/}
        mv $tmp $tmp2
    done

看起来很简单，于是执行试试，结果发现完全没变化，文件名全都没改变过来。看来模式匹配出了问题，难道是中文的原因？于是删除中文再匹配，结果还是一样。最后试了好几次才发现，原来是这几个中括号的问题，无法匹配到。既然直接写符号无法匹配，那就换正则试试。找了标点符号的正则表达式：`[:punct:]`，把所有中括号都替换过来（注意，用在模式里时，外面需再加一层中括号）：

    #列出文件名，循环重命名
    for fn in `ls`; do
        #先删除不需要的前缀
        tmp=${fn##[[:punct:]]www.budaixi.cn[[:punct:]][[:punct:]]剧集[[:punct:]][[:punct:]]}
        mv $fn $tmp
        #再删除不需要的后缀
        tmp2=${tmp/[[:punct:]][[:punct:]]720P[[:punct:]]/}
        mv $tmp $tmp2
    done

再执行，果然没问题了：
![destination name](/assets/images/2017-06-13-bash-rename-destination.png)

然后觉得这个模式太长了，最好是能用变量保存起来，执行时直接调用变量，于是改来试试：


    prefix=[[:punct:]]www.budaixi.cn[[:punct:]][[:punct:]]剧集[[:punct:]][[:punc    t:]]
    suffix=[[:punct:]][[:punct:]]720P[[:punct:]]

    for fn in `ls`; do
        tmp=${fn##prefix}
        mv $fn $tmp
        mv $tmp ${tmp/suffix/}
    done

执行，结果竟然失败！文件名无任何变化，想了半天，觉得应该是`pattern`里的变量名直接被当成`pattern`了，没解析成变量，于是试一下再加个`$`：

    prefix=[[:punct:]]www.budaixi.cn[[:punct:]][[:punct:]]剧集[[:punct:]][[:punc    t:]]
    suffix=[[:punct:]][[:punct:]]720P[[:punct:]]

    for fn in `ls`; do
        tmp=${fn##$prefix}
        mv $fn $tmp
        mv $tmp ${tmp/$suffix/}
    done

果然成功了！

任务完成，以后对类似的任务，只需稍微修改下`prefix`和`suffix`两个变量就可以了。
但是还是有个问题没解决：为什么直接从名称里复制过来的中括号都无法匹配呢？
