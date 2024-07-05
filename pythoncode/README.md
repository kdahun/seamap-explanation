
```
import pymysql
import pandas as pd

# 데이터베이스 연결 설정
connection = pymysql.connect(
  host = '202.31.147.129',
  user = 'jisung',
  passward = 'Wldnjs981212@@',
  db = 'weater',
  port = 13306
)

query = "SELECT * FROM wave3;"

df = pd.read_sql_query(query, connection)
connection.close()

display(df)
```
### MariaDB 연결 설정 적용 및 커서 생성
```
connection = pymysql.connect(
  host = '202.31.147.129',
  user = 'jisung',
  passward = 'Wldnjs981212@@',
  db = 'weater',
  port = 13306
)

# 커서 생성
cursor = connection.cursor()

# 데이터 베이스 생성
CREATE TABLE IF NOT EXISTS wave10("""
  num varchar(10),
  latitude varchar(10),
  longitude varchar(10),
  wavesize varchar(10)
""")
```

### 데이터프레임의 값을 MySQL에 삽입
```
for index, row in df.iterrows():
  cursor.execute("INSERT INTO wave10 (num, latitude, longitude, wavesize) VALUES (%s,%s,%s,%s);",(row[0],row[1],row[2],row[3]))

# 변경사항 저장
connection.commit()

# 저장된 경도와 위도 값 조회
select_query = "SELECT * FROM wave10"
cursor.execute(select_query)

# 결과 출력
results = cursor.fetchall()
print(results)
```

### 육지 처리
```
# 2023년 07월 30일 09시의 해구 정보만 df에 저장
df=df[df["Date"]=="2023073009"]

# num의 해구 번호에 '번'이 없는 열을 만들기 위해 hegu 리스트 만들기
hegu = []
for i in df['num']:
    hegu.append(i[:-1])

# hegu 열 추가
df['hegu'] = hegu

# df에 모든 열이 문자열이기 때문에 위도, 경도, 해구번호를 실수로 변경
df['latitude']=df['latitude'].astype(float)
df['longitude']=df['longitude'].astype(float)
df['hegu']=df['hegu'].astype(float)

# 파고를 실수 처리 하기 위함 / 비어있는 부분은 예외처리로 0을 넣어줌
li = []
for i in df['wavesize'].str[:-1]:
    try:
      li.append(float(i))
    except:
        li.append(float(0))

# 위도, 경도의 최소, 최대 값을 알아내 해구 데이터에서 얻을 수 없는 육지 위치 
import numpy as np
from itertools import product

array1 = np.arange(25,46,0.5)
array2 = np.arange(119,140,0.5)

# 두 배열의 모든 조합 생성
combinations = list(product(array1,array2))
```

### 파고에 따른 등급 나누기
```
new_data = []
# wavesize가 0인 경우
hegu_f = [5130, 5131, 5137, 5138, 81, 62, 63, 69, 75, 5087, 5097, 5098, 5099, 5113, 5114]
# wavesize가 없는 경우
unique_vals = [  75,  384, 5021, 5029, 5047, 5055, 5097, 5098, 5131, 5164, 6003, 6058,
 6062, 6066, 6076, 6077, 6137, 6138, 6139, 6454, 6455, 6486, 6601, 6602,
 6604, 6614, 6624, 6642, 6950, 6954, 6958, 6962, 6969, 7349, 7360, 7361,
 7373, 7383, 7413, 7922, 7923, 8300, 5213,  393,  394]

for lat, lon in combinations:
    continent = 0 # 육지인지 바다인지
    wave = 0 # 파도 크기
    for index, row in df.iterrows():
        hegu=0
        if row['latitude'] == lat and row['longitude'] ==lon:
            continent = 1
            hegu=row['hegu']
            wave = row['wavesize']
            if  wave < 1.5:
                wave = 7 # 내해 (황천 무급)
            elif wave < 2:
                wave = 6 # 내해 (황천 6급)
            elif wave < 2.5:
                wave = 5 # 내해 (황천 5급)
            elif wave < 3:
                wave = 4 # 내해 (황천 4급)
            elif wave < 3.5:
                wave = 3 # 내해 (황천 3급)
            elif wave < 4:
                wave = 2 # 내해 (황천 2급)
            elif wave >= 4:
                wave = 1 # 내해 (황천 1급)
            else :
                wave = 0 
            if hegu in hegu_f:
                wave = 0
            if hegu in unique_vals:
                wave = 0

            break

    new_data.append({'latitude':lat,'longitude':lon,'sea':continent,'wave':wave,'hegu':hegu})
new_df = pd.DataFrame(new_data)
```

