---
layout: post
title:  "为什么一个字节是8位"
subtitle: ""
date:   2020-07-11
background: '/img/imac_bg.png'
---

有没有思考过为什么一个byte是8位呢？一起来看下Quora上这个回答。

回答摘自：[Quora:Why is one byte formed by 8 bits?](https://www.quora.com/Why-is-one-byte-formed-by-8-bits)

>Buddha Buck
>>Historical accident. Early computers didn’t necessarily use 8-bit bytes.
>>
>>The term “byte” was coined in the 1950’s to refer to the addressable blocks of memory in the IBM 7030 Stretch computer. The size of a byte was variable, specified in the instruction, and was spelled “byte” to avoid accidentally getting shortened to “bit”.
>>
>>Other early computers worked with 4-bit bytes, 6-bit bytes, 7-bit bytes, based on the size of the character set they used and the choice of the hardware designer.
>>
>>The extremely popular IBM System/360 mainframe used an 8-bit byte, which helped increase the popularity of 8 bits. Around the same time, AT&T started introducing 8-bit  𝜇− law encoding for transmitting sound digitally over its lines, which also made 8-bits a convenient data transfer size for going over AT&T lines as well.
>>
>>So by the time that microcomputers were first being designed in the early 1970’s, there had already been a decade of encoding data in 8-bit chunks. The early microcomputers were thus designed around 8-bit data, so they used 8-bit bytes.
>>
>>IBM chose to use 8-bit characters for their System/360 for a few reasons. The first being that they wanted a character code that was roughly compatible with their previous encodings (which had a history dating back to the pre-computer punchcard systems originally designed in the late 1800’s), yet large enough to hold all the characters they felt were necessary to deal with. Some argue that they didn’t succeed in the latter, but it was a consideration of theirs. The pre-existing encoding they used was 6-bit, and extending it to just 7 bit wouldn’t allow them to maintain compatibility. So they went to 8-bit.
>>
>>If they had decided to just double the size of the previous 6-bit encoding to 12 bits, it is entirely possible that today we’d be asking why a byte was EXACTLY 12 bits.

看懂了吗？看不懂没关系，DeepL来翻译一下：

>历史的偶然。早期的计算机并不一定使用8位字节。
"字节 "一词是20世纪50年代发明的，指IBM 7030 Stretch计算机中的可寻址内存块。一个字节的大小是可变的，在指令中指定，并拼写为 "byte"，以避免意外地被缩短为 "bit"。
其他早期的计算机根据其使用的字符集大小和硬件设计者的选择，有4位字节、6位字节、7位字节的工作方式。
极为流行的IBM System/360大型机使用了8位字节，这有助于提高8-bit的普及率。大约在同一时间，AT&T开始引入8位𝜇-法编码，用于在其线路上传输声音的数字信号，这也使得8位也成为通过AT&T线路进行数据传输的方便尺寸。
因此，在20世纪70年代初微机刚开始设计的时候，已经有10年的时间用8位的数据块进行编码了。因此，早期的微型计算机是围绕着8位数据设计的，所以他们使用的是8位字节。
IBM选择为他们的System/360使用8位字符有几个原因。首先是他们想要一个与他们以前的编码大致兼容的字符代码（其历史可以追溯到19世纪末最初设计的计算机前打卡系统），但又足够大，可以容纳他们认为有必要处理的所有字符。有些人认为他们在后者上没有成功，但这是他们的一个考虑。他们使用的现有编码是6位的，如果只扩展到7位，就不能让他们保持兼容性。所以他们采用了8位。
如果他们决定只把以前的6位编码的大小增加一倍，达到12位，那么今天我们完全有可能会问为什么一个字节完全是12位。
*【通过www.DeepL.com/Translator（免费版）翻译】*

所以呢，1byte=8bit并不是一开始的选择，首先*byte*这个词是IBM发明使用的，但是*byte*这个概念肯定都有，表示用于编码单个字符所需要的比特数量。而各家具体使用多少bit都不一样，看自己的需求和选择。而IBM选择8-bit是想兼容以前的字符代码，又稍微比以前大一点，所以选择了8。而IBM大型机的流行又帮助推广了8-bit。再到后来，微机设计时候，直接就沿用了这个方式直到今天，现在已经变成了事实上标准了。
