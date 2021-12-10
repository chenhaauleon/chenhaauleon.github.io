---
layout:        post
title:         "禅道 | Python3 禅道管理类"
subtitle:      "定时任务统计禅道 bug 并发送钉钉消息"
author:        "Haauleon"
header-style:  text
catalog:       true
tags:
    - Zentao
    - Python
---

## 背景
&emsp;&emsp;测试人员在项目测试阶段时会输出 bug 问题单，有的公司用 jira ，有的公司用禅道。目前所在公司用的是禅道，禅道作为测试人员和开发人员之间沟通的桥梁，它起着至关重要的作用。虽然在项目测试阶段，项目组会集中关注上线的情况，但是禅道的问题单创建成功后，需要开发人员主动登录主页进行查找，需要花费一些时间，若错过了一些紧急重要的 bug，也会影响整个测试周期。除此之外，部分公司有每日 bug 统计的需求，统计出当前测试阶段/周期中产生的 bug 有多少？未解决的多少？无法重现的有多少？等等。所以，目前在使用过程中，遇到了以下问题：           

&emsp;&emsp;问题一，禅道创建的 bug 问题单无法即时通知开发人员。         

&emsp;&emsp;问题二，测试管理者需要快速知道当前测试阶段的 bug 统计情况。     

<br><br>

## 需求
&emsp;&emsp;作为测试人员，我希望自己在禅道上创建 bug 后可以通过钉钉项目群立即通知到项目组的开发人员。通知样式如下：                 
```
Hey！最新 Bug 提醒！
ID: 5462
状态: 激活
指派给: 陈巧伦
创建日期: 12-08 10:56
Bug标题: 【saas 积分商城】用户购买积分商城商品，订单确认页面使用购物金抵扣的金额显示不正确

@所有人
```   

&emsp;&emsp;作为测试管理，我希望自己每天下班前都可以通过钉钉项目群快速了解测试周期中产生的 bug 的处理情况。通知样式如下：                   
```
Hey！今日 Bug 统计结果！
(积分商城应用)

未解决: 2
设计如此: 0
重复Bug: 0
外部原因: 0
已解决: 12
无法重现: 1
延期处理: 0
不予解决: 0
转为需求: 0
有效率: 92.31%
总计: 15

@所有人
```

<br><br>

## 代码
&emsp;&emsp;写一个禅道服务类，完成以下功能：创建问题单、问题单的查询和统计、推送最新 bug。      

<br>

###### 日志配置
```python
import sys
import logging
from colorama import Fore, Style

_logger = logging.getLogger('zentao')           # 获取日志记录器
_logger.setLevel(logging.DEBUG)                 # 设置日志等级
_handler = logging.StreamHandler(sys.stdout)    # 输入到控制台的 handler
_logger.addHandler(_handler)   


def info(msg):
    '''日志信息'''
    now = time.strftime("%Y-%m-%d %H:%M:%S")
    _logger.info(Fore.GREEN + now + " [INFO] " + str(msg) + Style.RESET_ALL)

def error(msg):
    '''异常信息'''
    now = time.strftime("%Y-%m-%d %H:%M:%S")
    _logger.error(Fore.RED + now + " [ERROR] " + str(msg) + Style.RESET_ALL)

def _print(msg):
    '''打印信息'''
    now = time.strftime("%Y-%m-%d %H:%M:%S")
    _logger.debug(Fore.BLUE + now + " [PRINT] " + str(msg) + Style.RESET_ALL)
```

<br><br>

###### 装饰异常
```python
from functools import wraps


def traceback_error(func):
    @wraps(func)
    def wraper(self, *args, **kwargs):
        try:
            result = func(self, *args, **kwargs)
        except Exception as e:
            import traceback
            ex_msg = '{exception}'.format(exception=traceback.format_exc())
            print(ex_msg)
            result = ex_msg
        return result
    return wraper
```

<br><br>

###### 钉钉发送类
&emsp;&emsp;需要注册一个钉钉账号，登录后创建一个项目组，可以是公司内部群，也可以是拉人建群，都可以。然后在群内添加群机器人，拿到 token 值。         

