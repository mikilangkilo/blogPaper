---
title: 安卓基础-Android自动化打包脚本搭建
date: 2018-11-06 17:25:39
tag : "activity"
category : "安卓基础"
description : "本文主要是对Android自动化打包脚本搭建进行学习和反思"
---

从之前分析学习安卓的打包流程，然后参考各式方案，再结合项目需求，定下了一套目前来讲比较好的打包方法。

# 原理

安卓打包流程如下
![打包流程](/images/Android/Androidpackageimage.png)

通过apkbuilder构建之后就形成了未签名的apk包，然后通过jarsinger进行签名之后，就会生成签名的apk包。

换言之，在jarsinger签名之前的任何步骤，进行的操作，都需要重新经过apkbuilder，否则就不会生效，类似于R文件的生成，假如在apkbuilder之后重新想塞入一些资源，那么R文件是读不出来这些文件的。同时如果资源文件的错位，或许会导致之后索引产生一系列问题。最大的问题是，假如改了apkbuilder涉及的文件，会导致签名校验失败。

因此只能通过不参与签名校验的文件进行操作，唯一可以改的就是META-INF目录。

所以将需要放置的文件放到META-INF目录之后在进行对齐即可。

# 项目变更

由于市场的要求，我们的app在不同的平台上有不同的名字，比如在baidu上面，就是“好搭盒子”，而在小米和搜狗市场上面，叫“好搭盒子-穿衣搭配”，在其余的市场上面，叫"好搭盒子-教你穿衣搭配"。

虽然看起来只是名字的改变，但是如果通过打包想法来讲，改名字需要改manifest，这样就会导致签名验证失败，因此无法通过manifest改名字。

我的想法和行动是仍旧构建变体，分三个，一个是shortname变体，一个是middlename变体，另一个是longname变体，其实还有一种，是测试时候的dev变体（因为测试人员需要知道我给他的是debug包，通过名字知道是最直接的）

因此通过输入渠道的变化，构造不同的变体，但是对于已有构建好的就不需要重复构建了。

# 打包脚本

