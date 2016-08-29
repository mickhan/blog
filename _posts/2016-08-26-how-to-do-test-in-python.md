---
layout: post
title: Python如何做测试？
catalog: true
tags:
    - Python
    - 测试
---

真正开始关注测试这个话题，是从读[《测试驱动开发》][1]开始的。这篇博客的题目打个问号，是因为我自己对测试的理解还很浅薄，只是记录下总结和思考吧。

# unittest

单元测试是测试中最基本的概念。Python中有一个独立的模块[unittest][4]，专门用于单元测试。

这是一个简单的例子。我把它保存成了Sublime Text的snippet，直接基于这个例子来写测试。

{% highlight python %}
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import unittest


class ATestCase(unittest.TestCase):
    def setUp(self):
        pass

    def tearDown(self):
        pass

    def testA(self):
        self.assertEqual(
            1,
            1)

if __name__ == "__main__":
    unittest.main()
{% endhighlight %}

# mock
“单元测试”中的“单元”，是个很有意思的概念。一个不太正确的理解，可以把单元等价为函数。单元测试应该仅限于测试这个函数的功能，它对于外部的依赖和影响需要被完全隔离。那么如果被测函数发起网络请求或读写数据库，要如何隔离呢？这里就要用到mock的手段了。

## mock网络

### 直接使用mock

来自Stackoverflow的问答给了[一个好例子][2]，我稍微改写一下用在自己的代码里。这里的Mail.send调用了requests。mock模拟了request的请求和返回，并且在中间加入一些逻辑判断。这样可以隔离外部的依赖，同时测试到send单元里面的发送逻辑。

{% highlight python %}
import unittest
import mock
from alert import Mail


class MockResponse(object):
    def __init__(self, json_data, status_code):
        self.json_data = json_data
        self.status_code = status_code

    def json(self):
        return self.json_data


def mocked_requests_get(*args, **kwargs):
    if args[0] == "http://xxx/send_mail" and kwargs['data']['subject'] == 'subject':
        return MockResponse({"key1": "value1"}, 200)

    return MockResponse({}, 404)


class AlertTestCase(unittest.TestCase):
    def setUp(self):
        pass

    def tearDown(self):
        pass

    @mock.patch('requests.post', side_effect=mocked_requests_get)
    def testSendMail(self, mock_post):
        mail = Mail('foo@bar.com', 'subject', 'content')
        self.assertEqual(
            mail.send(),
            200)
{% endhighlight %}

### response

专门用于mock requests库。作者写了[一篇文章][3]来举例说明，看起来比使用mock模块更简便。但不确定能否增加自己的逻辑判断，还是只能模拟请求-返回。后面使用过之后再做更新。

## mock 数据库

### mysql

### redis

# nose

nose增加了一些特别方便的功能，可以搜索代码目录下的test文件，自动执行测试。有待进一步研究。

# pytest

flask使用了这个模块来做测试，看起来很简洁，有待研究。

[1]: https://book.douban.com/subject/1230036/ "测试驱动开发"
[2]: http://stackoverflow.com/questions/15753390/python-mock-requests-and-the-response "mocking - python mock Requests and the response - Stack Overflow"
[3]: http://cramer.io/2014/05/20/mocking-requests-with-responses "Mocking Python Requests with Responses - David Cramer's Blog"
[4]: https://docs.python.org/2/library/unittest.html "25.3. unittest — Unit testing framework — Python 2.7.12 documentation"
