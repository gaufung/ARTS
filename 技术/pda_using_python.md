---
date: 2017-01-20 18:49
status: draft
title: 'PDA 程序实现'
---

# 0 背景
最近帮朋友使用 PDA 方法处理一些数据，学术上的背景知识参考该[文献](http://cdmd.cnki.com.cn/Article/CDMD-10384-1014190914.htm)。项目难度不大，可能是第一次用 `python` 脚本语言处理这么复杂的数据，过程中遇到了一些问题，记录下来以供以后查阅，代码放在 github 上了。[传送门](https://github.com/gaufung/LMDI)

# 1 数据读取
数据是全部存放在 `excel` 中，以前用 CSharp 处理 `excel` 问题用两种方法，一种是微软的**office** 的 com 组件，程序中添加引用，但是速度超级慢，并且有 office 宿主环境需求，而且版本也有要求；另一种是开源的的 `NPOI` 组件，只需要添加一个 dll 连接即可。在 Mac 使用 .net 编写这些程序不太靠谱，于是换 `python` 处理这些程序。 `Python` 世界中有两个很重要的包 `xlrd` 和 `xlwt` 分别对应了 `excel` 的读和写，使用起来非常方便。

```python

import xlrd 
workbook = xlrd.open_workbook(XLS_FILE_PATH)
table = workbook.sheets()[idx]
row = table.row_values[row_idx]
column = table.row_value[col_idx]

import xlwt

workbook = xlwt.Workbook(encoding='utf8')
sheet = workbook.add_sheet(sheet_name)
sheet.write(0,0,label='gaufung')
workbook.save(XLS_FILE_PATH)
```

# 2 线性规划

线性规划的方法采用类似 `DEA` 的方法，不过与传统的认识的线性规划还不太一样，只给出的subject 条件，要最优化的变量也包含在 subject 条件中， 因此需要添加一些松弛变量，使不等式条件变成等式，将最优化的变量表达出来，转换成标准的线性规划。具体转换过程参考以前[博文](http://gaufung.info/post/solve_dea_using_r)。
`Python ` 开源包中有一个 `pulp` 能够处理线性规划问题，参考文档，使用起来也比较方便。

```python
from pulp import LpProblem, lpSum, LpVariable, LpMinimize, LpMaximize
prob = LpProblem("lambda_min", LpMinimize)
prob += lpSum([cost[i] * symbols[i] for i in ingredients])
prob += lpSum([pro_dict[i] * symbols[i]
                   for i in ingredients]) >= production_right
prob += lpSum([co2_dict[i] * symbols[i] for i in ingredients]) == co2_right
prob.solve()
```

# 3 config 设置
不同于编译后执行的程序, `pyhton` 解释型程序好处是可以通过直接写代码，将配置为文件写入中代码中，而不是繁琐的 `xml` 或者 `json` 文件写填写配置文件，所以在程序中将一些与数据读入的有关的配置文件直接写入到列表或者字典中，在程序中读取列表和字典中的值，获取数据。

# 4 缓存机制
虽说 `python` 的 slogan 是 *Life is short, I use python*， 但是处理数据过程中发现速度非常 *感人*， 通过代码审查，发现重复计算太多，比如一个指数，多次调用将会多次计算，而一个对象会在不同情景下多次被创建。为了加快数据处理速度，增加了两级缓存，在类的对象中增加一个 `_cache` 对象，每一个指数名称和该指数作为键值，添加到字典缓存中; 每个对象都存放到一个全局缓存中，通过该对象的名称作为 `key`， 查询对象时如果存在，则返回；反之，创建对象并返回。

```python
class LmdiFactory(object):
    '''
    Lmdi factory
    '''
    cache = {}
    @classmethod
    def build(cls, dmus_t, dmus_t1, name):
        if not LmdiFactory.cache.has_key(name):
            LmdiFactory.cache[name] = Lmdi(dmus_t, dmus_t1, name)
        return LmdiFactory.cache[name]
```

# 5 单元测试  
一直以来，对单元测试不够重视，在使用 `Python` 处理 `pda` 问题的时候，不停的修改需求和试错重构，单元测试太重要的，而且 `python` 自带了单元测试模块 `unittest`,使用起来非常方便，在完成代码后，第一时间进行单元测试，效果拔群。
```python
class TestModel(unittest.TestCase):
    '''
    test model
    '''
    def test_energy(self):
        '''
        test energy model
        '''
        energy = Model.Energy('北京', [10.2, 33.0, 43.2])
        self.assertEqual(energy.name, '北京')
        self.assertEqual(len(energy), 3)
        self.assertEqual(energy[0], 10.2)
        self.assertEqual(energy[-1], energy.total)
    def test_co2(self):
        '''
        test co2 model
        '''
        co2 = Model.Co2('上海', [88.2, 10.2, 98.4])
        self.assertEqual(co2.name, '上海')
        self.assertEqual(len(co2), 3)
        self.assertEqual(co2[0], 88.2)
        self.assertEqual(co2[-1], co2.total)
    def test_pro(self):
        '''
        test production model
        '''
        pro = Model.Production('江苏', 66)
        self.assertEqual(pro.name, '江苏')
        self.assertEqual(pro.production, 66)
if __name__ == '__main__':
    unittest.main()
```