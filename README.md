
# 爬取腾讯数据制作感染热力图

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




    {'confirm': 37251, 'suspect': 28942, 'dead': 812, 'heal': 2685}




```python
chinaAdd
```




    {'confirm': 2653, 'suspect': 1285, 'dead': 89, 'heal': 633}



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
    china_data['suspect'] = china_data['total'].map(suspect)
    china_data['dead'] = china_data['total'].map(dead)
    china_data['heal'] = china_data['total'].map(heal)
    china_data['addconfirm'] = china_data['today'].map(confirm)
    china_data['addsuspect'] = china_data['today'].map(suspect)
    china_data['adddead'] = china_data['today'].map(dead)
    china_data['addheal'] = china_data['today'].map(heal)
    china_data = china_data[["province","city","confirm","suspect","dead","heal","addconfirm","addsuspect","adddead","addheal"]]
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
      <th>suspect</th>
      <th>dead</th>
      <th>heal</th>
      <th>addconfirm</th>
      <th>addsuspect</th>
      <th>adddead</th>
      <th>addheal</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>湖北</td>
      <td>武汉</td>
      <td>14982</td>
      <td>0</td>
      <td>608</td>
      <td>877</td>
      <td>1379</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>湖北</td>
      <td>孝感</td>
      <td>2436</td>
      <td>0</td>
      <td>29</td>
      <td>45</td>
      <td>123</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>湖北</td>
      <td>黄冈</td>
      <td>2141</td>
      <td>0</td>
      <td>43</td>
      <td>135</td>
      <td>100</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>湖北</td>
      <td>荆州</td>
      <td>997</td>
      <td>0</td>
      <td>13</td>
      <td>33</td>
      <td>56</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>湖北</td>
      <td>襄阳</td>
      <td>988</td>
      <td>0</td>
      <td>7</td>
      <td>40</td>
      <td>81</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>



把清洗出的数据，存成csv。


```python
china_data.to_csv('china_data.csv')
```

## 4*、数据可视化

再进一步，把国内数据按照省和感染人数重新组合，为进一步可视化作准备


```python
#数据处理
area_data = china_data.groupby("province")["confirm"].sum().reset_index()
area_data.columns = ["province","confirm"]
area_data.head()
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
      <th>confirm</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>上海</td>
      <td>292</td>
    </tr>
    <tr>
      <th>1</th>
      <td>云南</td>
      <td>140</td>
    </tr>
    <tr>
      <th>2</th>
      <td>内蒙古</td>
      <td>54</td>
    </tr>
    <tr>
      <th>3</th>
      <td>北京</td>
      <td>315</td>
    </tr>
    <tr>
      <th>4</th>
      <td>台湾</td>
      <td>17</td>
    </tr>
  </tbody>
</table>
</div>



使用area_data绘图


```python
from pyecharts.charts import *
from pyecharts import options as opts
from pyecharts.globals import ThemeType

area_map = Map(init_opts=opts.InitOpts(theme=ThemeType.WESTEROS))
area_map.add("",[list(z) for z in zip(list(area_data["province"]), list(area_data["confirm"]))], "china",is_map_symbol_show=False)
area_map.set_global_opts(title_opts=opts.TitleOpts(title="2019_nCoV中国疫情地图"),visualmap_opts=opts.VisualMapOpts(is_piecewise=True,
                pieces = [
                        {"min": 1001 , "label": '>1000',"color": "#893448"}, #不指定 max，表示 max 为无限大
                        {"min": 500, "max": 1000, "label": '500-1000',"color": "#ff585e"},
                        {"min": 101, "max": 499, "label": '101-499',"color": "#fb8146"},
                        {"min": 10, "max": 100, "label": '10-100',"color": "#ffb248"},
                        {"min": 0, "max": 9, "label": '0-9',"color" : "#fff2d1" }]))
