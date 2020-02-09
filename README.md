
# 爬取腾讯2019_nCoV中国疫情数据

## 0,环境准备

1，anaconda默认安装所有需要的库，通过import来验证，或使用pip安装requests，json，pandas，jupyter notebook

2，准备chrome或火狐浏览器


```python
import requests
import json
import pandas as pd
requests.__version__, json.__version__, pd.__version__
```




    ('2.21.0', '2.0.9', '0.23.4')



## 1、分析数据源

1，通过浏览器打开数据源：https://news.qq.com/zt2020/page/feiyan.htm?from=timeline&isappinstalled=0

2，打开开发者选项，分析network流；

3，简述动态网页和静态网页的区别；

## 2、抓取数据

1，找到数据流，用python获取，查看内容；

2，用json加载数据，用可视化工具分析http://www.bejson.com/jsonviewernew/


```python
url = 'https://view.inews.qq.com/g2/getOnsInfo?name=disease_h5&callback=jQuery34105042688433769531_1581048970987&_=1581048970988'
reponse = requests.get(url=url)
data = reponse.text.split('jQuery34105042688433769531_1581048970987')[1][1:-1]
data = json.loads(data)
data
```


```python
data = json.loads(data['data'])
data
```

3，用json可视化工具，查看数据

4，封装函数：利用数据源得出的链接，获取到原始数据，然后用json处理成python字典并返回


```python
def catch_data():
    url = 'https://view.inews.qq.com/g2/getOnsInfo?name=disease_h5&callback=jQuery34105042688433769531_1581048970987&_=1581048970988'
    reponse = requests.get(url=url)
    data = reponse.text.split('jQuery34105042688433769531_1581048970987')[1][1:-1]
    data = json.loads(data)
    if data['ret'] == 0:
        return json.loads(data['data'])
```

5，利用函数获取到到数据，看一下国内感染数据及今日新增感染情况


```python
data = catch_data()
lastUpdateTime = data['lastUpdateTime']
chinaTotal = data['chinaTotal']
chinaAdd = data['chinaAdd']
chinaTotal
```




    {'confirm': 37252, 'suspect': 28942, 'dead': 813, 'heal': 2747}




```python
chinaAdd
```




    {'confirm': 2654, 'suspect': 1285, 'dead': 90, 'heal': 695}



## 3、利用pandas清洗国内感染数据并保存到csv


```python
# 定义数据处理函数，给定一个str，用eval变成字典，取其中元素并返回
def confirm(x):
    confirm = eval(str(x))['confirm']
    return confirm
def suspect(x):
    suspect = eval(str(x))['suspect']
    return suspect
def dead(x):
    dead = eval(str(x))['dead']
    return dead
def heal(x):
    heal =  eval(str(x))['heal']
    return heal
```

Pandas中文网：https://www.pypandas.cn/


```python
#细致筛选数据，生成list
def extract_china(china_data):
    china_list = []
    for a in range(len(china_data)):
        province = china_data[a]['name']
        province_list = china_data[a]['children']
        for b in range(len(province_list)):
            city = province_list[b]['name']
            total = province_list[b]['total']
            today = province_list[b]['today']
            china_dict = {}
            china_dict['province'] = province
            china_dict['city'] = city
            china_dict['total'] = total
            china_dict['today'] = today
            china_list.append(china_dict)

    china_data = pd.DataFrame(china_list)
    # 函数映射
    china_data['confirm'] = china_data['total'].map(confirm)
#     china_data['suspect'] = china_data['total'].map(suspect)
    china_data['dead'] = china_data['total'].map(dead)
    china_data['heal'] = china_data['total'].map(heal)
    china_data['addconfirm'] = china_data['today'].map(confirm)
#     china_data['addsuspect'] = china_data['today'].map(suspect)
    china_data['adddead'] = china_data['today'].map(dead)
    china_data['addheal'] = china_data['today'].map(heal)
    china_data = china_data[["province","city","confirm","dead","heal","addconfirm"]]
    return china_data
```

利用上面的函数，原始数据中‘areaTree’中的‘children’清洗出来