```python
import requests
import json


class DingDingNotice:
    '''钉钉发送类'''

    def __init__(self, ding_token=None, atMobiles=None, isAtAll=None):
        # 根据电话@用户
        self.atMobiles = ['13046367204',] if atMobiles==None else atMobiles
        # self.token = 'cbb3b771657ef' if ding_token==None else ding_token
        # 是否@所有人
        self.isAtAll = True if isAtAll==None else isAtAll
        self.token = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
        self.api = 'https://oapi.dingtalk.com/robot/send?access_token={}'.format(self.token)
        self.headers = {'Content-Type': 'application/json;charset=utf-8'}

    @traceback_error
    def send_msg(self,content):
        msg = {
            'msgtype': 'text',
            'text': {'content': content},
            'at': {'atMobiles': self.atMobiles, 'isAtAll': self.isAtAll}
        }
        data = requests.post(self.api, data=json.dumps(msg), headers=self.headers).json()
        return json.dumps(data)
```

<br><br>

###### 禅道服务类
**1. 用户登录**              

```python
import requests
import re
import time
import hashlib
import json
from lxml import etree
import datetime
import warnings
warnings.filterwarnings("ignore")


username = "username"      # 禅道登录账号
pw = "password"            # 禅道登录密码
base = "http://xx.xx.com"  # 服务器的域名

user = requests.Session()
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.106 Safari/537.36"
}


class Zentao:

    def __init__(self):
        self.login_url = "%s/zentao/user-login.html" %base
        self.product_url = "%s/zentao/product-index-no.json" %base
        self.rand = None
        self.pwd = None
        self.product_id = None
        self.modules_dict = None

    def union_option(self):
        self.get_rand()
        self.get_password()
        self.user_login()
        self.check_login()

    def get_rand(self):
        '''获取 rand 值'''
        while True:
            res_rand = user.get(self.login_url, headers=headers)
            # res_rand.encoding = 'utf-8'
            # print("res_rand.text = %s" %res_rand.text)
            rand = re.findall(r"'verifyRand' value='(.+?)'", res_rand.text)
            # print("rand[0] = {}".format(rand[0]))
            if len(rand[0]) == 10:   # rand 的长度不固定可为9或10，判断长度为10时就不再请求登录页面接口
                self.rand = rand[0]
                break
        _print(self.rand)

    def get_password(self):
        '''获取 password'''
        # 方式一
        hash = hashlib.md5()
        hash.update(pw.encode('utf-8'))
        f = hash.hexdigest() + self.rand
        # print("f = %s" %f)
        # 方式二
        hash2 = hashlib.md5(f.encode('utf-8'))
        self.pwd = hash2.hexdigest()
        _print("pwd = %s" %self.pwd)

    def user_login(self):
        '''用户登录'''
        data = {
            "account": username,
            "password": self.pwd,
            "referer": "",
            "verifyRand": self.rand
        }
        res_login = user.post(self.login_url, headers=headers, data=data)
        # res_login.encoding = 'utf-8'
        # print("res_login.text = %s" %res_login.text)

    def check_login(self):
        '''检查登录'''
        res_check = user.get("%s/zentao/bug-browse-%s-0-openedbyme.html" %(base, self.product_id), headers=headers)
        # res_check.encoding = 'utf-8'
        # print("res_check.text = %s" %res_check.text)
        result = re.findall(r"\<a href=\'\/zentao\/user-logout.html' \>(.+?)\<\/a\>", res_check.text)
        if result[0] == "退出":
            _print("登录成功")


if __name__ == '__main__':
    _print("脚本开始执行")
    zentao = Zentao()
    zentao.union_option()
    _print("脚本执行完成")
```

<br><br>

**2. 创建问题单**      
&emsp;&emsp;创建 bug 单功能。作为禅道的常用功能，当然是不能少的了，在脚本里可用作一个自动化测试的辅助工具，若自动化测试出现 bug 则自动创建 bug 单。这是目前要实现的一个方向。          

