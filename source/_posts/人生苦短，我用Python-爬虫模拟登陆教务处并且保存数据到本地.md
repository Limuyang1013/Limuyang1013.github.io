---
title: 人生苦短，我用Python--爬虫模拟登陆教务处并且保存数据到本地
toc: true
date: 2016-08-04 19:26:46
categories: Python
tags: [Python,爬虫,模拟登陆,保存数据]
---

刚开始接触Python，看很多人玩爬虫我也想玩，找来找去发现很多人用网络爬虫干的第一件事就是模拟登陆，增加点难度就是模拟登陆后在获取数据，但是网上好少有Python 3.x的模拟登陆Demo可以参考，加上自己也不怎么懂HTML，所以这第一个Python爬虫写的异常艰难，不过最终结果还是尽如人意的，下面把这次学习的过程整理一下。
<!--more-->
**工具**
- 系统：win7  64位系统
-  浏览器：Chrome
- Python版本：Python 3.5 64-bit
- IDE：JetBrains PyCharm (貌似很多人都用这个)

我把目标瞄准了我们的教务处，这次爬虫的目的是从教务处获取成绩并且把成绩输入Excel表格中保存起来，

我们学校教务处的地址是：http://jwc.ecjtu.jx.cn/ ，往常每次我们获取成绩都需要先进入教务处，然后点击成绩查询，输入公共的账号密码进入，最后输入相关信息获取成绩表格，这里登陆不需要验证码省了我一番功夫，这样我们先进入成绩查询系统登陆界面，先看看怎么模拟登陆这个过程，在Chrome浏览器下按F12打开开发者面板：

