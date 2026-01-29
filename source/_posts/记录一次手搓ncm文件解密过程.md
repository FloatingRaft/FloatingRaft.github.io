---
title: 记录一次手搓ncm文件解密过程
---

**免责声明**

本文为纯技术性分析文章，旨在探讨NCM文件格式的加密原理，属于密码学领域的学术研究范畴。请注意：

1. **研究目的**：仅供学习、研究加密技术实现原理
2. **合法使用**：不鼓励、不支持任何形式的版权侵权行为
3. **责任自负**：读者应对自身行为承担法律责任
4. **支持正版**：我们鼓励通过官方渠道获取音乐内容

本文不提供任何具体的解密工具或代码，仅进行技术原理分析，符合合理使用原则。

# 前言

在给校队招新赛出一道音频隐写题时，在网易云里精挑细选半天后挑了这首歌`LANY - 13`，然后下载毫不意外的只拿到了ncm文件

想到之前网上有大量的ncm文件在线解密工具，github上也有一些关于ncm解密的工具，还有之前在某个论坛看到有人提到过ncm加密方式其实比较草率，就想着自己手动解密试试

毕竟ncm文件是可以被本地的网易云音乐正常解码并播放的，说明加密密钥要么是固定的要么是存储在文件内部，并且解密流程肯定不复杂，否则应该会影响到用户听歌体验（上述内容为笔者个人猜测，如有谬误望大佬们指出）

Deepseek给出如下评价：

> 这个工具的存在说明网易云音乐的.ncm加密主要是**防君子不防小人**的轻度保护，主要目的是防止直接文件分享，而不是高强度安全加密。

# 参考文章和参考项目

