---
title: "音视频-HLS"
date: 2020-08-10T21:59:56+08:00
tag : "音视频"
category : "进阶-音视频"
description : "本文对近期工作上面处理的电台视频流做了一次复盘学习"
---

HLS是apple的动态码率自适应技术，主要包括一个m3u(8)的索引文件，ts媒体分片文件和key加密串文件

# M3U(8)

```

M3U8文件是M3U文件的一种，只不过它的编码格式是UTF-8。M3U使用Latin-1字符集编码。M3U的全称是Moving Picture Experts Group Audio Layer 3 Uniform Resource Locator，即mp3 URL。M3U是纯文本文件；

所以UTF-8编码的M3U文件也简称为 M3U8；

作者：JeffMony
链接：https://www.jianshu.com/p/dbac4c041de8
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

 