### networkx를 사용해 그래프 만들기
```
import pandas as pd
import networkx as nx
import matplotlib.pyplot as plt

# 그래프 생성
G = nx.Graph()

# 노드 추가
for idx, row in new_df.iterrows():
    if not G.has_node(idx):
        G.add_node(idx, latitude=row['latitude'], longitude=row['longitude'], sea=row['sea'],wave=row['wave'])

# 노드 간 에지 연결
for idx1, node1 in new_df.iterrows():
    for idx2, node2 in new_df.iterrows():
        lat_diff = abs(node2['latitude'] - node1['latitude'])
        lon_diff = abs(node2['longitude'] - node1['longitude'])

        if (lat_diff == 0.5 and lon_diff == 0) or (lat_diff == 0 and lon_diff == 0.5)or (lat_diff == 0.5 and lon_diff == 0.5):
            G.add_edge(idx1, idx2)

# 간격 설정 (예: 1 unit)
x_gap = 2
y_gap = 2

unique_latitudes = sorted(new_df['latitude'].unique())
unique_longitudes = sorted(new_df['longitude'].unique())

pos = {}
for idx, row in new_df.iterrows():
    x = unique_longitudes.index(row['longitude']) * x_gap
    y = unique_latitudes.index(row['latitude']) * y_gap
    pos[idx] = (x, y)
```
1. 그래프 생성(G = nx.Graph()) : networkx.Graph 객체 생성 / G는 무방향 그래프를 나타내고 노드와 엣지를 추가할 수 있다.
2. 노드 추가
```
for idx, row in new_df.iterrows():
    if not G.has_node(idx):
        G.add_node(idx, latitude=row['latitude'], longitude=row['longitude'], sea=row['sea'], wave=row['wave'])
```
new_df 데이터프레임의 각 행을 반복하면서 그래프에 노드를 추가해준다.
  - new_df.iterrows() : 각 행과 해당 행의 인덱스를 가져온다.
  - G.has_node(idx)는 그래프 G에 인덱스 idx를 가진 노드가 이미 존재하는지 확인
  - 만약 노드가 없다면 G.add_node(idx, latitude=row['latitude'], longitude = row['longitude'], sea = row['sea'], wave = row['wave'])로 노드를 추가한다. 여기서 idx는 식별자이고, 노드에 latitude, longitude, sea, wave라는 속성을 추가

3. 노드간 에지 연결
```
for idx1, node1 in new_df.iterrows():
    for idx2, node2 in new_df.iterrows():
        # idx1과 idx2의 위도를 뺀 값
        lat_diff = abs(node2['latitude'] - node1['latitude'])
        
        # idx1과 idx2의 경도를 뺀 값
        lon_diff = abs(node2['longitude'] - node1['longitude'])

        # 위도, 경도, (위도, 경도) 들이 0.5 차이가 날때
        if (lat_diff == 0.5 and lon_diff == 0) or (lat_diff == 0 and lon_diff == 0.5) or (lat_diff == 0.5 and lon_diff == 0.5):
            G.add_edge(idx1, idx2) # 조건이 만족하면 idx1과 idx2 노드간에 에지를 추가
```
4. 노드의 위치 설정
```
x_gap = 2
y_gap = 2

unique_latitudes = sorted(new_df['latitude'].unique())
unique_longitudes = sorted(new_df['longitude'].unique())
```
  - x_gap과 y_gap은 노드 간의 가로 및 세로 간격을 설정
  - unique_latitudes와 unique_longitudes는 고유한 latitude와 longitude 값들을 정렬하여 리스트로 만듬

5. 노드 위치 계산
```
pos = {}
for idx, row in new_df.iterrows():
    x = unique_longitudes.index(row['longitude']) * x_gap
    y = unique_latitudes.index(row['latitude']) * y_gap
    pos[idx] = (x, y)
```

6. 
7. 