```python
...
...
...

class Zentao:

    ...
    ...
    ...

    def union_option(self):
        self.get_rand()
        self.get_password()
        self.user_login()
        self.get_product_list("saas")
        self.check_login()
        self.get_module_list()

    def search_for_key(self, dict_value, dict_name):
        '''根据字典的值查找键'''
        return list(filter(lambda k:dict_name[k] == dict_value, dict_name))[0]

    def get_product_list(self, product_name):
        '''获取产品列表'''
        res_product = user.get(self.product_url).json()
        products_str = res_product["data"].encode('utf-8').decode('unicode_escape') # python3 取消了decode，要想str中的unicode转中文需要先编码再解码
        products_dict = json.loads(products_str)["products"]  # str 转 dict
        self.product_id = self.search_for_key(product_name, products_dict)

    def get_module_list(self):
        '''获取各模块列表'''
        res_module = user.get("%s/zentao/bug-create-%s-0-moduleID=0.json" %(base, self.product_id)).json()
        modules_str = res_module["data"].encode('utf-8').decode('unicode_escape')
        self.modules_dict = json.loads(modules_str)

    def get_project_id(self, project_name):
        '''获取项目id'''
        project_id = self.search_for_key(project_name, self.modules_dict["projects"])
        return project_id

    def get_version(self, version_name):
        '''获取影响版本'''
        version_key = self.search_for_key(version_name, self.modules_dict["builds"])
        return version_key

    def get_member(self, member_name):
        '''获取指派成员'''
        member_key = self.search_for_key(member_name, self.modules_dict["projectMembers"])
        return member_key

    def create_bug(self):
        '''创建一个 bug 问题单'''
        data = {
            "product": self.product_id,
            "module": 0,
            "project": self.get_project_id("saas"),
            "openedBuild[]": self.get_version("主干"),
            "assignedTo": self.get_member("C:陈巧伦"),
            "deadline": "",
            "type": "automation",
            "os": "",
            "browser": "",
            "title": "[测试脚本]测试禅道自动发布bug问题单",
            "color": "",
            "severity": "3",
            "pri": "3",
            "steps": "<p>[步骤]</p><p>1</p><p><br /></p><p>[结果]</p><p>2</p><p><br /></p><p>[期望]</p>3",
            "story": "",
            "task": "",
            "oldTaskID": "0",
            "mailto[]": "", 
            "keywords": "",
            "labels[]": "",
            "files[]": "",
            "case": "0",
            "caseVersion": "0",
            "result": "0",
            "testtask": "0"
        }
        res_bug = user.post("%s/zentao/bug-create-%s-0-moduleID=0.html" %(base, self.product_id), data=data)


if __name__ == '__main__':
    _print("脚本开始执行")
    zentao = Zentao()
    zentao.union_option()
    zentao.create_bug()
    _print("脚本执行完成")
```

<br><br>

**3. 问题单统计**     
&emsp;&emsp;获取 bug 统计页面的数据。先请求禅道统计 > 测试页面，解析测试 bug 统计结果，将结果进行二次处理后发送至顶顶群。此功能是可以为测试管理者提供每日 bug 自动统计，避免人工去统计而出错。需要传入统计开始时间，结束时间写死为今天，后期如果有传结束时间的需求再优化即可。可单独作为一个执行脚本放在服务器上定时执行，比如每天下午 6 点执行一次。

```python
...
...
...

class Zentao:

    ...
    ...
    ...

    def get_bug_report(self, start_time, tips):
        '''获取每日bug统计（适用于测试经理下班前统计）'''
        res_bug_report = user.get("%s/zentao/report-bugcreate-%s-%s-%s-%s.html" %(base, start_time, datetime.datetime.now().strftime('%Y%m%d'), str(self.product_id), str(self.get_project_id(("saas")))))
        html = res_bug_report.text
        dom = etree.HTML(html)
        # 获取行首标题列表
        cols_head = dom.xpath('//*[@id="bug"]/thead/tr/th[position()>0][position()<13]/text()')
        # 获取//*[@id="bug"]/tbody下有多少个tr标签，计算tr标签列表长度
        rows_count = len(dom.xpath('//*[@id="bug"]/tbody//tr'))
        report_msg_list = []
        for i in range(1, rows_count+1):
            # 获取每一行的值
            row_value = dom.xpath('//*[@id="bug"]/tbody/tr[%s]/td[position()>0 and position()<13]/text()'%str(i))
            # 与行首合并成字典
            single_msg_dict = dict(map(lambda x,y:[x,y], cols_head, row_value))
            # 用列表去追加字典元素
            report_msg_list.append(single_msg_dict)

        report_msg = ""
        for msg in report_msg_list:
            for key,value in msg.items():
                # 将字典解开合成字符串格式返回
                report_msg = report_msg + key + ": " + value + "\n"
            report_msg += "\n"
        return "Hey！今日Bug跟踪结果！\n(%s)\n\n"%tips + report_msg


def run_bug_statistics_export():
    '''每日bug统计执行并发送消息（一般适用于测试经理在下班时进行当日统计）'''
    zentao = Zentao()
    zentao.union_option()
    dingding_msg = zentao.get_bug_report(20211201, "积分商城应用")
    if dingding_msg:
        # print(dingding_msg)
        dingding = DingDingNotice()
        _print(dingding.send_msg(content=dingding_msg))
        _print("钉钉消息发送成功")

if __name__ == '__main__':
    _print("脚本开始执行")
    zentao = Zentao()
    zentao.union_option()
    run_bug_statistics_export()
    _print("脚本执行完成")
```

