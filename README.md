# Case5: Cyber Warrior Detection (ONGOING)

### Overview
The purpose of the project is to identify cyber warriors on PTT. There are two main indice: Political party impression, Comment density.

### Data set
2020/12/05 - 2020/12/15 all articles and comments


### Political party impression
1. data cleaning and target political party
```
# 讓使用者依據政黨網民觀察用字
import pandas as pd
print('關鍵字輸入方式:')
print('   國民黨、民進黨、民眾黨等')
print('   政治人物全名')  
target = input('欲觀測對象:')

# 資料匯入與擷取，降低模型測試時間
df = pd.read_csv('/content/drive/MyDrive/08 Data Base/cyber warrior.csv')
df = df.head(50000)

# 調整資料型態
del df['Unnamed: 0']
del df['matching_article_or_comment']
df.rename(columns={'if_news':'News', 'if_political':'Political', 'article_title':'Title', 'article_content':'Article', 'article_author':'ArticleID',
                   'article_ip':'ArticleIP', 'article_country':'ArticlePlace', 'article_time':'ArticleTime', 'if push':'Push', 
                   'comment_author':'CommentID', 'comment_content':'Comment', 'comment_time':'CommentTime'}, inplace=True)

# 段詞斷句
import jieba
import jieba.analyse

# 設定詞庫
file_path = '/content/drive/MyDrive/08 Data Base/dict.txt'
jieba.set_dictionary(file_path)

# 設定停用字詞庫
with open('/content/drive/MyDrive/08 Data Base/stop.txt', 'r', encoding='utf-8') as file:
  stop = file.read().split('\n')
words = ['欸','ㄟ','!','！','/', ' ', '會', '吃', '說', '沒','=','真的','米','知道','https','http','com','imgur','i','~','喔','買','想','一個','話','…','+','(',')','..','....']
for word in words:
  stop.append(word)

# 刪除停用詞
def remove_stop_words(seg_list):
  new_list = []  
  for seg in seg_list:
    if seg not in stop:
      new_list.append(seg)
  return new_list

# 清除\n分行符號
import re
df['Article'] = df['Article'].apply(str)
df['Comment'] = df['Comment'].apply(str)
def clean(x):
  if x is not None:
    x = re.sub('[\n]', '', x)
  else:
    pass
  return x

df['Article'] = df['Article'].apply(lambda x: clean(x))
df['Comment'] = df['Comment'].apply(lambda x: clean(x))

def backnone(x):
  if x == 'nan':
    x = None
  return x
df['Article'] = df['Article'].apply(lambda x: backnone(x))
df['Comment'] = df['Comment'].apply(lambda x: backnone(x))

# 段詞斷句
def CutSen(x):
  if x is not None:
    x = remove_stop_words(jieba.lcut(x, cut_all=False))
  return x
df['Article'] = df['Article'].apply(lambda x:CutSen(x))
df['Comment'] = df['Comment'].apply(lambda x:CutSen(x))

# 依據target篩選
def dummy(comment):
  x = 0
  if comment is not None:
    if target in comment:
      x = 1
  return x
df['Target'] = df['Comment'].apply(lambda x: dummy(x))
df = df[df['Target'] == 1]
```

2. target political party related words
```
# 計算詞頻
df = df.reset_index(drop=True) # 重新設定indx
freq = dict()
for i in range(len(df)):
  for k in range(len(df['Comment'][i])):
    if df['Comment'][i][k] not in freq.keys():
      freq[df['Comment'][i][k]] = 1
    else:
      freq[df['Comment'][i][k]] += 1
      
# 生成文字雲
from wordcloud import WordCloud
%matplotlib inline
import matplotlib.pyplot as plt

wc = WordCloud(width=5000,height=5000,max_words = 100, font_path = '/usr/local/lib/python3.6/dist-packages/matplotlib/mpl-data/fonts/ttf/taipei_sans_tc_beta.ttf')
# 使用dictionary的內容產生文字雲
wc.generate_from_frequencies(freq)

# 視覺化呈現
plt.imshow(wc)
plt.axis("off")
plt.figure(figsize=(20,10), dpi =200)
plt.show()
```


3. DPP related words

![](https://i.imgur.com/GTUfvXS.png)


4. KMT related words

![](https://i.imgur.com/9jScQVr.png)

### Cyber Warrior ID detection

1. data cleaning 
```
import pandas as pd

# 資料匯入
df = pd.read_csv('/content/drive/MyDrive/08 Data Base/cyber warrior.csv')

# 調整資料型態
del df['Unnamed: 0']
del df['matching_article_or_comment']
df.rename(columns={'if_news':'News', 'if_political':'Political', 'article_title':'Title', 'article_content':'Article', 'article_author':'ArticleID',
                   'article_ip':'ArticleIP', 'article_country':'ArticlePlace', 'article_time':'ArticleTime', 'if push':'Push', 
                   'comment_author':'CommentID', 'comment_content':'Comment', 'comment_time':'CommentTime'}, inplace=True)
del df['News']
del df['Political']
del df['Article']
del df['ArticlePlace']

# 以留言最多的coffee112為觀測對象
from pandasql import sqldf
pysqldf = lambda q: sqldf(q, globals())
q = """ SELECT CommentID, COUNT(Comment) AS Amount FROM df GROUP BY CommentID ORDER BY Amount DESC;"""
CommentAmount = pysqldf(q)
```

2. observe behavior pattern
```
target = CommentAmount['CommentID'][1]
Target = pd.DataFrame()
Target = df[df['CommentID'] == target]
Target = Target.reset_index(drop=True)

Target['CommentTime'] = pd.to_datetime(Target['CommentTime'])
q = "SELECT * FROM Target ORDER BY CommentTime"
Target = pysqldf(q)
Target['CommentTime'] = pd.to_datetime(Target['CommentTime'])

import datetime
interval = datetime.timedelta(minutes=1)

Target['Density'] = 0
for i in range(len(Target)):
  cen = Target['CommentTime'][i]
  for j in range(len(Target)):
    if Target['CommentTime'][j] >= cen:
      if Target['CommentTime'][j] - cen <= interval:
        Target['Density'][i] += 1
    elif Target['CommentTime'][j] <= cen:
      if cen - Target['CommentTime'][j] <= interval:
        Target['Density'][i] += 1
%matplotlib inline
import matplotlib.pyplot as plt

plt.figure(figsize=(20,6))
plt.plot(Target['Density'])
```
![](https://i.imgur.com/VlhYGki.png)