![开发者面板](https://www.tuchuang001.com/images/2018/04/27/5213cf10f7b38ed7.png)

这里我们学校的教务处查询系统的密码是公共的jwc也就是拼音缩写，我们输入用户名和密码点击登陆，这时候注意POST请求：

![注意post请求](https://www.tuchuang001.com/images/2018/04/27/89f24d7d4c579e6d.gif)

发现了什么，好像Chrome并没有把Post提交的表单信息保留下来直接跳转到了另一个界面然后展示另一个界面的数据，这里就需要我们自己动手操作一下，注意开发者面板左上角的小红点表示这时候正在抓取数据，如果点击一下就会变成灰色，就可以变相地保存下当时抓取到的包，我在点击登陆后新界面未刷新出来之前点击了这个小红点，如愿以偿的得到了Post的表单数据：

![得到post表单数据](https://www.tuchuang001.com/images/2018/04/27/9b70f0535f02d601.gif)

这样就获取了浏览器在登陆时候向服务器传递的表单数据，看一下这个表单都有些什么：

![查看表单数据](https://www.tuchuang001.com/images/2018/04/27/a65289b9a55a4923.png)

这里看到我们需要传递三个参数，分别是：user、pass、Submit，可以很容易的理解这几个单词的字面意思，这样有了思路，我们就可以写出这次代码的第一步：**模拟登陆教务处**

直接上代码:

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import requests
url = 'http://jwc.ecjtu.jx.cn/mis_o/login.php'
datas = {'user': 'jwc',
         'pass': 'jwc',
         'Submit': '%CC%E1%BD%BB'
         }
headers = {'Referer': 'http://jwc.ecjtu.jx.cn/mis_o/login.htm',
           'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 '
                         '(KHTML, like Gecko) Chrome/52.0.2743.82 Safari/537.36',
           'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
           'Accept-Language': 'zh-CN,zh;q=0.8',
           }
sessions = requests.session()
response = sessions.post(url, headers=headers, data=datas)
print(response.status_code)
```
代码输出：

```
200
```
说明我们模拟登陆成功了，这里用到了Requests模块，还不会使用的可以查看[中文文档](http://docs.python-requests.org/zh_CN/latest/user/quickstart.html#cookie) ，它给自己的定义是：HTTP for Humans，因为简单易用易上手，我们只需要传入Url地址，构造请求头，传入post方法需要的数据，就可以模拟浏览器登陆了，这里因为有进一步获取成绩的操作所以使用了session来保持连接，这里单看最后的返回码的话我们是成功了的，具体如何还要看下一步操作，接下来：

![抓包](https://www.tuchuang001.com/images/2018/04/27/search.png)

这里为了简便代码我们设定输入学号查询所有成绩，减少其他判断，同样对Post数据进行抓包：

![对post数据抓包](https://www.tuchuang001.com/images/2018/04/27/4b81e10138628dc5.png)

同样查看Post的数据：

![查看post数据](https://www.tuchuang001.com/images/2018/04/27/querrypos.png)

因为这里就分析输入学号的情况所以其他都为空，这样我们就可以写出查询成绩的代码：

```python
    score_healders = {'Connection': 'keep-alive',
                      'User - Agent': 'Mozilla/5.0 (Windows NT 6.1; WOW64) '
                                      'AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.82 Safari/537.36',
                      'Content - Type': 'application / x - www - form - urlencoded',
                      'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                      'Content - Length': '69',
                      'Host': 'jwc.ecjtu.jx.cn',
                      'Referer': 'http: // jwc.ecjtu.jx.cn / mis_o / main.php',
                      'Upgrade - Insecure - Requests': '1',
                      'Accept - Language': 'zh - CN, zh;q = 0.8'
                      }
    score_url = 'http://jwc.ecjtu.jx.cn/mis_o/query.php?start=' + str(
        pagenum) + '&job=see&=&Name=&Course=&ClassID=&Term=&StuID=' + num
    score_data = {'Name': '',
                  'StuID': num,
                  'Course': '',
                  'Term': '',
                  'ClassID': '',
                  'Submit': '%B2%E9%D1%AF'
                  }

    score_response = sessions.post(score_url, data=score_data, headers=score_healders)
    content = score_response.content
```

这里解释一下上面的代码，上面的`score_url` 并不是浏览器上显示的地址，我们要获取真正的地址，在Chrome下右键--查看网页源代码，找到这么一行：

```vbscript-html
a href=query.php?start=1&job=see&=&Name=&Course=&ClassID=&Term=&StuID=xxxxxxx
```
这个才是真正的地址，点击这个地址转入的才是真正的界面，因为这里成绩数据较多，所以这里采用了分页显示，这个`start=1`说明是第一页，这个参数是可变的需要我们传入，还有`StuID`后面的是我们输入的学号，这样我们就可以拼接出Url地址：

```
score_url = 'http://jwc.ecjtu.jx.cn/mis_o/query.php?start=' + str(pagenum) + '&job=see&=&Name=&Course=&ClassID=&Term=&StuID=' + num
```
同样使用Post方法传递数据并获取响应的内容：

```python
score_response = sessions.post(score_url, data=score_data,headers=score_healders)
content = score_response.content
```
这里采用[Beautiful Soup 4.2.0](https://www.crummy.com/software/BeautifulSoup/bs4/doc/index.zh.html)来解析返回的响应内容，因为我们要获取的是成绩，这里到教务处成绩查询界面，查看获取到的成绩在网页中是以表格的形式存在：

![网页源代码](https://www.tuchuang001.com/images/2018/04/27/qr.png)
观察表格的网页源代码：

```html
<table align=center border=1>
<tr><td bgcolor=009999>学期</td>
<td bgcolor=009999>学号</td>
<td bgcolor=009999>姓名</td>
<td bgcolor=009999>课程</td>
<td bgcolor=009999>课程要求</td>
<td bgcolor=009999>学分</td>
<td bgcolor=009999>成绩</td>
<td bgcolor=009999>重考一</td>
<td bgcolor=009999>重考二</td></tr>
...
...
</tr></table>
```
这里拿出第一行举例，虽然我不太懂Html但是从这里可以看出来`<tr>` 代表的是一行，而`<td>`应该是代表这一行中的每一列，这样就好办了，取出每一行然后分解出每一列，打印输出就可以得到我们要的结果：

```Python
from bs4 import BeautifulSoup
soup = BeautifulSoup(content, 'html.parser')
# 找到每一行
target = soup.findAll('tr')
```
这里分解每一列的时候要小心，因为这里表格分成了三页显示，每页最多显示30条数据，这里因为只是收集已经毕业的学生的成绩数据所以不对其他数据量不足的学生成绩的情况做统计，默认收集的都是大四毕业的学生成绩数据。这里采用两个变量`i`和`j`分别代表行和列：

```python
# 注:这里的print单纯是我为了验证结果打印在PyCharm的控制台上而已
i=0, j=0
for tag in target[1:]:
            tds = tag.findAll('td')
            # 每一次都是从列头开始获取
            j = 0
            # 学期
            semester = str(tds[0].string)
            if semester == 'None':
                break
            else:
                print(semester.ljust(6) + '\t\t\t', end='')
            # 学号
            studentid = tds[1].string
            print(studentid.ljust(14) + '\t\t\t', end='')
            j += 1
            # 姓名
            name = tds[2].string
            print(name.ljust(3) + '\t\t\t', end='')
            j += 1
            # 课程
            course = tds[3].string
            print(course.ljust(20, ' ') + '\t\t\t', end='')
            j += 1
            # 课程要求
            requirments = tds[4].string
            print(requirments.ljust(10, ' ') + '\t\t', end='')
            j += 1
            # 学分
            scredit = tds[5].string
            print(scredit.ljust(2, ' ') + '\t\t', end='')
            j += 1
            # 成绩
            achievement = tds[6].string
            print(achievement.ljust(2) + '\t\t', end='')
            j += 1
            # 重考一
            reexaminef = tds[7].string
            print(reexaminef.ljust(2) + '\t\t', end='')
            j += 1
            # 重考二
            reexamines = tds[8].string
            print(reexamines.ljust(2) + '\t\t')
            j += 1
            i += 1
```
这里查了很多别人的博客都是用正则表达式来分解数据，表示自己的正则写的并不好也尝试了但是没成功，所以无奈选择这种方式，如果有人有测试成功的正则欢迎跟我说一声，我也学习学习。

**把数据保存到Excel**

因为已经清楚了这个网页保存成绩的具体结构，所以顺着每次循环解析将数据不断加以保存就是了，这里使用`xlwt`写入数据到Excel，因为`xlwt`模块打印输出到Excel中的样式宽度偏小，影响观看，所以这里还加入了一个方法去控制打印到Excel表格中的样式:

```python
file = xlwt.Workbook(encoding='utf-8')
table = file.add_sheet('achieve')
# 设置Excel样式
def set_style(name, height, bold=False):
    style = xlwt.XFStyle()  # 初始化样式
    font = xlwt.Font()  # 为样式创建字体
    font.name = name  # 'Times New Roman'
    font.bold = bold
    font.color_index = 4
    font.height = height
    style.font = font
    return style
```
运用到代码中：

```
for tag in target[1:]:
            tds = tag.findAll('td')
            j = 0
            # 学期
            semester = str(tds[0].string)
            if semester == 'None':
                break
            else:
                print(semester.ljust(6) + '\t\t\t', end='')
                table.write(i, j, semester, set_style('Arial', 220))
            # 学号
            studentid = tds[1].string
            print(studentid.ljust(14) + '\t\t\t', end='')
            j += 1
            table.write(i, j, studentid, set_style('Arial', 220))
            table.col(i).width = 256 * 16
            # 姓名
            name = tds[2].string
            print(name.ljust(3) + '\t\t\t', end='')
            j += 1
            table.write(i, j, name, set_style('Arial', 220))
            # 课程
            course = tds[3].string
            print(course.ljust(20, ' ') + '\t\t\t', end='')
            j += 1
            table.write(i, j, course, set_style('Arial', 220))
            # 课程要求
            requirments = tds[4].string
            print(requirments.ljust(10, ' ') + '\t\t', end='')
            j += 1
            table.write(i, j, requirments, set_style('Arial', 220))
            # 学分
            scredit = tds[5].string
            print(scredit.ljust(2, ' ') + '\t\t', end='')
            j += 1
            table.write(i, j, scredit, set_style('Arial', 220))
            # 成绩
            achievement = tds[6].string
            print(achievement.ljust(2) + '\t\t', end='')
            j += 1
            table.write(i, j, achievement, set_style('Arial', 220))
            # 重考一
            reexaminef = tds[7].string
            print(reexaminef.ljust(2) + '\t\t', end='')
            j += 1
            table.write(i, j, reexaminef, set_style('Arial', 220))
            # 重考二
            reexamines = tds[8].string
            print(reexamines.ljust(2) + '\t\t')
            j += 1
            table.write(i, j, reexamines, set_style('Arial', 220))
            i += 1

file.save('demo.xls')
```

最后稍加整合，写成一个方法：

```
# 获取成绩
# 这里num代表输入的学号，pagenum代表页数，总共76条数据，一页30条所以总共有三页
def getScore(num, pagenum, i, j):
    score_healders = {'Connection': 'keep-alive',
                      'User - Agent': 'Mozilla/5.0 (Windows NT 6.1; WOW64) '
                                      'AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.82 Safari/537.36',
                      'Content - Type': 'application / x - www - form - urlencoded',
                      'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                      'Content - Length': '69',
                      'Host': 'jwc.ecjtu.jx.cn',
                      'Referer': 'http: // jwc.ecjtu.jx.cn / mis_o / main.php',
                      'Upgrade - Insecure - Requests': '1',
                      'Accept - Language': 'zh - CN, zh;q = 0.8'
                      }
    score_url = 'http://jwc.ecjtu.jx.cn/mis_o/query.php?start=' + str(
        pagenum) + '&job=see&=&Name=&Course=&ClassID=&Term=&StuID=' + num
    score_data = {'Name': '',
                  'StuID': num,
                  'Course': '',
                  'Term': '',
                  'ClassID': '',
                  'Submit': '%B2%E9%D1%AF'
                  }

    score_response = sessions.post(score_url, data=score_data, headers=score_healders)
    # 输出到文本
    with open('text.txt', 'wb') as f:
        f.write(score_response.content)
    content = score_response.content
    soup = BeautifulSoup(content, 'html.parser')
    target = soup.findAll('tr')
    try:
        for tag in target[1:]:
            tds = tag.findAll('td')
            j = 0
            # 学期
            semester = str(tds[0].string)
            if semester == 'None':
                break
            else:
                print(semester.ljust(6) + '\t\t\t', end='')
                table.write(i, j, semester, set_style('Arial', 220))
            # 学号
            studentid = tds[1].string
            print(studentid.ljust(14) + '\t\t\t', end='')
            j += 1
            table.write(i, j, studentid, set_style('Arial', 220))
            table.col(i).width = 256 * 16
            # 姓名
            name = tds[2].string
            print(name.ljust(3) + '\t\t\t', end='')
            j += 1
            table.write(i, j, name, set_style('Arial', 220))
            # 课程
            course = tds[3].string
            print(course.ljust(20, ' ') + '\t\t\t', end='')
            j += 1
            table.write(i, j, course, set_style('Arial', 220))
            # 课程要求
            requirments = tds[4].string
            print(requirments.ljust(10, ' ') + '\t\t', end='')
            j += 1
            table.write(i, j, requirments, set_style('Arial', 220))
            # 学分
            scredit = tds[5].string
            print(scredit.ljust(2, ' ') + '\t\t', end='')
            j += 1
            table.write(i, j, scredit, set_style('Arial', 220))
            # 成绩
            achievement = tds[6].string
            print(achievement.ljust(2) + '\t\t', end='')
            j += 1
            table.write(i, j, achievement, set_style('Arial', 220))
            # 重考一
            reexaminef = tds[7].string
            print(reexaminef.ljust(2) + '\t\t', end='')
            j += 1
            table.write(i, j, reexaminef, set_style('Arial', 220))
            # 重考二
            reexamines = tds[8].string
            print(reexamines.ljust(2) + '\t\t')
            j += 1
            table.write(i, j, reexamines, set_style('Arial', 220))
            i += 1
    except:
        print('出了一点小Bug')
    file.save('demo.xls')
```
在模拟登陆操作后增加一个判断：

```
# 判断是否登陆
def isLogin(num):
    return_code = response.status_code
    if return_code == 200:
        if re.match(r"^\d{14}$", num):
            print('请稍等')
        else:
            print('请输入正确的学号')
        return True
    else:
        return False
```

最后在`__main__`中这么调用：

```
if __name__ == '__main__':
    num = input('请输入你的学号：')
    if isLogin(num):
        getScore(num, pagenum=0, i=0, j=0)
        getScore(num, pagenum=1, i=31, j=0)
        getScore(num, pagenum=2, i=61, j=0)
```

在PyCharm下按`alt`+`shift`+`x`快捷键运行程序：

![控制台输出](https://www.tuchuang001.com/images/2018/04/27/inputnum.png)

控制台会有如下输出(这里只截取部分，不要吐槽没有对齐，这里我也用了格式化输出还是不太行，不过最起码出来了结果，而且我们的目的是输出到Excel中不是吗)

![控制台输出](https://www.tuchuang001.com/images/2018/04/27/achievre.png)

然后去程序根目录找看看有没有生成一个叫`demo.xls`的文件，我的程序就放在桌面，所以去桌面找：

![桌面图标](https://www.tuchuang001.com/images/2018/04/27/rexls.png)

点开查看是否成功获取：

![最终获取结果](https://www.tuchuang001.com/images/2018/04/27/finalresult.png)

至此，大功告成

**小结**
刚开始接触Python一个星期的样子，这次写了这么一个简单的网络爬虫检验一下学习成果，虽然程序还有些许Bug，不过总归得到了一定收获，当然也为下一步学习打下了基础，嗯哼，为了接下来批量获取网络上美女图片并分类保存我会继续自学Python，荆轲刺秦王~

[源码](https://github.com/GiitSmile/login/blob/master/login.py)
