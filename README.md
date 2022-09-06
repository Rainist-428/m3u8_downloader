> **声明：本文只作学习研究，禁止用于非法用途，否则后果自负，如有侵权，请告知删除，谢谢！**

@[TOC](Python爬取m3u8格式的视频目录)
# 背景
在某一天，群友分享了一些小视频，手机端可以正常观看，但是到了电脑上，输入网址之后会下载下来一个m3u8格式的文件，这就让我犯了难。所以我就研究了一下，并使用Python来将该文件爬取了下来。
参考文章如下：
[西北乱跑娃 --- python m3u8库](https://blog.csdn.net/human_soul/article/details/103263573)
[Python 手把手实现M3U8视频抓取](https://blog.csdn.net/weixin_38640052/article/details/119680697)
[python实战案例：解析m3u8视频文件](https://www.bilibili.com/read/cv15078586)
[python爬取m3u8视频教程](https://blog.csdn.net/weixin_42587620/article/details/124591215)
# 1.文件信息
链接图如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/ccfb9dc0a1db46b9a47aea94fd3ec942.png)
下载下来呢，也是m3u8文件，一种非常特殊的文件。我们点击查看，就可以查看到
3u8的一些信息
我在这里屏蔽了`网址`，免得暴露我在下载什么hhhhh

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:2
#EXT-X-PLAYLIST-TYPE:VOD
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-KEY:METHOD=AES-128,URI="https://h09.t*********wc.com/20220703/9K***sCCV/30**0kb/hls/key.key"
#EXTINF:1.16,
https://h09.*********.com/20220703/9K***V/300**b/hls/Foszl**SV.ts
#EXTINF:1.2,
https://h09.**********.com/20220703/9KR**CV/30**kb/hls/b7tCBoOu.ts
```
经过查阅资料，该文件的我信息大致如下
```
# EXTM3U：.m3u8文件的格式定义
# EXT-X-KEY： 密钥的信息
# METHOD： 加密的方法，这里采用的是AES-128的加密方式
# URI： 密钥的地址，需要获取访问得到密钥的信息
# IV： 偏移量，AES加密的方法，通过这个密钥就可以解密，获取正确的视频信息
```


## 那什么是m3u8呢？

M3U8 是 Unicode 版本的 M3U，用 UTF-8 编码。"M3U" 和 "M3U8" 文件都是苹果公司使用的 HTTP Live
Streaming（HLS） 协议格式的基础，这种协议格式可以在 iPhone 和 Macbook 等设备播放。

> 通俗点可以理解为，一个视频文件拆分为多个.ts片段的视频文件，然后将所有的ts合成MP4文件就可以播放了。但是为什么有些在线或者m3u8的播放器会用不了呢？由于大部门的m3u8生成的结果不太一致，与该播放器的解密格式不一样，所以导致打开。

一般来说，在m3u8文件中都会有key以及AES的偏移量IV，我们根据其中的URL来获取他们，之后我们根据m3u8文件中的`.ts`的文件URL来下载视频片段，解密之后拼接成`mp4`的文件就行了。
# 2.构造请求获得m3u8文件
我们已经有了m3u8文件的URL，因此免去了抓包这一项工作。我们直接构造response即可

```python
def get_file(url,count):
    resp=requests.request('GET',url)
    # print(resp.content)
    with open('./m3u8_link/'+str(count)+'.m3u8','wb') as f:
        f.write(resp.content)
```
# 3.获得m3u8文件中的key以及偏移量IV
根据我们上一步获得的文件内容，这里我选择使用正则来匹配出来key以及IV对应的URL，之后在构造resp即可获得其内容。
这里要说明一下，我们的m3u8文件中AES加密的时候是没有使用IV的，所以使用时实际上令`IV=Key`
`.*?`，经典的正则在爬虫中的使用了。

```python
def get_key(path):
    with open(path, 'rb') as f:
        file_content=str(f.read())
    print(file_content)
    key_link = re.search('URI=\"(.*?)\"', file_content).group(1)
    key=requests.request('GET',key_link).content
    return key
```
# 4.获取.ts文件链接
其实这一步也可以使用正则来继续匹配，但是我偷了一点小懒，使用了m3u8库来进行匹配。

```python
import m3u8
def get_ts_list(path):
    m3u8_obj = m3u8.load(path)
    ts_urls = []
    for i, seg in enumerate(m3u8_obj.segments):
        # if i<=100:
            ts_urls.append(seg.uri)
    # print(ts_urls)
    return ts_urls
```
经过这一步之后，我们就获得了m3u8文件中整个`.ts`文件对应的URL，接下来只要根据`.ts`文件对应的URL来获取视频片段，解密之后拼接就可以了。
# 5.进行解密
解密的库 `pip install pycryptodome`
就是一些常规的AES解密操作了，我们获得的m3u8文件中没有IV偏移量，这里用使`iv＝key`即可。注意，AES要补0的，补字节0到16的倍数。
```python
from Crypto.Cipher import AES
sprytor = AES.new(key, AES.MODE_CBC,iv)
# 获取ts文件二进制数据
ts = requests.get(ts_url).content
# 密文长度不为16的倍数，则添加b"0"直到长度为16的倍数
# decrypt方法的参数需要为16的倍数，如果不是，需要在后面补二进制"0"
while len(ts) % 16 != 0:
    ts += b"0"
with open(name, "ab") as file:
    file.write(sprytor.decrypt(ts))
```
# 6.下载拼接
之后下载以字节流的形式进行拼接就可以了。

```python
def download(ts_urls,key,iv,count,path):
    name=path
    print("视频",count,"需要下载的文件长度为", len(ts_urls))
    for i in range(len(ts_urls)):
        ts_url=ts_urls[i]
        if i%10==0:
            print("视频",count,"当前下载进度：",str(i/len(ts_urls)*100)[:4],'%')
        # 如果连接末尾没有.ts手动加上
        ts_name = ts_url.split("/")[-1] + '.ts'  # ts文件名
        # 解密，new有三个参数，
        # 第一个是秘钥（key）的二进制数据，
        # 第二个使用下面这个就好
        # 第三个IV在m3u8文件里URI后面会给出，如果没有，可以尝试把秘钥（key）赋值给IV
        sprytor = AES.new(key, AES.MODE_CBC,iv)
        # 获取ts文件二进制数据
        ts = requests.get(ts_url).content
        # 密文长度不为16的倍数，则添加b"0"直到长度为16的倍数
        while len(ts) % 16 != 0:
            ts += b"0"
        # 写入mp4文件
        with open(name, "ab") as file:
            # # decrypt方法的参数需要为16的倍数，如果不是，需要在后面补二进制"0"
            file.write(sprytor.decrypt(ts))
    print(name, "下载完成")
```
其实还可以使用多线程优化，完整项目请见本人的github。