<br><br>

**4. 推送最新 bug**     
&emsp;&emsp;获取 bug 列表的最新一条。实现的原理是：每分钟都请求 bug 列表接口，然后解析 bug ID 的值，然后打开本地 bug_list.json 文件并读取 bug_ids 的字段值，如果此 bug ID 存在于该字段值中则不需要发送钉钉消息，若不存在则将此 bug ID 追加到此字段值中并发送钉钉消息。此功能要达到的效果是一有新 bug 产生就通知开发人员，通常用于测试人员创建 bug 后及时反馈至开发人员处理。可单独作为一个执行脚本放在服务器上定时执行，比如每分钟执行一次。注意：执行此方法前需要创建一个 bug_list.json 文件，并写入内容 `{"bug_ids": []}` 。       

```python
...
...
...

class Zentao:

    ...
    ...
    ...

    def send_new_bug(self):
        '''根据id进行bug的查找，并发送一条最新bug至钉钉（适用于测试人员提醒开发人员）'''
        res_bug_list = user.get("%s/zentao/bug-browse-%s-0-openedbyme.html" %(base, str(self.product_id)))
        html = res_bug_list.text
        dom = etree.HTML(html)
        # 获取 Bug ID
        bug_id = dom.xpath('//*[@id="bugList"]/tbody/tr[1]/td[1]/div/label/text()')[0]
        # 获取 Bug 标题
        bug_titile = dom.xpath('//*[@id="bugList"]/tbody/tr[1]/td[3]/a/text()')[0]
        # 获取 Bug 状态
        bug_status = dom.xpath('//*[@id="bugList"]/tbody/tr[1]/td[5]/span/text()')[0]
        # 获取 Bug 创建时间
        bug_create = dom.xpath('//*[@id="bugList"]/tbody/tr[1]/td[6]/text()')[0]
        # 获取 Bug 指派人
        bug_fix = dom.xpath('//*[@id="bugList"]/tbody/tr[1]/td[9]/a/span/text()')[0]
        # 读取本地 bug_list.json 文件，将字符串变为数据类型
        with open("./bug_list.json", "r") as load_f:
            load_dict = json.load(load_f)
            # print(load_dict)
        # 判断当前 bug id 是否存在于文件中，如果没有就追加并发送钉钉消息
        if bug_id not in load_dict["bug_ids"]:
            load_dict["bug_ids"].append(bug_id)
            with open("./bug_list.json", "w") as dump_f:
                json.dump(load_dict, dump_f)
            return "Hey！最新 Bug 提醒！\nID: {}\n指派给: {}\n创建日期: {}\n状态: {}\nBug标题: {}\n".format(bug_id,bug_fix,bug_create,bug_status,bug_titile)


def run_new_bug_create():
    '''定时任务发送一条最新bug至钉钉（适用于测试人员提醒开发人员）'''
    zentao = Zentao()
    zentao.union_option()
    dingding_msg = zentao.send_new_bug()
    if dingding_msg:
        # print(dingding_msg)
        dingding = DingDingNotice()
        _print(dingding.send_msg(content=dingding_msg))
        _print("钉钉消息发送成功")


if __name__ == '__main__':
    _print("脚本开始执行")
    zentao = Zentao()
    zentao.union_option()
    run_new_bug_create()
    _print("脚本执行完成")
```

<br><br>

## TODO
- 调研使用哪种自动化测试执行策略才可以跟禅道创建问题单功能进行完美糅合
- 调研最新问题单推送是否要判断优先级，是要@指派人还是@所有人