---
title: 人生苦短，我用Python--分分钟下载知乎美图给你看
date: 2016-08-11 19:45:46
categories: Python
tags: [爬虫,数据采集,知乎]
toc: true
---
### 起

上次说了要爬知乎的图片，于是花了一下午的时间去完成这件事，发现暂时接触到的爬虫总是逃脱不了一个规律：
- 模拟登陆
- 获取真实网页HTML源代码
- 解析获取到的网页源代码
- 获取想要的资源(下载到某个文件夹或者输出到表格中整合起来)

也许和我说的有一些出入，应该是刚学这个东西的原因，接下来还想研究一下多线程爬虫、添加代理、爬取海量数据并整合成图表形式，先把能做的做了。

<!--more-->
### 承

因为是在上一次的基础上进行的，所以没有看[**上一篇文章**](http://www.limuyang.cc/2016/08/09/%E4%BA%BA%E7%94%9F%E8%8B%A6%E7%9F%AD%EF%BC%8C%E6%88%91%E7%94%A8Python-%E4%B8%80%E8%B5%B7%E6%9D%A5%E7%88%AC%E7%9F%A5%E4%B9%8E%E5%A8%98/)的可以先看一下，这里用到的工具跟之前一样：

- win7 64位 旗舰版
- Python 3.5 64-bit
- PyCharm

这里模拟登陆是跟之前一样的代码，直接贴就是：

```python
logn_url = 'http://www.zhihu.com/#signin'

session = requests.session()

headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.82 Safari/537.36',
}

content = session.get(logn_url, headers=headers).content
soup = BeautifulSoup(content, 'html.parser')


def getxsrf():
    return soup.find('input', attrs={'name': "_xsrf"})['value']
    # 获取验证码
def get_captcha():
    t = str(int(time.time() * 1000))
    captcha_url = 'http://www.zhihu.com/captcha.gif?r=' + t + "&type=login"
    r = session.get(captcha_url, headers=headers)
    with open('captcha.jpg', 'wb') as f:
        f.write(r.content)
        f.close()
    # 用pillow 的 Image 显示验证码
    # 如果没有安装 pillow 到源代码所在的目录去找到验证码然后手动输入
    try:
        im = Image.open('captcha.jpg')
        im.show()
        im.close()
    except:
        print(u'请到 %s 目录找到captcha.jpg 手动输入' % os.path.abspath('captcha.jpg'))
    captcha = input("please input the captcha\n>")
    return captcha


def isLogin():
    # 通过查看用户个人信息来判断是否已经登录
    url = "https://www.zhihu.com/settings/profile"
    login_code = session.get(url, allow_redirects=False).status_code
    if int(x=login_code) == 200:
        return True
    else:
        return False


def login(secret, account):
    # 通过输入的用户名判断是否是手机号
    if re.match(r"^1\d{10}$", account):
        print("手机号登录 \n")
        post_url = 'http://www.zhihu.com/login/phone_num'
        postdata = {
            '_xsrf': getxsrf(),
            'password': secret,
            'remember_me': 'true',
            'phone_num': account,
        }
    else:
        print("邮箱登录 \n")
        post_url = 'http://www.zhihu.com/login/email'
        postdata = {
            '_xsrf': getxsrf(),
            'password': secret,
            'remember_me': 'true',
            'email': account,
        }
    try:
        # 不需要验证码直接登录成功
        login_page = session.post(post_url, data=postdata, headers=headers)
        login_code = login_page.text
        print(login_page.status)
        print(login_code)
    except:
        # 需要输入验证码后才能登录成功
        postdata["captcha"] = get_captcha()
        login_page = session.post(post_url, data=postdata, headers=headers)
        login_code = eval(login_page.text)
        print(login_code['msg'])
```
这里的代码来自GitHub上的[**fuck-login**](https://github.com/xchaoinfo/fuck-login)项目，在此表示感谢，我在原始代码上进行了改进，原始代码是适配了Python2.x和Python3.x，但是我学的是Python3.x所以去掉了一些我没用过的模块，也就是说我改进了后的代码是适用于Python3.x的。

下面就是准备获取图片了，先找一个目标，最近有一个问题很火：
> [**长得好看，但没有男朋友是怎样的体验**](https://www.zhihu.com/question/37709992)

还记得我列出来的步骤么，模拟登陆之后是获取真实的网页源代码，什么叫真实的，这个问题问得好，你没发现知乎很喜欢用动态加载技术么，也就是说，你看到的只是表象，这里也一样。

### 转

![问题界面](https://www.tuchuang001.com/images/2018/04/27/2b2973716c9c095a.png)

来，我们先点开点赞数最高的妹子上传的图片：

![Beauty](https://www.tuchuang001.com/images/2018/04/27/beauty.png)

咳咳咳，好像跑偏了，我们的目标是星辰大海，正确的做法是鼠标右键查看网页源代码：

![网页源代码](https://www.tuchuang001.com/images/2018/04/27/b5e4d869e42a7659.png)

是不是看到了很多图片链接，当然我们要找.jpg、.jpeg、.png后缀的：

```vbscript-html
<img data-rawwidth="1632" data-rawheight="2040" src="//zhstatic.zhihu.com/assets/zhihu/ztext/whitedot.jpg" class="origin_image zh-lightbox-thumb lazy" width="1632" data-original="https://pic2.zhimg.com/b6274542f3785c27ab4a38d4db906efd_r.jpg" data-actualsrc="https://pic2.zhimg.com/b6274542f3785c27ab4a38d4db906efd_b.jpg">
```

这里有两个：`data-original`和`data-actualsrc`，实际查看的图片是`data-original`的图片比`data-actualsrc`的大，下载下来也是如此，但是因为是使用正则去匹配规则，而`data-original`有多项，上面代码只是贴出来的一部分，实际匹配的结果类似这样：

data-actualsrc
```
data-actualsrc="https://pic2.zhimg.com/be7600989233bdf438e5ba23f2cdb685_b.jpg">
data-actualsrc="https://pic2.zhimg.com/b6274542f3785c27ab4a38d4db906efd_b.jpg">
```

data-original

```
data-original="https://pic2.zhimg.com/be7600989233bdf438e5ba23f2cdb685_r.jpg">
data-original="https://pic2.zhimg.com/be7600989233bdf438e5ba23f2cdb685_r.jpg" data-actualsrc="https://pic2.zhimg.com/be7600989233bdf438e5ba23f2cdb685_b.jpg">
data-original="https://pic2.zhimg.com/b6274542f3785c27ab4a38d4db906efd_r.jpg">
data-original="https://pic2.zhimg.com/b6274542f3785c27ab4a38d4db906efd_r.jpg" data-actualsrc="https://pic2.zhimg.com/b6274542f3785c27ab4a38d4db906efd_b.jpg">
data-original="https://pic2.zhimg.com/0930549116d22ffce22e98c32683d621_r.jpg">

```


这是在同一段网页源码测试下的结果，匹配后一种会得到多个相同的url地址，解析起来也更麻烦，这也跟正则写的简单有关系，有兴趣的可以到时候自己修改一下正则表达式，这样下下来的图片也更高清的多。

分析了正则，下面要获取所有的图片该分析Chrome开发者面板的Post数据，因为知乎默认只显示部分回答，我们可以不断往下拉，直到看到这个：

![更多](https://www.tuchuang001.com/images/2018/04/27/more.png)

点击的时候注意观察开发者面板：

![开发者面板](https://www.tuchuang001.com/images/2018/04/27/4cf773a03dfbb51c.png)

简直完美，传递的数据：

```
method:next
params:{"url_token":37709992,"pagesize":10,"offset":30}
```
很眼熟，`url_token`就是问题后面那串数字：

```
https://www.zhihu.com/question/37709992
```

`pagesize`是固定的10，最后一个`offset`偏移量同样很好理解，这里显示10应该说的就是默认显示的10个答案，后面还查看到如下数据：
```
method:next
params:{"url_token":37709992,"pagesize":10,"offset":20}
```

```
method:next
params:{"url_token":37709992,"pagesize":10,"offset":30}
```
也就是说我们在浏览器上每翻过10个答案浏览器就会向服务器发送Post请求在加载十个答案，恩差不多可以开始写代码了。

### 合

模拟登陆之后的操作是找到Post的真实地址模拟浏览器向服务器发送请求：

```python
    url = 'https://www.zhihu.com/node/QuestionAnswerListV2'
    header = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.82 Safari/537.36',
        'Referer': 'https://www.zhihu.com/question/37709992',
        'Origin': 'https://www.zhihu.com',
        'Accept-Encoding': 'gzip, deflate, br',
    }
    data = {
        'method': 'next',
        'params': '{"url_token":' + str(37709992) + ',"pagesize": "10",' + \
                  '"offset":' + str(offset) + "}",
        '_xsrf': getxsrf(),

    }
```

**注意**

发送Post请求时候请加上`'_xsrf': getxsrf()`这一行，否则的话返回的只会是`404 Forbidden`，应该是做了防伪登陆的缘故


然后是写正则，这里发现图片都是被包含在这里面：

```html
<div class="zm-editable-content clearfix">
...
...
</div>
```

所以先匹配到这一大串内容：

```python
        pattern = re.compile('<a class="author-link".*?<span title=.*?<div class="zh-summary.*?' +
                             '<div class="zm-editable-content.*?>(.*?)</div>', re.S)
```

然后在匹配`data-actualsrc`里面的图片链接：

```python
pattern = re.compile('data-actualsrc="(.*?)">', re.S)
```
还有一点要注意的是我们请求之后返回来的是json格式的数据，所以这里还要用到json模块：

```python
 question = session.post(url, headers=header, data=data)
 dic = json.loads(question.content.decode('ISO-8859-1'))
 li = dic['msg'][0]
```
然后对其进行解析：

```python
# 这里使用的是第一个正则表达式
items = re.findall(pattern, li)
# 接下来
items = re.findall(pattern, li)
# 存储图片链接
imagesurl = []
pattern = re.compile('data-actualsrc="(.*?)">', re.S)
for item in items:
    urls = re.findall(pattern, item)
    imagesurl.extend(urls)
```
执行下载操作：

```python
# 存放图片的地址
PWD = "D:/work/python/zhihu/"
        for url in imagesurl:
            myurl = url
            filename = PWD + str(count) + '.jpg'
            if os.path.isfile(filename):
                print("文件存在：", filename)
                count += 1
                continue
            else:
            # 执行下载操作的方法
                downpic(filename, myurl)
                count += 1
                photoNum += 1
            print("一共下载了{0} 张照片".format(photoNum))
            if not os.path.exists(PWD):
                os.makedirs(PWD)
                # 递归调用
        change(offset, count, photoNum)

# downpic方法源码
def downpic(filename, url):
    print("正在下载 " + url)
    try:
        r = requests.get(url, stream=True)
        with open(filename, 'wb') as fd:
            for chunk in r.iter_content():
                fd.write(chunk)
    except Exception as e:
        print("下载失败了", e)
```

**运行结果**

![运行结果](https://www.tuchuang001.com/images/2018/04/27/f4d67b5e4c160a8a.gif)

![结果](https://www.tuchuang001.com/images/2018/04/27/0fe87437efbd1980.png)

这只是一部分，我之前下了四五百张还在下~当然这是后话，感觉现在写的东西都很简单，希望下一次能写出难一点的东西出来。

这里正则部分参考了这里：

> [**通过Python爬虫爬取知乎某个问题下的图片**](http://blog.csdn.net/willib/article/details/51873259)

最后是[**源码**](https://github.com/GiitSmile/downloadpic.py)

源码中注释部分只能下载前十个答案里包含的图片的方法，还有一些想法未完成，本来是想打印一下正在下载哪个答主的回答，然后把图片分别保存到相应的单独文件夹，实现起来有点麻烦就没去搞，仅供参考。

亲测如果需要下载另一个问题的答案，只需要在:
```
 data = {
        'method': 'next',
        'params': '{"url_token":' + str(37709992) + ',"pagesize": "10",' + \
                  '"offset":' + str(offset) + "}",
        '_xsrf': getxsrf(),

    }
```
更换那串数字就行，就好比这样的形式：
> https://www.zhihu.com/question/48720845

但是这种形式的把数字换上去不起效：

> https://www.zhihu.com/question/49078894#answer-41776282

这个好像是知乎热门问答的链接形式，暂时没有深究