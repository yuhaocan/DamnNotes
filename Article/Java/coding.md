# 问题
1. android开发选择本地相册图片用绝对路径获取bitmap为null，而有些图片则可以获取到。debug之后才知道有中文字符的绝对路径是按照UrlEncode编码显示，也就造成根据UrlEncode编码的路径找不到相应的bitmap对象。解决办法就是将UrlEncode格式进行解码获取真正的带有中文字符的正确路径。
```
URLDecoder.decode(path,"UTF-8");
```
2. 网络请求返回数据部分中文乱码
源代码
```
byte[] buffer = new byte[4096];
int count;
while ((count = inputStream.read(buffer)) > 0) {
   retValue.append(new String(buffer, 0, count, "utf-8"));
}
```
字节流读取每次读取4096个字节，UTF-8编码，英文1个字节，中文2个字节。但每次4096个字节可能刚好截取到某个汉字的一半，所以转字符串会造成部分中文乱码的现象。解决办法是将字节流读取更换成字符流就不会出现乱码现象了。
```
StringBuffer retValue = new StringBuffer();
InputStreamReader reader = new InputStreamReader(inputStream,"utf-8");
char buf[] = new char[4096];
int bufLen = reader.read(buf);
while (bufLen != -1){
     retValue.append(new String(buf, 0, bufLen));
     bufLen = reader.read(buf);
}
```

# 编码的历史
既然遇到了编码相关问题，那就需要知道什么是编码，为什么会有这么多种编码格式呢。
## 基本概念
1. 比特（bit）：也可称为“位”，是计算机信息中的最小单位，是 binary digit（二进制数位） 的 缩写，指二进制中的一位
2. 字节（Byte）：计算机中信息计量的一种单位，一个位就代表“0”或“1”，每8个位（bit）组成一个字节（Byte）
3. 字符（Character）：文字与符号的总称，可以是各个国家的文字、标点符号、图形符号、数字等
4. 字符集（Character Set）：是多个字符的集合
5. 编码（Encoding）： 信息从一种形式或格式转换为另一种形式的过程
6. 解码（decoding）： 编码的逆过程
7. 字符编码（Character Encoding）： 按照何种规则存储字符
## ASCII （American Standard Code for Information Interchange，美国信息交换标准代码）
> 8个晶体管的“通”或“断”即可以代表一个字节，刚开始，计算机只在美国使用，所有的信息在计算机最底层都是以二进制（“0”或“1”两种不同的状态）的方式存储，而8位的字节一共可以组合出256（2的8次方）种状态，即256个字符
## EASCII （Extended ASCII，延伸美国标准信息交换码）
> EASCII可以说是ASCII扩充，欧洲国家也开始使用上计算机后128个字符明显不够，于是，一些欧洲国家就决定，利用字节中闲置的最高位编入新的符号。不同的国家有不同的字母，因此，哪怕它们都使用256个符号的编码方式，代表的字母却不一样。但所有这些编码方式中0--127表示的符号是一样的，不一样的只是128--255的这一段。
## GB2312 （GB为国标汉语拼音的首字母）
> EASCII码对于部分欧洲国家基本够用了，但过后的不久，计算机便来到了中国，要知道汉字是世界上包含符号最多并且也是最难学的文字。 据不完全统计，汉字共包含了古文、现代文字等近10万个文字，就是我们现在日常用的汉字也有几千个，那么对于只包含256个字符的EASCII码也难以满足天朝的需求了。 于是⌈中国国家标准总局⌋（现已更名为⌈国家标准化管理委员会⌋）在1981年，正式制订了中华人民共和国国家标准简体中文字符集，全称《信息交换用汉字编码字符集·基本集》，项目代号为GB 2312 或 GB 2312-80，此套字符集于当年的5月1日起正式实施。

