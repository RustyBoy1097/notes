## 快速查看整体信息

##### 1. `.info()`

是DataFrame才可用的API，快捷 查看多种信息：总行数和列数、每列元素类型和non-NaN的个数，总内存。
```python
DataFrame.info(verbose=None, memory_usage=True, null_counts=True)
```

* verbose：True or False，字面意思是冗长的，也就说如何DataFrame有很多列，是否显示所有列的信息，如果为否，那么会省略一部分；
* memory_usage：True or False，默认为True，是否查看DataFrame的内存使用情况；
* null_counts：True or False，默认为True，是否统计NaN值的个数。 

##### 2. `.ndim`, `.shape`, `.size`

查看维数，形状，元素个数。

##### 3. `.head()`, `.tail()`

默认分别查看头5行和后5行。

##### 4. `.describe()`

快速查看每一列的统计信息，默认排除所有NaN元素。

```py
DataFrame.describe( include= [np.number])
```

* include：'all'或者[np.number 或 np.object]。numberic只对元素属性为数值的列做数值统计，object只对元素属性为object的列做类字符串统计。

 
##### 5. `.columns()`

查看列名

##### 6. `.isnull.sum()`, `.notnull.sum()`

查看缺失值总数

##### 7. `(df.shape[0]-df.count())/df.shape[0]`

查看每个特征的缺失率

`(df.shape[1]-df.T.count())/df.shape[1]`

查看每个个体所缺失的特征个数

##### 8. 使用missingno库

```python
import missingno as msno
missingValueColumns = merged.columns[merged.isnull().any()].tolist()
/** bar图缺失值可视化分析 */
msno.bar(merged[missingValueColumns],figsize(20,8),color="#34495e",fontsize=12,labels=True)
/** matrix密集图查看数据完整性 */
msno.matrix(merged[missingValueColumns],width_ratios=(10,1),figsize=(20,8),color=(0,0, 0),fontsize=12,sparkline=True,labels=True)
/** heatmap图查看相关性 */
msno . heatmap(merged[missingValueColumns],figsize = ( 20 , 20 ))
/** dendrogram树状图查看相关性 */
msno.dendrogram(collisions)
```

##### 9. 使用pandas_profiling库

```python
import pandas_profiling
pandas_profiling.ProfileReport(df)
```

1. bar图缺失值可视化分析

```
msno.bar(merged[missingValueColumns],figsize(20,8),color="#34495e",fontsize=12,labels=True)
```

2. matrix密集图查看数据完整性

```
msno.matrix(merged[missingValueColumns],width_ratios=(10,1),figsize=(20,8),color=(0,0, 0),fontsize=12,sparkline=True,labels=True)
```

3. heatmap图查看相关性

```
msno. heatmap(merged[missingValueColumns],figsize =(20 , 20))
```

4. dendrogram树状图查看相关性

```
msno.dendrogram(collisions)
```

## 处理缺失值

##### 1. `.dropna()`

parameter axis=1 deletes the columns

##### 2. `.fillna()`

* 1. 数字，`method='ffill'`（向前复制），`method='ffill'`（向后复制）
* 2. `df['height'].fillna(df.groupby('gender')['height'].transform('mean'), inplace=True)`  
   按照其他属性的统计值进行填充
* 3. `df['weight'].fillna(df['weight'].median(), inplace=True)`  
   按中位数(median),平均值(mean),众数(mode),等填充
* 4. `df['age'].interpolate(inplace=True)`  
   按插值填充

##### 3. 预测值填充

将缺失的数据当成label，没缺失的作为训练集，缺失的作为测试集，通过某种机器学习算法进行预测，填补缺失值。

##### 4. 聚类填充

##  数据分析

##### 1. EDA分析  
`plot`
##### 2. 异常检测算法(LOF,OCSVM,IFOREST,DBSCAN等）

##  数据清洗