```
# coding=utf-8
import zipfile
import shutil
import os
import sys
import requests
import json
import urllib2
import subprocess
import pwd
import os
import re

findline = u"(versionCode\s+:.+)"
findversion = u"([0-9]+)"
findversionname = u"(?<= versionName      : \").+?(?=\")"
depgradlefilepath = "dependencies.gradle"
BRANCH = "dev"


def release(channelName):
    channelName = channelName
    print 'start build ' + channelName

    apk = './app/build/outputs/apk/dev/release/app-dev-release.apk'
    hasApk = os.path.exists(apk)
    path = './release_apks/dev/'
    clean_code = subprocess.check_call("./gradlew clean", shell=True)
    if clean_code != 0:
        print "clean failed"
        sys.exit()
    if channelName == 'long':
        apk = './app/build/outputs/apk/longname/release/app-longname-release.apk'
        path = './release_apks/long/'
        if os.path.exists("release_apks/long"):
            shutil.rmtree("release_apks/long")
        print "delete exist release_apks/long success"
        os.mkdir("release_apks/long")
        assemble_release_code = subprocess.check_call("./gradlew assembleLongnameRelease",
                                                      shell=True)
    elif channelName == 'short':
        apk = './app/build/outputs/apk/shortname/release/app-shortname-release.apk'
        path = './release_apks/short/'
        if os.path.exists("release_apks/short"):
            shutil.rmtree("release_apks/short")
        print "delete exist release_apks/short success"
        os.mkdir("release_apks/short")
        assemble_release_code = subprocess.check_call("./gradlew assembleShortnameRelease",
                                                      shell=True)
    elif channelName == 'middle':
        apk = './app/build/outputs/apk/middlename/release/app-middlename-release.apk'
        path = './release_apks/middle/'
        if os.path.exists("release_apks/middle"):
            shutil.rmtree("release_apks/middle")
        print "delete exist release_apks/middle success"
        os.mkdir("release_apks/middle")
        assemble_release_code = subprocess.check_call("./gradlew assembleMiddlenameRelease",
                                                      shell=True)
    else:
        if os.path.exists("release_apks/dev"):
            shutil.rmtree("release_apks/dev")
        print "delete exist release_apks/dev success"
        os.mkdir("release_apks/dev")
        assemble_release_code = subprocess.check_call("./gradlew assembleDevRelease", shell=True)

    if assemble_release_code != 0:
        print "assembleRelease failed"
        sys.exit()

    emptyFile = 'xxx.txt'
    f = open(emptyFile, 'w')
    f.close()

    if channelName == 'long':
        with open('channelNameLong.txt', 'r') as f:
            contens = f.read()
        lines = contens.split('\n')
    elif channelName == 'short':
        with open('channelNameShort.txt', 'r') as f:
            contens = f.read()
        lines = contens.split('\n')
    elif channelName == 'middle':
        with open('channelNameMiddle.txt', 'r') as f:
            contens = f.read()
        lines = contens.split('\n')
    else:
        with open('devChannel.txt', 'r') as f:
            contens = f.read()
        lines = contens.split('\n')

    if not os.path.exists(path):
        os.mkdir(path)
    else:
        for f in os.listdir(path):
            if not f.endswith('.gitignore'):
                os.remove(path + f)

    for line in lines:
        print line
        channel = 'channel_' + line
        destfile = path + '%s.apk' % channel
        shutil.copyfile(apk, destfile)
        zipped = zipfile.ZipFile(destfile, 'a', zipfile.ZIP_DEFLATED)
        channelFile = "META-INF/{channelname}".format(channelname=channel)
        zipped.write(emptyFile, channelFile)
        zipped.close()
    os.remove('./xxx.txt')

    for f in os.listdir(path):
        if f.endswith('.apk'):
            os.system('zipalign -f -v 4 ' + path + f + ' ' + path + 'temp-' + f)
            os.remove(path + f)

    for f in os.listdir(path):
        if f.startswith('temp-'):
            os.system('zipalign -f -v 4 ' + path + f + ' ' + path + f.replace('temp-', ''))
            os.remove(path + f)


def versionCodePlusPlus(msg):
    gitstash = subprocess.check_call("git stash", shell=True)
    if gitstash != 0:
        print "git stash fail"
        sys.exit()
    print "git stash success"
    gitcheckout = subprocess.check_call("git checkout " + BRANCH, shell=True)
    if gitcheckout != 0:
        print "git checkout dev fail"
        sys.exit()
    print "git checkout dev success"
    gitfetchall = subprocess.check_call("git fetch --all", shell=True)
    if gitfetchall != 0:
        print "git fetch all fail"
        sys.exit()
    gitresetall = subprocess.check_call("git reset --hard origin/" + BRANCH, shell=True)
    if gitresetall != 0:
        print "git reset hard origin dev fail"
        sys.exit()
    gitpull = subprocess.check_call("git pull", shell=True)
    if gitpull != 0:
        print "git pull fail"
        sys.exit()
    print "git pull success"

    originContent = open(depgradlefilepath).read()
    originVersionCodeLine = re.search(findline, originContent).group(0)
    originVersionCodeString = re.search(findversion, originVersionCodeLine).group(0)
    versionname = re.search(findversionname, originContent).group(0)
    originVersionCode = int(originVersionCodeString)
    finalVersionCode = originVersionCode + 1
    finalVersionCodeLine = "versionCode      : " + str(finalVersionCode) + ","
    finalContent = originContent.replace(originVersionCodeLine, finalVersionCodeLine, 1)
    open(depgradlefilepath, 'w').write(finalContent)
    gitadd = subprocess.check_call("git add .", shell=True)
    if gitadd != 0:
        print "git add failed"
        sys.exit()
    gitCommit = subprocess.check_call(
        u"git commit -m \"build/auto increase versionCode = " + str(
            finalVersionCode) + "," + msg + "\"",
        shell=True)
    if gitCommit != 0:
        print "gitCommit failed"
        sys.exit()
    gitPush = subprocess.check_call(
        "git push origin " + BRANCH, shell=True
    )
    if gitPush != 0:
        print "gitpush failed"
        sys.exit()

    gitpull = subprocess.check_call("git pull", shell=True)
    if gitpull != 0:
        print "git pull fail"
        sys.exit()
    print "git pull success"
    return finalVersionCode, versionname


if __name__ == '__main__':
    if os.path.exists("release_apks"):
        shutil.rmtree("release_apks")
        print "delete exist release_apks success"
    os.mkdir("release_apks")

    if os.path.exists("app/build"):
        shutil.rmtree("app/build")
    print "delete exist release_apks success"
    channel = 'dev'
    if len(sys.argv) >= 2:
        channel = sys.argv[1]
    versionCodePlusPlus(channel)
    release(channel)

```

其中做了一个额外的动作，就是versioncode++这个操作，这个主要也是项目需要，每次build的包，需要有迹可溯，因此每次build都会使versioncode+1，并且进行提交。

上面是打包的流程，还有个全渠道打包的执行文件就比较简单了

```
# coding=utf-8
from release import release, versionCodePlusPlus,BRANCH,findline,findversion,findversionname,depgradlefilepath
import os
import shutil
import subprocess


FLAVOR_FILE = 'all_channels.txt'
if __name__ == '__main__':
    if os.path.exists("release_apks"):
        shutil.rmtree("release_apks")
        print "delete exist release_apks success"
    if os.path.exists("app/build"):
        shutil.rmtree("app/build")
    print "delete exist release_apks success"
    os.mkdir("release_apks")

    versionCodePlusPlus('short')
    release('short')
    versionCodePlusPlus('middle')
    release('middle')
    versionCodePlusPlus('long')
    release('long')

```

因为也不算涉及到公司机密，以上都是很常规的操作，留下来做个备注防止以后采坑