需要注意的是每个汉字及符号以两个字节来表示，第一个字节为“高位字节”，第二个字节为“低位字节”
## GBK 标准
> GBK 包括了 GB2312 的所有内容，同时又增加了近20000个新的汉字（包括繁体字）和符号。后来少数民族也要用电脑了，于是再扩展，又加了几千个新的少数民族的字，GBK 扩成了 GB18030。
## BIG5
> 要知道港澳台同胞使用的是繁体字，而中国大陆制定的GB2312编码并不包含繁体字，于是信息工业策进会在1984年与台湾13家厂商签定“16位个人电脑套装软件合作开发（BIG-5）计划”，并开始编写并推出BIG5标准。 之后推出的倚天中文系统则基于BIG5码，并在台湾地区取得了巨大的成功。在BIG5诞生后，大部分的电脑软件都使用了Big5码，BIG5对于以台湾为核心的亚洲繁体汉字圈产生了久远的影响，以至于后来的window 繁体中文版系统在台湾地区也基于BIG5码进行开发。

同样的用两个字节来为每个字符编码，第一个字节称为“高位字节”，第二个字节称为“低位字节”
## Unicode (Universal Multiple-Octet Coded Character Set)
> 在计算机进入中国大陆的相同时期，计算机也迅速发展进入了世界各个国家。 特别是对于亚洲国家而言，每个国家都有自己的文字，于是每个国家或地区都像中国大陆这样去制定了自己的编码标准，以便能在计算机上正确显示自己国家的符号。 但带来的结果就是国家之间谁也不懂别人的编码，谁也不支持别人的编码，连大陆和台湾这样只相隔了150海里，都使用了不同的编码体系。 于是，世界相关组织意识到了这个问题，并开始尝试制定统一的编码标准，以便能够收纳世界所有国家的文字符号。

## UTF（UCS Transfer Format）
UTF是为了考虑网上传输UNICODE的问题，顾名思义，UTF8 就是每次8个位传输数据，而 UTF16 就是每次16个位，只不过为了传输时的可靠性，从UNICODE到 UTF时并不是直接的对应，而是要过一些算法和规则来转换。
* UTF-8
> 使用1~4个字节为每个UCS中的字符编码：  
128个ASCII字符只需一个字节编码（Unicode范围由U+0000至U+007F）  
拉丁文、希腊文、西里尔字母、亚美尼亚语、希伯来文、阿拉伯文、叙利亚文及它拿字母需要二个字节编码（Unicode范围由U+0080至U+07FF）    
大部分国家的常用字（包括中文）使用三个字节编码  
其他极少使用的生僻字符使用四字节编码
* UTF-16/UCS-2
> UCS-2的父集，使用2个或4个字节来为每个UCS中的字符编码：  
 128个ASCII字符需两个字节编码  
 其他字符使用四个字节编码
* UTF-32/UCS-4
> UCS-2的父集，使用2个或4个字节来为每个UCS中的字符编码：  
128个ASCII字符需两个字节编码  
其他字符使用四个字节编码


（可以这样理解：Unicode是字符集，UTF-32/ UTF-16/ UTF-8是三种字符编码方案。）

## URL编码
>URL只能使用英文字母、阿拉伯数字和某些标点符号，不能使用其他文字和符号。比如，世界上有英文字母的网址"http://www.abc.com"，但是没有希腊字母的网址"http://www.aβγ.com"（读作阿尔法-贝塔-伽玛.com）。这是因为网络标准RFC 1738做了硬性规定：   
"...Only alphanumerics [0-9a-zA-Z], the special characters "$-_.+!*'()," [not including the quotes - ed], and reserved characters used for their reserved purposes may be used unencoded within a URL."
"只有字母和数字[0-9a-zA-Z]、一些特殊符号"$-_.+!*'(),"[不包括双引号]、以及某些保留字，才可以不经过编码直接用于URL。"

参考资料：

[http://blog.csdn.net/ldanduo/article/details/8203532/](http://blog.csdn.net/ldanduo/article/details/8203532/)

[http://djt.qq.com/article/view/658?ADTAG=email.InnerAD.weekly.20130902&bsh_bid=281085951](http://djt.qq.com/article/view/658?ADTAG=email.InnerAD.weekly.20130902&bsh_bid=281085951)

[http://blog.csdn.net/xingcaizxc/article/details/8736399](http://blog.csdn.net/xingcaizxc/article/details/8736399)
[http://www.ruanyifeng.com/blog/2010/02/url_encoding.html](http://www.ruanyifeng.com/blog/2010/02/url_encoding.html)