[NCM文件的加解密笔记 | manalogues](https://manalogues.com/posts/2025/03/10#概述)

[Orinpl/Ncm2Flac: 将网易云下载的ncm音频文件转换为flac格式方便在播放器上播放](https://github.com/Orinpl/Ncm2Flac?tab=readme-ov-file)

# 正文

## 文件结构

根据大佬总结的文件结构，笔者对手里的ncm文件的关键数据段进行了逐步解析

### 文件头

偏移量：`0x00`

长度：8B

备注：固定为`43 54 45 4E 46 44 41 4D`，即`CTENFDAM`

![image-20260126121503276](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260126121510.png)

### AES密钥长度

偏移量：`0x0A`

长度：4B

备注：为小端存储数据，大端序存储为`0x00000080`，即128，表示加密标准为AES128，目前只存在这一种规格

![image-20260126122158487](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260126122158.png)

### 音频AES密钥

偏移量：`0x0E`

长度：128B

备注：由ASCII字符构成，AES-ECB加密，异或`0x64`后需使用已知的密钥解密，明文开头固定为`neteasecloudmusic`

![image-20260126164815045](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260126164815.png)

### 元数据长度

偏移量：`0x8E`

长度：4B

备注：小端序存储，大端序数据为`0x00000256`，即598，表示元数据长度为598字节

### 元数据

偏移量：`0x92`

长度：meta_lenth

备注：AES-ECB加密，Base64表示的JSON文本，异或`0x63`后开头固定为`163 key(Don't modify):`，需使用已知的密钥解密

### 封面CRC

偏移量：`0x92+meta_lenth`

长度：4B

备注：封面图片的CRC校验值

### 封面数据长度

偏移量：`0x9B+meta_lenth`

长度：4B

备注：封面图片的数据长度，小端序存储，大端序数据为`0x0002AE76`，即175734，表示封面数据长度为175734字节

### 封面数据

偏移量：`0x9F+meta_lenth`

长度：image_size

备注：无加密，原始jpg文件数据，`FFD8`标记文件开始，以`FFD9`标记文件数据结束

### 音频数据

偏移量：`0x9F+meta_lenth+image_size`

备注：将音频密钥基于RC4算法生成密钥流，再与密钥流进行异或运算

## 解密流程

### 1.文件头校验

根据Orinpl佬的项目源代码来看，狗屎网易云音乐的ncm格式竟然还有flac头

（这不是我说的啊这是代码里写的）

![image-20260127103134405](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260127103134.png)

因此需要对文件头进行校验，也就是文件开头的8字节数据

ncm文件的文件头为`0x4354454E4644414D`，对应的ASCII码为`CTENFDAM`

manalogues佬在文章中提到了如下内容

> anoymous5l将其标注为2*4B、小端存储的文件头，此时对应`NETCMADF`，似乎更有意义一些~~（NETease Cloud Music AuDio Format，仅为随意猜测）~~

（感觉挺有道理的）

### 2.获取音频密钥

#### 2.1 定位密文

即上文中“文件结构”所提及的音频AES密钥部分，其偏移量与长度已给出，这里不再赘述

#### 2.2 解密

##### 2.2.1 异或运算

密钥采用的是AES-ECB方式加密，并在加密后与`0x64`进行了按字节异或运算。由于异或的自反性，所以可以通过一次相同的异或运算得到AES-ECB加密后的密文

![image-20260127104907844](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260127104907.png)

##### 2.2.2 AES解密

根据Orinpl佬的项目代码可以知道，这里用于AES-ECB加密的密钥`core_key`为`0x687A4852416D736F356B496E62617857`

以AES-ECB模式解密可以得到如下图所示的结果

![image-20260127105415963](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260127105416.png)

根据manalogues佬的文章提到的，这串音频密钥的明文满足如下格式

```
neteasecloudmusic?????????????E7fT49x7dof9OKCgg9cdvhEuezy3iZCL1nFvBFd1T4uSktAJKmwZXsijPbijliionVUXXg9plTbXEclAE9Lb\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e
```

其中密钥前17字节固定为`neteasecloudmusic`，需要进行剔除。此处的`?`代表的是一串数字，似乎即使下载同一首歌的同一个音质版本，不同用户得到的这串数字也不尽相同，相同用户下载不同乐曲或同意乐曲的不同音质版本也不同，但相同用户在不同时间内重复下载同一首歌，这串数字并不会改变，猜测可能是某种用户标识符。由于密钥长度小于128B，因此在加密前还对密钥进行了填充，填充内容为填充字节数，这一部分也需要剔除

笔者发现，manalogues佬给出的这串音频密钥中的用户标识符为13位，而笔者这里的用户标识符为14位，也就是说用户标识符的长度也不尽相同，那么也就是说明后续的填充部分的填充内容也不尽相同

按照manalogues佬的文章来说，笔者这里得到的明文末尾应该存在13字节的`\x0d`填充，但CyberChef中并没有显示出这一部分，原因暂时未知，暂时先搁置一边

根据manalogues佬的小范围测试，似乎后面的部分，即`E7fT49x7dof9OKCgg9cdvhEuezy3iZCL1nFvBFd1T4uSktAJKmwZXsijPbijliionVUXXg9plTbXEclAE9Lb`是完全一致的，即使是不同乐曲、不同音质、不同用户，这一段都是完全一致的

### 3.获取元数据

#### 3.1 定位密文

即上文中“文件结构”所提及的元数据部分，其偏移量与长度已给出，这里不再赘述

#### 3.2 解密

##### 3.2.1 异或运算

与音频密钥同理，元数据也被按字节进行了异或运算，但运算数为`0x63`。异或后得到如下结果

![image-20260127113637048](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260127113637.png)

##### 3.2.2 AES解密

前22字节固定为`163 key(Don't modify):`，之后是以Base64编码的元数据密文，Base64解码后得到密文

然后根据Orinpl佬的项目代码可以知道，这里对元数据进行AES-ECB加密使用的密钥`meta_key`为`2331346C6A6B5F215C5D2630553C2728`

以AES-ECB模式解密可以得到如下图所示的结果

![image-20260127141553655](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260127141553.png)

根据manalogues佬的文章，末尾存在和上文相同的填充方式，但这里依然没有显示

将得到的json数据格式化

```json
music: {
	"musicId": "480648759",
	"musicName": "13",
	"artist": [["LANY", "999460"]],
	"albumId": "35670398",
	"album": "LANY",
	"albumPicDocId": "109951167126498354",
	"albumPic": "http://p4.music.126.net/dLy5SCiUAHgACytZOXUmjQ==/109951167126498354.jpg",
	"bitrate": 320000,
	"mp3DocId": "d15998cbfd698914249c4e548ebc358a",
	"duration": 234818,
	"mvId": "",
	"alias": [],
	"transNames": [],
	"format": "mp3",
	"fee": 1,
	"volumeDelta": -9.0401,
	"privilege": {
		"flag": 3637252
	}
}
```

### 4.获取封面

元数据结尾得到4B的封面文件CRC，间隔5个字节后得到用4B小端表示的封面文件大小。封面文件没有做任何封装，直接读取即可。上例中的CRC值为`0x7E95C1CC`封面大小为`0x0002AE76`，即`52376`字节，读取后可以得到一个JPG格式的封面图片。（此处间隔的5B是`0x0002AE7601`，似乎是一个与文件大小有关的值）

和manalogues佬的数据对照了一下，发现间隔的那5B似乎第一个字节都是`0x01`，后面四个字节则为封面大小的十六进制小端序表示

这个JPG数据是可以直接提取出来保存为`.jpg`文件的，提取出来的结果如下

![1](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260127143343.jpg)

### 5.获取音频

音频文件使用RC4算法加密，由于该算法同属于对称加密算法，只需构建出密钥流即可对密文进行解密

不过这里采用的RC4在基础的RC4算法上进行了一些变化

标准的RC4算法流程大致分为KSA和PGRA两个阶段

**KSA（Key Scheduling Algorithm）密钥调度算法** 初始化S盒（长度一般为256，存储0~255的排列），定义一个可存储256字节的T盒，利用给定密钥循环填充T盒。用T盒打乱S盒顺序，形成初始状态

**PRGA（Pseudo-Random Generation Algorithm）伪随机数生成算法**  不断交换 S 盒元素，生成密钥流字节。 将密钥流与明文逐字节异或，完成加密

这里采用的RC4对PRGA阶段进行了一定的改动，python代码实现解密如下：

```python
# KSA阶段
# 初始化S-Box
s_box = bytearray(range(256))
# 定义音频密钥key
key = "xxxxxxxxxxxxxxxx"  # 解密后的音频密钥

j = 0
for i in range(256):
    # 计算j
    j = (j + s_box[i] + key[i % len(key)]) & 0xff # 循环填充T-Box
    # 交换数据
    swap = s_box[i]
    s_box[i] = s_box[j]
    s_box[j] = swap
    
# PRGA阶段
# 原数据流读取
source = "xxxxxxxxxxxxxxxx" # 音频数据

output = bytearray()

i, j = 0, 0
for r in range(len(chunk)):
    i = (i + 1) & 0xff
    j = (i + s_box[i]) & 0xff
    output.append(source[r] ^ s_box[(s_box[i] + s_box[j]) & 0xff])
    

```

Orinpl佬对上述解密流程进行了优化，改用numpy包的矩阵进行解密，并采用了多线程，大大提高了解密效率

利用上述解密逻辑，对加密音频流的前10个字节`0xAC7198A13955A8566205`进行解密，得到的解密后的数据为`0x49443303000000000900`，对应ASCII文件头为`ID3`，即.mp3格式文件的文件头，验证解密成功

![image-20260129134519251](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260129134519.png)

# 总结

ncm文件整体文件结构如图

![image-20260129141842692](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260129141842.png)

播放器解析ncm文件的流程大致如下

![deepseek_mermaid_20260129_7819a6](https://cdn.jsdelivr.net/gh/FloatingRaft/picgo@main/20260129141956.png)

其实总体上来说，ncm文件的加密逻辑并不复杂，但是笔者此前并没有很多关于加密文件逆向的经验，所以稍微多花了一些时间，如有谬误请指出