##### 1. 删除重复（看情况而定）  
`data.duplicated()`和`data.drop_duplicates()`，前者标记出哪些是重复的（true），后者直接将重复删除；也可以根据单变量剔除重复，`data.drop_duplicates(['Area'])`
##### 2. 替换异常值  
  
例如用`describe()`进行一个描述分析：查看他们最大值，最小值等来分析是否存在明显的异常；  
通过`data[条件]`将这些异常的数据筛选出来进行分析，将异常值进行替换：
```
data['Age'].replace([158, 6], np.nan)
data['Package'].replace(-9, 0)
```
##### 3. 数据映射  

以Areas为例，Areas取四个地区：A/B/C/D，这四个地区在分析的时候并没有什么意义，但A/B/C为城市，D为农村，这个很有意义，所以我要根据areas创建新变量CType：U-城市、R-农村。

```
areas_to_ctype={'A':'U', 'B':'U', 'C':'U', 'D':'R', }
data['CType']=data['Areas'].map(areas_to_ctype)
```

##### 4. 数字变量类型化

以年龄为例：
```
cutPoint=[0,30, 40, 50,80]
groupLabel=[0,1,2,3]
data['ageGroup'] =pd.cut(data['Age'],cutPoint, labels=groupLabel)
```
按分位数划分：`qcut(data, n)`
```
data['ageGroup'] =pd.qcut(data['Age'],2)
```

##### 5. 创建哑变量

```
get.dummies( data[‘SHabit’] )
data =pd.merge(data, pd.get_dummies(data['SHabit'], prefix='SHabit'),right_index=True, left_index=True)
```
具体编码形式还有：

* LabelEncoder()
    
  ```
  from sklearn.preprocessing import OneHotEncoder, LabelEncoder
  le = LabelEncoder()data_df[ 'street_address'] = le.fit_transform(data_df['street_address'])
  ```
* OneHotEncoder()
    
  ```
  ohe = OneHotEncoder(n_values= 'auto', categorical_features='all', dtype=np.float64, sparse=True, handle_unknown='error')one_hot_matrix = ohe.fit_transform(data_df['street_address'])
  ```

* MeanEncoder()
  
  具体代码见 ()[https://zhuanlan.zhihu.com/p/26308272]

* corgi库（CountEncoder,MeanEncoder,OneHotEncoders）


##### 6. 文本处理

* 去除空白
    
    如：
    ```
    dat['Area'] = ['A ', 'C', ' A', 'D', ' B','D ', ' D', 'B ', 'C', ' A']
    ```
    
    `strip()`这个函数可以解决单个字符串的问题,但Series是不可以直接用strip()的；
    
    ```
    data['Areas'].map(str.strip)
    ```

* 分列
    
    ```
    data['ID'] = ['1:0', '2:0', '3:1', '4:0', '5:0', '6:0']
    ```
    
    (分号前面的是ID，分号后面的代表性别，0为男性，1为女性。)
    
    如果是单个字符串，`split()`可以直接使用，但对于Pandas的数据结构，则要写循环：
    ```
    IDGender = pd.DataFrame((x.split(':') for x in data.ID)， columns=['ID_new','Gender'])
    pd.merge(IDGender, data_noDup_rep_dum, right_index=True,left_index=True)
    ```
    如若之前做了数据清洗，需要换成原数据的索引只需在上面句程序机加上`index=data_noDup_rep_dum.index`

* 把多选题的文本创建成哑变量
    
    ```
    data['SHabit'] = ['1,2', '1,3', '2,3', '2', '2,3', '1,4', '1,2']
    ```
    想要的就是四个选项变成的四个问题：1 2 3 4，当一个人多选了1和2，那么就在问题1下面和问题2下面赋值为1，其他赋值为0。
    
    `get_dummies`并不能满足需求，`str.contains()` 可以在`SHabit`列中查找某个元素，当含有这个元素时，赋值为True，否则为False：
    
    ```
    data['SHabit_1'] = data['SHabit'].str.contains('1')
    ```
    用同样的方法生成SHabit_2、SHabit_3、SHabit_4。