```python
areaTree = data['areaTree']
china_data = areaTree[0]['children']#0-中国
china_data = extract_china(china_data)
china_data.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>province</th>
      <th>city</th>
      <th>confirm</th>
      <th>dead</th>
      <th>heal</th>
      <th>addconfirm</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>湖北</td>
      <td>武汉</td>
      <td>14982</td>
      <td>608</td>
      <td>877</td>
      <td>1379</td>
    </tr>
    <tr>
      <th>1</th>
      <td>湖北</td>
      <td>孝感</td>
      <td>2436</td>
      <td>29</td>
      <td>45</td>
      <td>123</td>
    </tr>
    <tr>
      <th>2</th>
      <td>湖北</td>
      <td>黄冈</td>
      <td>2141</td>
      <td>43</td>
      <td>135</td>
      <td>100</td>
    </tr>
    <tr>
      <th>3</th>
      <td>湖北</td>
      <td>荆州</td>
      <td>997</td>
      <td>13</td>
      <td>40</td>
      <td>56</td>
    </tr>
    <tr>
      <th>4</th>
      <td>湖北</td>
      <td>襄阳</td>
      <td>988</td>
      <td>7</td>
      <td>40</td>
      <td>81</td>
    </tr>
  </tbody>
</table>
</div>



把清洗出的数据，存成csv。


```python
china_data.to_csv('china_data.csv')
```

全国按天计数数据


```python
chinaDayList = pd.DataFrame(data['chinaDayList'])
chinaDayList = chinaDayList[['date','confirm','suspect','dead','heal']]
chinaDayList.to_csv('china_daily_data.csv')
chinaDayList
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>date</th>
      <th>confirm</th>
      <th>suspect</th>
      <th>dead</th>
      <th>heal</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>01.13</td>
      <td>41</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>01.14</td>
      <td>41</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>01.15</td>
      <td>41</td>
      <td>0</td>
      <td>2</td>
      <td>5</td>
    </tr>
    <tr>
      <th>3</th>
      <td>01.16</td>
      <td>45</td>
      <td>0</td>
      <td>2</td>
      <td>8</td>
    </tr>
    <tr>
      <th>4</th>
      <td>01.17</td>
      <td>62</td>
      <td>0</td>
      <td>2</td>
      <td>12</td>
    </tr>
    <tr>
      <th>5</th>
      <td>01.18</td>
      <td>198</td>
      <td>0</td>
      <td>3</td>
      <td>17</td>
    </tr>
    <tr>
      <th>6</th>
      <td>01.19</td>
      <td>275</td>
      <td>0</td>
      <td>4</td>
      <td>18</td>
    </tr>
    <tr>
      <th>7</th>
      <td>01.20</td>
      <td>291</td>
      <td>54</td>
      <td>6</td>
      <td>25</td>
    </tr>
    <tr>
      <th>8</th>
      <td>01.21</td>
      <td>440</td>
      <td>37</td>
      <td>9</td>
      <td>25</td>
    </tr>
    <tr>
      <th>9</th>
      <td>01.22</td>
      <td>571</td>
      <td>393</td>
      <td>17</td>
      <td>25</td>
    </tr>
    <tr>
      <th>10</th>
      <td>01.23</td>
      <td>830</td>
      <td>1072</td>
      <td>25</td>
      <td>34</td>
    </tr>
    <tr>
      <th>11</th>
      <td>01.24</td>
      <td>1287</td>
      <td>1965</td>
      <td>41</td>
      <td>38</td>
    </tr>
    <tr>
      <th>12</th>
      <td>01.25</td>
      <td>1975</td>
      <td>2684</td>
      <td>56</td>
      <td>49</td>
    </tr>
    <tr>
      <th>13</th>
      <td>01.26</td>
      <td>2744</td>
      <td>5794</td>
      <td>80</td>
      <td>51</td>
    </tr>
    <tr>
      <th>14</th>
      <td>01.27</td>
      <td>4515</td>
      <td>6973</td>
      <td>106</td>
      <td>60</td>
    </tr>
    <tr>
      <th>15</th>
      <td>01.28</td>
      <td>5974</td>
      <td>9239</td>
      <td>132</td>
      <td>103</td>
    </tr>
    <tr>
      <th>16</th>
      <td>01.29</td>
      <td>7711</td>
      <td>12167</td>
      <td>170</td>
      <td>124</td>
    </tr>
    <tr>
      <th>17</th>
      <td>01.30</td>
      <td>9692</td>
      <td>15238</td>
      <td>213</td>
      <td>171</td>
    </tr>
    <tr>
      <th>18</th>
      <td>01.31</td>
      <td>11791</td>
      <td>17988</td>
      <td>259</td>
      <td>243</td>
    </tr>
    <tr>
      <th>19</th>
      <td>02.01</td>
      <td>14380</td>
      <td>19544</td>
      <td>304</td>
      <td>328</td>
    </tr>
    <tr>
      <th>20</th>
      <td>02.02</td>
      <td>17236</td>
      <td>21558</td>
      <td>361</td>
      <td>475</td>
    </tr>
    <tr>
      <th>21</th>
      <td>02.03</td>
      <td>20471</td>
      <td>23214</td>
      <td>425</td>
      <td>632</td>
    </tr>
    <tr>
      <th>22</th>
      <td>02.04</td>
      <td>24363</td>
      <td>23260</td>
      <td>491</td>
      <td>892</td>
    </tr>
    <tr>
      <th>23</th>
      <td>02.05</td>
      <td>28060</td>
      <td>24702</td>
      <td>564</td>
      <td>1153</td>
    </tr>
    <tr>
      <th>24</th>
      <td>02.06</td>
      <td>31211</td>
      <td>26359</td>
      <td>637</td>
      <td>1542</td>
    </tr>
    <tr>
      <th>25</th>
      <td>02.07</td>
      <td>34598</td>
      <td>27657</td>
      <td>723</td>
      <td>2052</td>
    </tr>
    <tr>
      <th>26</th>
      <td>02.08</td>
      <td>37251</td>
      <td>28942</td>
      <td>812</td>
      <td>2651</td>
    </tr>
  </tbody>
</table>
</div>



## 4*、数据可视化