area_map.render_notebook()
```





<script>
    require.config({
        paths: {
            'echarts':'https://assets.pyecharts.org/assets/echarts.min', 'china':'https://assets.pyecharts.org/assets/maps/china', 'westeros':'https://assets.pyecharts.org/assets/themes/westeros'
        }
    });
</script>

        <div id="254b8d43750d4fd2a493b97ab79e2f58" style="width:900px; height:500px;"></div>

<script>
        require(['echarts', 'china', 'westeros'], function(echarts) {
                var chart_254b8d43750d4fd2a493b97ab79e2f58 = echarts.init(
                    document.getElementById('254b8d43750d4fd2a493b97ab79e2f58'), 'westeros', {renderer: 'canvas'});
                var option_254b8d43750d4fd2a493b97ab79e2f58 = {
    "animation": true,
    "animationThreshold": 2000,
    "animationDuration": 1000,
    "animationEasing": "cubicOut",
    "animationDelay": 0,
    "animationDurationUpdate": 300,
    "animationEasingUpdate": "cubicOut",
    "animationDelayUpdate": 0,
    "series": [
        {
            "type": "map",
            "label": {
                "show": true,
                "position": "top",
                "margin": 8
            },
            "mapType": "china",
            "data": [
                {
                    "name": "\u4e0a\u6d77",
                    "value": 292
                },
                {
                    "name": "\u4e91\u5357",
                    "value": 140
                },
                {
                    "name": "\u5185\u8499\u53e4",
                    "value": 54
                },
                {
                    "name": "\u5317\u4eac",
                    "value": 315
                },
                {
                    "name": "\u53f0\u6e7e",
                    "value": 17
                },
                {
                    "name": "\u5409\u6797",
                    "value": 78
                },
                {
                    "name": "\u56db\u5ddd",
                    "value": 386
                },
                {
                    "name": "\u5929\u6d25",
                    "value": 90
                },
                {
                    "name": "\u5b81\u590f",
                    "value": 45
                },
                {
                    "name": "\u5b89\u5fbd",
                    "value": 779
                },
                {
                    "name": "\u5c71\u4e1c",
                    "value": 435
                },
                {
                    "name": "\u5c71\u897f",
                    "value": 115
                },
                {
                    "name": "\u5e7f\u4e1c",
                    "value": 1120
                },
                {
                    "name": "\u5e7f\u897f",
                    "value": 195
                },
                {
                    "name": "\u65b0\u7586",
                    "value": 45
                },
                {
                    "name": "\u6c5f\u82cf",
                    "value": 468
                },
                {
                    "name": "\u6c5f\u897f",
                    "value": 740
                },
                {
                    "name": "\u6cb3\u5317",
                    "value": 206
                },
                {
                    "name": "\u6cb3\u5357",
                    "value": 1033
                },
                {
                    "name": "\u6d59\u6c5f",
                    "value": 1075
                },
                {
                    "name": "\u6d77\u5357",
                    "value": 128
                },
                {
                    "name": "\u6e56\u5317",
                    "value": 27100
                },
                {
                    "name": "\u6e56\u5357",
                    "value": 838
                },
                {
                    "name": "\u6fb3\u95e8",
                    "value": 10
                },
                {
                    "name": "\u7518\u8083",
                    "value": 79
                },
                {
                    "name": "\u798f\u5efa",
                    "value": 250
                },
                {
                    "name": "\u897f\u85cf",
                    "value": 1
                },
                {
                    "name": "\u8d35\u5dde",
                    "value": 96
                },
                {
                    "name": "\u8fbd\u5b81",
                    "value": 105
                },
                {
                    "name": "\u91cd\u5e86",
                    "value": 446
                },
                {
                    "name": "\u9655\u897f",
                    "value": 208
                },
                {
                    "name": "\u9752\u6d77",
                    "value": 18
                },
                {
                    "name": "\u9999\u6e2f",
                    "value": 26
                },
                {
                    "name": "\u9ed1\u9f99\u6c5f",
                    "value": 307
                }
            ],
            "roam": true,
            "zoom": 1,
            "showLegendSymbol": false,
            "emphasis": {}
        }
    ],
    "legend": [
        {
            "data": [
                ""
            ],
            "selected": {
                "": true
            },
            "show": true,
            "padding": 5,
            "itemGap": 10,
            "itemWidth": 25,
            "itemHeight": 14
        }
    ],
    "tooltip": {
        "show": true,
        "trigger": "item",
        "triggerOn": "mousemove|click",
        "axisPointer": {
            "type": "line"
        },
        "textStyle": {
            "fontSize": 14
        },
        "borderWidth": 0
    },
    "title": [
        {
            "text": "2019_nCoV\u4e2d\u56fd\u75ab\u60c5\u5730\u56fe",
            "padding": 5,
            "itemGap": 10
        }
    ],
    "visualMap": {
        "show": true,
        "type": "piecewise",
        "min": 0,
        "max": 100,
        "inRange": {
            "color": [
                "#50a3ba",
                "#eac763",
                "#d94e5d"
            ]
        },
        "calculable": true,
        "inverse": false,
        "splitNumber": 5,
        "orient": "vertical",
        "showLabel": true,
        "itemWidth": 20,
        "itemHeight": 14,
        "borderWidth": 0,
        "pieces": [
            {
                "min": 1001,
                "label": ">1000",
                "color": "#893448"
            },
            {
                "min": 500,
                "max": 1000,
                "label": "500-1000",
                "color": "#ff585e"
            },
            {
                "min": 101,
                "max": 499,
                "label": "101-499",
                "color": "#fb8146"
            },
            {
                "min": 10,
                "max": 100,
                "label": "10-100",
                "color": "#ffb248"
            },
            {
                "min": 0,
                "max": 9,
                "label": "0-9",
                "color": "#fff2d1"
            }
        ]
    }
};
                chart_254b8d43750d4fd2a493b97ab79e2f58.setOption(option_254b8d43750d4fd2a493b97ab79e2f58);
        });
    </script>



