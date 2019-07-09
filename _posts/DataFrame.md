1. **df.at vs df.loc**

   - `df.at` can only access a single value at a time.

     `df.loc` can select multiple rows and/or columns.

   - .at tries to conserve the datatype 

   - **.at is faster than .loc not a bit !**

2. **df[name] vs df.loc[:,name]**

   df.loc[:,name] is better than df[name]，df[name] may cause memory error !

   df.loc[:,name:name] is better than df.loc[:,[name]]

3. **df.groupby 分组后分别对每组进行操作**

   ```
   df = pd.DataFrame({"name":["Qi","Bill","Bill","Qi","Bill","Bill","Qi","Qi"],
   "score":[2,3,4,1,2,3,4,5]})
   // 计算每人的最高得分
   // 第一种方法适用于需要指定其他参数的函数
   df.groupby("name").apply(lambda x: max(x["score"]))
   // 第一种方法适用于不需指定多余参数的函数
   df.groupby("name")["score"].max()
   
   // 计算每人过去三次考试的最高得分
   // 不需要多余参数
   df.groupby("name")["score"].rolling(3).max()
   // 需要指定多余参数 apply not supported by groupby.rolling
   df.groupby("name")["score"].rolling(3).apply(lambda x:max(x),raw=True)
   //df.groupby("name").rolling(3).apply(lambda x:max(x["score"]),raw=True) // False
   ```

4. **pandas apply 用法** (仅需关注 **传入的参数**、**是否自定义函数** )

   - 当调用的函数需要多余参数时，apply就显得很有必要。通过 ```.apply(lambda x: func(x,...))``` 的形式
   - 当使用custom function 时，也需要用 apply 函数。```.apply(func)``` or ```.apply(lambda x: func(x,...))```

   注1：关于传入```.apply(lambda x: ...)``` 中```x```的类型：

   - ```df.apply```  the type of passed x is series
   - ```df["name"]``` the type of passed x is string
   - ```df.groupby("name").apply ```the type of passed x is DataFrameGroupBy
   - ```df.groupby("name")["score"].apply``` the type of passed x is SeriesGroupBy
   - ```df.groupby("name").rolling(3).apply``` window
   - ```df.groupby("name")["score"].rolling(3).apply``` window

   注2: 关于```.apply()```的返回值类型：

   ​	由function决定，有三种可能：single value、series、dataframe

   扩展：

   ​	实际应用中，我们可能会遇到这样的需求，分组批量改变原始dataframe或series。这里的一个深坑就是for循环遍历每个group，然后.loc[] update original values by conditions。这样效率极低，笔者深受其害。。。此时大杀器apply便派上用场，用custom function修改传入的DataFrameGroupBy or SeriesGroupBy，然后返回新的Datafame or series即可，这样效率可以大大提升。

5. **pandas transform 用法**

   ​	data science 中计算dataframe数据的统计信息数据通常抽象成为原始dataframe增加一列，该列信息即为统计数据。一般情况下我们的需求是对dataframe的单列进行统计，如果是多列统计则将多列信息计算抽象成成单列，问题来了，该列应该怎么附加在原始数据帧。

   ​	如果统计信息仅是一个value,  df[$column_name] = value 即可

   ​	如果统计信息为multiple values, 这种情况比较常见，是由于分组统计原始dataframe信息导致的。那我们应该怎么办？显而易见，首先对dataframe进行groupby操作，此时我们得到SeriesGroupBy类型的数据，接下来对该SeriesGroupBy进行相应所需的函数操作。函数返回值又有两种可能，一种情况是single value，另一种情况是multiple value。

   ​	如果函数返回值为single value, 可以与key组成dataframe 然后与原始dataframe 进行 merge 操作，这种方法比较繁琐。精简的做法是大杀器tansform操作，然后得到与原来dataframe同index的series，直接将其assign到原始dataframe即可。用法如下：

   ```factor["temp"] = factor[factor_name].groupby(factor["F034V"]).transform('mean')```

   ```factor["temp"] = factor.groupby(factor["F034V"])[factor_name].transform('mean')```

   ​	如果函数返回值为multiple value，通常返回值series的size与传入的参数series相同，我们只需对返回值series调用reset_index(0,drop=True)将groupby key去掉，得到与原始dataframe相同index的series，再将其assign到原始dataframe。用法如下：

   ```python
   MOM["FACTOR"] = bar.groupby("SECCODE").apply(lambda x: talib.MOM(x["F007N"], timeperiod=dict_factor[factor])).reset_index(0,drop=True)
   ```

   ```python
   # 重置index
   portfolio_data = data.groupby("TRADEDATE").apply(lambda x: x.head(30)).reset_index(drop=True)
   # 保留原有index
   portfolio_data = data.groupby("TRADEDATE").apply(lambda x: x.head(30)).reset_index(0,drop=True)
   ```



   需要注意的是，函数返回值为multiple value时，返回值series并不一定含有groupby key, 此时也无需进行reset_index 操作。例：

   ```python
   from numpy.random import *
   df = pd.DataFrame({'A' : ['foo', 'bar', 'foo', 'bar',
                             'foo', 'bar', 'foo', 'foo'],
                      'B' : ['one', 'one', 'two', 'three',
                            'two', 'two', 'one', 'three'],
                      'C' : randn(8), 'D' : randn(8)})
   
   zscore = lambda x: (x - x.mean()) / x.std() 
   print(df.groupby('A')["C"].apply(zscore)) 
   '''
   0    0.520973
   1   -0.945072
   2    0.151964
   3    1.047105
   4    1.186857
   5   -0.102033
   6   -0.397239
   7   -1.462555
   Name: C, dtype: float64
   '''
   ```

   返回值series含有groupby key一般是由于rolling操作导致的，不涉及rolling可不用考虑reset_index,如：

   ```python
   print(df.groupby('A')["C"].rolling(2).apply(max, raw=True))  # equal to print(df.groupby('A')["C"].rolling(2).max())
   '''
   A
   bar  1         NaN
        3    0.230627
        5    0.310597
   foo  0         NaN
        2    0.763590
        4    0.763590
        6   -1.084255
        7    0.343261
   Name: C, dtype: float64
   '''
   ```

6. **pandas merge vs join vs concat vs append**

7. **pandas filter by condition**

   df.loc 

   df.at

8. **dataframe fillna**

   It is filled by index.

   - df1.fillna(df2, inplace=True)
   - df1["column1"].fillna(df2["column1"], inplace=True)