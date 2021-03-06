---
title: 【Day 4】关于我是如何干掉Appium和RobotFramework这件事的——PO有向图
date: 2021-06-28 18:00:00 +/-TTTT
categories: [客户端自动化,框架设计]
tags: [python,自动化,iOS,Android,Appium]
typora-copy-images-to: ../assets/img
typora-root-url: ../../vancheung.github.io
---

### 一、问题背景

先写一个简单的登录Action

```python
class UiClient:
    def __init__(self):
        self.client = wda.client()

    def login(self,username,password):
        self.client.input(帐号输入框, username)
        self.client.input(密码输入框, password)
        self.client.click(登录)

if __name__ == '__main__':
    ui = UiClient()
    ui.login('张三', 123456)
```

这样的登录操作其实隐含了一个前提：当前已经在登录页。

而在实际测试过程中，没办法保证前一次测试的环境清理成功完成，因此也引入了如下状态：

（1）已登录状态，打开APP时第一个页面为行情页

（2）首次打开APP+未登录，为“选择登录方式”的登录页

（3）非首次打开APP+未登录，为“可输入账号密码”的登录页

（4）上一条测试用例环境清理失败，停留在任意页面

简化版的流程如下（实际情况更复杂）：

```python
def login(username,password):
    if not self.is_login_page():        
        if is_page_xxx:
            close_page_xxx()
        if is_page_yyy:
            close_page_yyy()    
        self.click('主页')
        self.click('我的页') # 简化流程
        if self.client.find_element('帐号')            
            self.click('设置')
            self.click('退出登录')
    self.client.input(帐号输入框, username)
    self.client.input(密码输入框, password)
    self.client.click(登录)
```

流程图大致如下：

![image-20210628185513874](/assets/img/image-20210628185513874.png)

### 二、优化方案一

上述流程会引入多个问题，例如违背单一职责原则、调用链长易出错等。因此，做出第一版优化方案，主要针对方法的抽象。

1、抽象 “进入登录页 ”部分，在prepare_login_page中实现。

```python
def login_by_account(self,username,password):
    self.prepare_login_page()
    self.client.input(帐号输入框, username)
    self.client.input(密码输入框, password)
    self.client.click(登录)

def prepare_login_page(self):        
    if self.is_login_page():
        if '首次登录':
            self.client.click('使用帐号登录')
        return
    if is_page_xxx:
        close_page_xxx()
    if is_page_yyy:
        close_page_yyy()    
    self.click('主页')
    self.click('我的页') # 简化流程
    if self.client.find_element('帐号')            
        self.click('设置')
        self.click('退出登录')
```

2、抽象 “其他页返回主页” ，在return_main_page()方法中实现，主页->我的页->退出登录（可选）->登录页

```python
def prepare_login_page(self):        
    if self.is_login_page():
        if '首次登录':
            self.client.click('使用帐号登录')
        return
    self.return_main_page()
    self.click('我的页') # 简化流程
    if self.client.find_element('帐号')            
        self.click('设置')
        self.click('退出登录')        

def return_main_page():
    if is_page_xxx:
        close_page_xxx()
    if is_page_yyy:
        close_page_yyy()    
    self.click('主页') # 简化流程
```

但这种抽象结构仍未解决以下问题：

（1）随着编写用例涉及到的页面增多，return_main_page方法会持续膨胀；

（2）调用链会变得很长： 

[is_xxx_page -> 对应操作]*n -> 主页 -> 我的页 -> (设置页->退出登录) -> 登录页->登录操作 

（3）判断页面时存在先后顺序的问题，例如yyy页面出现在xxx页面之前，则无法正常关闭xxx页面。

因此，需要尝试进一步优化的方案。



### 三、优化方案二

[有向图](https://baike.baidu.com/item/有向图) 是一个经典的数据结构，如果把页面抽象成图的节点，页面跳转关系抽象为图的边，可以直接跳转的页面之间边的权重(weight)为1。则任意一个页面A到B的跳转路径，可以简化为：求节点A到节点B的最短路径。

![image-20210628185721549](/assets/img/image-20210628185721549.png)

例如下图，A->E的最短路径为A->C->E。求得最短路径之后，只需要按记录的Action列表依次操作，即可完成页面跳转。

![image-20210629162728965](/assets/img/image-20210629162728965.png)

1、为实现这样的数据结构，首先定义一个BasePage，作为页面类的基类。

BasePage具有以下特性：

（1）BasePage是抽象类，无法直接被实例化，其他PageObject都需要继承BasePage；

（2）BasePage定义一组必须实现的接口（抽象方法），如is_page() 等；

（3）通常BasePage的子类应该是单例，即不能重复初始化同一个页面。

```python
class BasePage:
    __metaclass__ = ABCMeta

    def __init__(self, ui_client: UiClient):
        self.client = ui_client

    def __new__(cls, *args, **kwargs):
        """
        单例
        """
        if not hasattr(cls, 'instance'):
            cls.instance = super(BasePage, cls).__new__(cls)
        return cls.instance

    @abstractmethod
    def is_page(self):
        pass
```

2、定义PageTree，管理有向图相关的操作。在此使用networkx库的有向图DiGraph()。

PageTree具有以下特性：

（1）包含一个networkx.DiGraph()对象；

（2）add_node方法，默认传入的节点名为str类型；支持传入其他属性；

（3）add_turn方法，传入两个节点和一个跳转函数，为这两个节点添加一条边，默认weight为1；

（4）get_func方法，获取两个节点间的最短路径，返回一个函数列表，列表内容为：这个最短路径中，每条边对应的函数

（5）change_weight方法：修改一条边的权重。用于某些操作后，页面跳转关系发生变化的情况。

```python
class PageTree:
    def __init__(self):
        self.G = networkx.DiGraph()

    def add_node(self, name, **kwargs):
        self.G.add_node(name, **kwargs)

    def add_turn(self, v, w, func, *args, **kwargs):
        if not func:
            func = 'wait'
        if not self.G.has_node(v):
            self.G.add_node(v)
        if not self.G.has_node(w):
            self.G.add_node(w)
        self.G.add_edges_from([(v, w)], page_name=v, weight=1, func=func, args=args, kwargs=kwargs)

    def get_func(self, u, v):
        result = []
        path = networkx.shortest_path(self.G, source=u, target=v)

        def _get_neighbor_func(u, v):
            result.append(self.G.get_edge_data(u, v))
            return v
        reduce(_get_neighbor_func, path)
        return result

    def change_weight(self, u, v, weight):
        self.G[u][v]['weight'] = weight
```

3、定义一个Navigation类，管理所有页面跳转关系。

Navigation具有以下特性：

（1）导入所有PageObject；

（2）_get_page：根据页面名称查找对应的PageObject；

（3）handle_page：获取当前页。需要遍历PageObject的is_page()方法，page_map顺序对此会有影响；

（4）navigate_to_page：页面跳转。依次执行PageTree.get_func中获取到的函数。

```python
class Navigation:
    page_map = {
        '我的页': mine.MinePage,
        '登录页': login.LoginPage,
        '设置页': setting.SettingPage,
        'XXX页': xxx.XXXPage,
        'YYY页': yyy.YYYPage,
        '主页': main.MainPage
    }

    def __init__(self, client):
        self.handle = None
        self.page_tree = PageTree()
        for page_name in self.page_map:
            setattr(self, page_name, self._get_page(page_name)(client))
            self.page_tree.add_node(page_name, instance=self._get_page(page_name))
        self._set_page_tree()

    def _get_page(self, page_name):
        if hasattr(self, page_name):
            return getattr(self, page_name)
        return self.page_map[page_name]

    def handle_page(self):
        if self.handle and self.handle.is_page():
            return self.handle
        for page in self.page_map:
            if getattr(self, page).is_page():
                self.handle = getattr(self, page)
                return self.handle

    def navigate_to_page(self, to_page_name: str):
        self.handle = self.handle_page()
        to_page = self._get_page(to_page_name)
        if self.handle is to_page:  # 都是单例，可以通过比较内存地址判断
            logger.info('当前在[%s]页面，无需跳转', to_page_name)
            return False

        func_map = self.page_tree.get_func(str(self.handle), to_page_name)
        for f in func_map:
            args = f.get('args')
            kwargs = f.get('kwargs')
            func = f.get('func')
            func(*args, **kwargs)
        return True
```

4、定义初始化页面跳转关系的操作：

```python
  def _set_page_tree(self):
        """
        设置页面有向图。init中调用
        """
        self.page_tree.add_turn('我的页', '设置页', self._get_page(page_name='我的页').goto_setting)
        self.page_tree.add_turn('设置页', '登录页', self._get_page(page_name='设置页').goto_login)
        self.page_tree.add_turn('我的页', '登录页', self._get_page(page_name='我的页').goto_login)

        for p in self.page_map.keys():
            self.page_tree.add_turn(p, '主页', self._get_page(page_name=p).return_main_page)
        
        self.page_tree.add_turn('XXX页', '主页', self._get_page(page_name='XXX页').return_main_page)
        self.page_tree.add_turn('YYY页', '主页', self._get_page(page_name='XXX页').return_main_page)
```

这个方法强依赖具体业务，对应的图如下：

![image-20210628190048281](/assets/img/image-20210628190048281.png)

5、新的登录操作-登录页面内完成：

```python
class LoginPage(BasePage):
    name = '登录页'

    def login_by_account(self, username, password):
        if 首次登录:
            self.client.click('帐号密码登录')  # 自选    
        self.client.input(帐号输入框, username)
        self.client.input(密码输入框, password)
        self.client.click(登录)
```

新的登录操作-完整版，带上跳转页之后的操作：

```python
class Operation(BaseOperation):
    def __init__(self, device_id):
        self.client = UiClient(device_id)
        self.navigation = Navigation(self.client)

     def login_by_account(self, username, password):
        if not self.navigation.navigate_to_page('登录页'):
            self.exit_login()
        if getattr(self.navigation, '登录页').login_by_account(username, password):
            self.navigation.page_tree.change_weight('我的页', '登录页', 999)
```

