# final_pjt
아시아경제 파이널 프로젝트

git pull

git add . # 다 올린다

git commit -m '작업내용 상세히 작성' # 올리면서 한번더 확인

git push # 최종 찐

------

# Lecture




# 2월 6일

(경로조심)

파이썬 venv 설치해야

키오스크의 로직을 연습하는 중

### get 방식 / post 방식으로 파라미터 받는 경우

```py
@app.route('/test_prm_get', methods = ['GET'])
def test_prm_get():
    id = request.args.get('DDD_ID')
    pw = request.args.get('DDD_PW')
    print(id, pw)
    
    @app.route('/test_prm_get', methods=['POST'])
def test_prm_post():
    id = request.form.get('DDD_ID')
    pw = request.form.get('DDD_PW')
    print(id, pw)
    return id+","+pw
```

### 키오스크 플라스크 (최종)

```py
#------------------------------------------------
# pip install Flask,  requests
#------------------------------------------------

from flask import Flask, session, render_template, make_response, jsonify, request, redirect, url_for

import cx_Oracle

import random
app = Flask(__name__)

app.secret_key = "1111122222"


@app.route('/')
def index():

    tel = str(random.randint(10, 99)) + str(random.randint(10, 99))
    session['MY_TEL_SESSION'] = tel
    return render_template('index.html')



@app.route('/menu')
def menu():
    conn = cx_Oracle.connect("ai", "0000", "localhost:1521/XE")
    sql = "select GOOD_SEQ,GOOD_NAME,GOOD_IMG,GOOD_PRICE,GOOD_DESC from kio_goods order by reg_date desc"
    cur = conn.cursor()
    cur.execute(sql)
    goods_list_dict = []
    for row in cur:
        #[3, 'CRISPY CHICKEN', 'static/images/menu/burger-11.jpg', 11, 'Chicken breast, chilli sauce, tomatoes, pickles, coleslaw']
        dic = {}
        dic["GOOD_SEQ"]  = row[0]
        dic["GOOD_NAME"] = row[1]
        dic["GOOD_IMG"]  = row[2]
        dic["GOOD_PRICE"] = row[3]
        dic["GOOD_DESC"] = row[4]
        goods_list_dict.append(dic)
    cur.close()
    conn.close()
    print(goods_list_dict)

    return render_template('menu.html'
                           , KEY_GOODS_LIST_DICT = goods_list_dict)





@app.route('/detail_view',  methods=['GET'])
def detail_view():
    v_seq = request.args.get("prm_good_seq")

    print("상세보기")
    conn = cx_Oracle.connect("ai", "0000", "localhost:1521/XE")
    sql = """  select GOOD_SEQ,GOOD_NAME,GOOD_IMG,GOOD_PRICE,GOOD_DESC
                from kio_goods
                where good_seq= :1"""
    cur = conn.cursor()
    cur.execute(sql, [v_seq])
    goods_list_dict = []
    for row in cur:
        # [3, 'CRISPY CHICKEN', 'static/images/menu/burger-11.jpg', 11, 'Chicken breast, chilli sauce, tomatoes, pickles, coleslaw']
        dic = {}
        dic["GOOD_SEQ"] = row[0]
        dic["GOOD_NAME"] = row[1]
        dic["GOOD_IMG"] = row[2]
        dic["GOOD_PRICE"] = row[3]
        dic["GOOD_DESC"] = row[4]
        goods_list_dict.append(dic)
    cur.close()
    conn.close()
    print(goods_list_dict)

    return render_template('product-single.html', KEY_GOODS_LIST_DICT = goods_list_dict)



def mydef_cart_list() :
    conn = cx_Oracle.connect("ai", "0000", "localhost:1521/XE")
    sql = """ select g.GOOD_SEQ, g.GOOD_NAME, g.GOOD_IMG, g.GOOD_DESC
                          , c.GOOD_PRICE,  c.ORDER_AMOUNT
                          ,  (c.GOOD_PRICE*c.ORDER_AMOUNT) as TOT_PRICE
                          , c.CART_SEQ , c.TEL
                    from KIO_GOODS g,  KIO_CART c
                    where g.GOOD_SEQ = c.GOOD_SEQ
                          and TEL = :1
                """
    cur = conn.cursor()
    cur.execute(sql,  [session['MY_TEL_SESSION']])
    goods_list_dict = []
    for row in cur:
        # [3, 'CRISPY CHICKEN', 'static/images/menu/burger-11.jpg', 11, 'Chicken breast, chilli sauce, tomatoes, pickles, coleslaw']
        dic = {}
        dic["GOOD_SEQ"] = row[0]
        dic["GOOD_NAME"] = row[1]
        dic["GOOD_IMG"] = row[2]
        dic["GOOD_DESC"] = row[3]
        dic["GOOD_PRICE"] = row[4]
        dic["ORDER_AMOUNT"] = row[5]
        dic["TOT_PRICE"] = row[6]
        dic["CART_SEQ"] = row[7]
        dic["TEL"] = row[8]
        goods_list_dict.append(dic)
    cur.close()
    conn.close()
    return goods_list_dict


@app.route('/add_cart',  methods=['POST'])
def add_cart():

    v_good_seq   = request.form.get("good_seq")      #hidden
    v_good_price = request.form.get("good_price")  #hidden
    v_amount     = request.form.get("amount")        #수량
    print("카트담기")

    conn = cx_Oracle.connect("ai", "0000", "localhost:1521/XE")
    sql = """ insert into
        KIO_CART(CART_SEQ, TEL, GOOD_SEQ, GOOD_PRICE, ORDER_AMOUNT, REG_DATE)
        values(KIO_CART_SEQ.nextval, :1, :2, :3, :4, sysdate)"""
    cur = conn.cursor()
    cur.execute(sql, [ session['MY_TEL_SESSION'], v_good_seq, v_good_price, v_amount])
    conn.commit()
    cur.close()
    conn.close()

    goods_list_dict = mydef_cart_list()
    return render_template('cart.html'
                           , KEY_GOODS_LIST_DICT = goods_list_dict)


@app.route('/cart_delete',  methods=['GET'])
def cart_delete():
    v_cart_seq = request.args.get("prm_cart_seq")
    conn = cx_Oracle.connect("ai", "0000", "localhost:1521/XE")
    sql = "delete from kio_cart where cart_seq = :1"
    cur = conn.cursor()
    cur.execute(sql, [v_cart_seq])
    conn.commit()
    cur.close()
    conn.close()

    goods_list_dict = mydef_cart_list()
    return render_template('cart.html', KEY_GOODS_LIST_DICT=goods_list_dict)


@app.route('/orders',  methods=['POST'])
def orders():
    v_price = int(request.form.get("order_price"))

    print("주문완료")
    conn = cx_Oracle.connect("ai", "0000", "localhost:1521/XE")
    sql = """insert into
        kio_order(ORDER_SEQ, TEL, ORDER_PRICE, PAY_GUBUN, REG_DATE)
        values(kio_order_seq.nextval, :1, :2, '1', sysdate)"""
    cur = conn.cursor()
    cur.execute(sql, [session['MY_TEL_SESSION'], v_price])
    conn.commit()
    cur.close()
    conn.close()

    session.pop('MY_TEL_SESSION')

    return redirect("/")  #메인화면으로 가기


@app.route('/test')
def test_list():
    test_list = [1,2,3]
    test_dict = {"uid":"kim", "upw":"111"}
    test_list_dict= [ {"uid": "kim1", "upw": "555"},
                      {"uid": "kim2", "upw": "666"} ]

    return render_template('test.html'
        , KEY_TEST_LIST = test_list
        , aaa = test_dict
        , KEY_TEST_LIST_DICT = test_list_dict
                           )


@app.route('/test_prm_get', methods=['GET'])
def test_prm_get():
    id = request.args.get('DDD_ID')
    pw = request.args.get('DDD_PW')
    print(id, pw)
    return id+","+pw

@app.route('/test_prm_post', methods=['POST'])
def test_prm_post():
    id = request.form.get('DDD_ID')
    pw = request.form.get('DDD_PW')
    print(id, pw)
    return id+","+pw



if __name__ == '__main__':
    app.debug = True
    app.run(host='0.0.0.0', port=80)
```

<목표> 

(현재 0505로 하드코딩해서 '주체'를 고정한 상태)

장바구니에 담은 수량, 종류, good_seq보여주기 → 결제하기

index, product-single, cart → html 수정하자 

cart.html에서 delete를 통해 cart item루프 찾고

{for dic in ~}: 딕트에서 ~를 꺼내 → ex) dic.GOOD_IMG

오라클함수에서 delete사용하여 삭제 구현 → 다시 카트.html

그러려면, 함수를 활용해야함(반복되니까)

cf. 파라미터 전달받을 경우 "글자"로 인식하기에 int()처리해줘야함

------

###  jinja문법

```html
{% set ~ = 0 %}  선언


```

### random 문법

```python
import random
aa = random.radint(x, y) # x~y사이에서 랜덤 숫자 뽑기
print(aa)

tel = str(random.randit(10,99)) + str(random.randint(10,99))
print(tel)
```

인덱스가 처음 열릴 때, 랜덤번호가 발급되는 논리 → 핸드폰 번호처럼 생각해서 회원 / 비회원을 구분하는 역할

------

### 플라스크 세션을 활용하자(쿠키, 세션)

매번 파라미터를 넘길 필요 없이 세션을 시작하고 종료함에 따라 주체의 구분이 가능해짐

----------------------------------------------------------------------------------------------------

꺼낸다 = 해당 컬럼명을 sql = """ """ 에 추가

탬플릿 html 수정시 크롬 f12에서 검사를 통해 어디를 위치하는지 확인

----------------------------------------------------------------------------------------------------

경로: 첫 화면(매장/포장)(전화번호발급) →인덱스 → 디테일 → 장바구니 

0505 하드코딩한 것 수정해야

### 포트폴리오를 위해 항상 f12에서 웹화면 + 모바일화면 프린트스크린 해놓기(반응형 UI/UX): 키오스크 개발

포트폴리오를 화면캡쳐해서 모아놓는 습관 들이자

------

### 웹을 배우는 이유

타인에게 showing해야 하기 때문에

------

내일부터는 데이터 프레임 핸들링(변수파트 공부하기)

프로세스만이라도 이해하도록 노력하자(면접때 활용가능하니까)

------

# 2월 7일

doc: 랭귀지 만든 사람이 작성한 사용법

pandas doc

https://dataitgirls2.github.io/10minutes2pandas/

https://pandas.pydata.org/docs/user_guide/10min.html

오늘은 데이터 분석!

------

## numpy와 pandas 준비

넘피가 좀더 큰 범주라서 판다스부터 시작

(파이참)jupyter 패키지 설치: 한줄한줄 결과보여주는 패키지

(파이참)jupyterlab도 같이 설치하기

(파이참)numpy 설치(부모)

(파이참) pandas 설치(자식)

requests 설치해야하는지,,,?

------

colab: 인터넷에서 주피터역할, 인터넷에서 쓰는 파이참

.ipynb: 주피터 확장자, 주피터에서만 열 수 있음

interpreter: 한줄한줄 프로그램 명령을 읽고 해석

코랩은 무료, 단 끊김이 있는 편, 프로젝트할때는 주피터 유료결제해라

! pwd : (리눅스) 현재 경로 실행

항상 구글드라이브에 마운트해서 작업해라

------

## pandas 입문

파이참 venv설정 해놓기

파일 - 세팅 - 툴즈 - 터미널

shell path: powershell -NoExit -File ".\venv\Scripts\activate.ps1" 입력

default tab name: Local 로 설정하기

★실행시, 하위폴더만 볼 수 있음 

------

Series: 1열 (1-dimensional) 세로 한 칸!

dataFrame: 행렬(2-dimensional)

NaN: 결측 데이터(null)

가능: Insert, delete, 정렬, group by, 형 변환, slicing, indexing, merging, joining, reshaping(행과 열을 바꾸는 개념), 시계열 데이터 위주

SAS: 통계패키지프로그램

------

 ">" : 인용문

### tex문법

수식넣기: $수식$

### 판다스문법

dtypes: 타입 / 리스트 같은건 object

to_numpy: 배열타입으로

#### dataFrame(리스트타입, 넘피배열, 딕셔너리)

#### 값을 가져올때: 리스트, 슬라이싱, 단일값

슬라이싱은 ~이상 ~미만이지만, 값은 ~값부터 ~값까지(포함)

여러값을 가져올 때는 리스트로

: 데이터 전부

#### df[df조건식]]

isin: 있니없니

값으로 꺼낼때 loc     -> 리스트, 단일값, 슬라이싱

~번째 iloc

정렬: sort index / s.value_counts



drop

fill

isna 위와 용법 다름

mean: 평균

apply: 함수나 기능을 적용

lower,upper 함수는 str로 빼서 사용할 것

concat: 합치다

merge: join

categorical: 글자로 분류하는 경우

그룹핑 한 후 개수는 size

녹색글씨는 예약어, list 이런거 그대로 쓰지마

object : 글자를 의미

array는 리스트에 담은 후, np.array로 출력하는 개념

shape: 몇행 몇열인지 (5,) = 1행 5열 array(배열)은 행열 순으로 개념 but 출력은 반대로 

[1, 2, 3] 리스트

[1 2 3 ] array, 배열 = 행렬!!!!

리스트만 shape이 불가!!

shape은 함수가 아님 --> 뒤에 괄호 붙이지 마

mylist.shape (X)

df.info : 정보를 확인

dataframe의 정보를 확인하는법: info(), describe(), head(), tail(), shape 은 괄호 없음

------

# 2월 8일

오늘은 dataframe의 select문법인 loc(줄, 칸) iloc(줄, 칸) 에 대해 배울 것

### 문법

loc(줄, 칸)  ---- 단일값, 리스트, 슬라이싱

df1[조건절] --- df[df1['A']>20 and ~ ]

df1['컬럼명']

셀 실행이 꼬이면 커넬 - 리스타트 커넬 앤 런 올 셀 선택

------

### 데이터처리시

데이터불러오기 - 이상치(범위에서 벗어난 값)/결측치 처리가 가장 중요한 문제

이를 잘 처리해야 학습이 잘 된다

------

### 괄호닫기 실수 조심

기본구조 쓰기 emp[][]

가운데 들어갈 조건 하단에 쓰기

조건 붙여넣기

왜 배움? 데이터 담고 수정하고 그 역할부터 하겠지

------

# 2월 9일

조회: emp.iloc[0:3, 0:2]

수정: emp.iloc[0:3, 0:2 ] = 값

df.iloc[ : , ~~ ]  ----- 열만 뽑으려면



# 2월 10일

ignore index: 인덱스가 서로 다른 경우, 뒤의 인덱스를 무시하고 앞에 맞춰라

axis = 1 : 옆으로 합쳐라

axis = 0; 눈사람 생각(위아래로 합쳐라)

full outer join: 나한번 기준, 너한번 기준으로 결측채우고 합친다

inner join 해당 인덱스가 일치하는 지점을 찾아서 조인함(컬럼은 문제되지 않고 옆으로 붙는는다) → 키가 맞는 곳을 합친다

조인보단 머징

left_on / right_on: 다를 경우 어떤 것을 위치 시킬지

```py
pd.merge(
    left,
    right,
    how="inner",
    on=None,       컬럼이 같은 경우
    left_on=None,  컬럼이 서로 다를 경우 어떤 것을 오른쪽/왼쪽에 위치?
    right_on=None,
    left_index=False,
    right_index=False,
    sort=True,
    suffixes=("_x", "_y"),     첫번째컬럼: _x
    copy=True,
    indicator=False,
    validate=None,
)
```

https://pandas.pydata.org/docs/user_guide/merging.html

left  LEFT OUTER JOINright RIGHT OUTER JOINouter FULL OUTER JOINinner INNER JOIN

행렬 채로 데이터프레임화 → 이중리스트로 삽입가능

https://www.kaggle.com/ 코드 타이핑 연습하기



value_counts()는 groupby와 유사함(종류별 정렬)



# 2월 13일

결측확인 df.isna()

groupby에서는 함수명을 넣을때 " "로 넣는것잊지말기

concat: [리스트]로 나열

merge: 2개만 나열

타입변경: 컬럼 = 컬럼.astype

이번주 목표: 데이터 수집 - 분석 - 시각화

팀 인원, 주제 자율임

데이터 분석시 항상 결측부터 확인 → 확인된다면 무엇으로 채울지?

데이터의 쏠림현상: 왜도

inplace=True(제대로 지워졌는지): 덮어 씌우기

print(train.group()): 정확하게 매칭된 글자만 빼올 수 있음



# 2월 14일

머신러닝시: 결측 없어야, object없어야 (컴퓨터는 글자를 학습할 수 없다)

분석이 끝날때까지 컬럼은 함부로 지우지 마라

정규분포를 형성하기 위해서는 최소 30 이상(큰수의 법칙)

메타데이터: 데이터를 위한 데이터, 예를 들어 pclass를 카테고리화 하면 code가 뭘 의미하는지	 알아야하니까

# 2월 15일

# 2월 17일

임포트 4개는 무조건: numpy, pandas, matplotlib, seaborn

파이썬통계모델: statsmodels, scipy

------

## 통계

범주: 질 - 글씨(categorical)

연속: 양 - 숫자(numerical)

![Image20230217120629](C:\Users\ASIA\Desktop\이미지\Image20230217120629.png)

이산적변수 ex) 남자1 여자0 (수치데이터가 더 나올수 없는 경우)

연속적변수 ex) 온도

KDE: 확률의 범위에서 추정

여기서 밀도란, AI에서는 "분포"를 의미

확률분포표에서 면적이 확률값을 의미(적분)

모수값(평균, 표준편차) → 분포의 모양이 달라짐

커널함수: 주어진 데이터로 정확히 모르더라도 정규분포를 활용하여 밀도를 추정하는 것

(라인플로우을 추정하는 것)

왜 log를 씌우면 그래프가 다듬어지는가? → 스케일을 확 낮추니까(지수의 단위로만 그려지니까) → 정규분포화 시키기 위해서

즉, 로그를 씌우면 정규분포화되어서 추정이 쉬워진다: scaling

상관계수: 상관관계 분석에서 두 변수 간에 선형관계의 정도를 수량화하는 측도

분산(표준편차): 내 데이터와 내 데이터의 관계(ex) 평균에서 얼마나 떨어져 있는지?

공분산: 2개 feature의 관계(나와 다른 피쳐와의 관계)

------

ex) 키 140 -> 150 / 키 110 -> 120 누가 더 많이 성장?

→ 비교를 위해서는 표준화가 필요: 1. 특정 비율화(단일화) 2. 비교  --> 1. 상관계수 2. 상관분석

# 2월 20일

target feature: casual, registered, regcount

파생 feature: y, m, d, h

# 2월 21일

barplot의 원리: groupby 한 평균을 그릴때

countplot의 원리: record의 counting(= value_counts)

버리기: train.drop(del_idx_list, axis = 0, inplace=True)

데이터 학습 전, 어떤 feature가 영향도가 클까?

박스 플롯에서 median의 위치를 통해서 판별할 수도 있다(중간값을 이은 선이 평탄하다면 큰 영향 자체가 없다는 의미)

* 다중공선의 문제: 상관계수가 너무 높은 피쳐가 들어가면 과대평가될 수 있음 ---> 제거 후 분석 시작해야

데이터 시각화가 중요한 이유는 무엇을 학습에 적용시킬지 '판단'하는 과정이기에

크로스 도메인(로컬호스트와 구글호스트)은 불가능



Rest(ful) [ 분산서버를 둔 서비스 방식 ]

1) 비동기/동기
2) 외부의 자원(지도, 위치정보, 실명인증결과값 등)을 내 것처럼 쓰는 것 

3. only 웹 형태로만 요청/응답 가능
4. 텍스트, json형태로 주고받음



cf. URI 방식으로 파라미터를 주고받을 수 있다(△)

![json](C:\Users\ASIA\Desktop\이미지\json.png)



# 2월 22일

교수님 구글 api키

api 키: AIzaSyAeBE8_KfYLWXqM39CDsxyNaOiu6ej0Jv8

d86303d03401fb083ae1be14355863bb294f26d2

https://python-visualization.github.io/folium/quickstart.html#Getting-Started folium api

from folium import plugins  --> plugins.Markercluster

dart(출처, 시간, row col을 명시할 것)

json(자바스크립트에서 쓰는 객체 표현) = dictionary

{"key" : 1111}



# 2월 23일

```
{% for dic in KEY_MYDATA %}
    {{dic["KID"]}}
    {{dic["KPW"]}}<br>
{% endfor %}
```



cf. rest api 시 오류 종류

400번대: 서버 자체에 진입을 못하는 오류(보통 주소 잘못 입력)

500번대: 서버에 진입했지만 api의 오류

requests: 통신을 위한 파이썬 라이브러리

print(변수)를 생활화하자

데이터프레임을 만들기 위해서 append보다는 concat을 활용해라

읽어들이는 것은 cx_Oracle

쓰는 것은 sqlalchemy





# 2월 24일

문제가 될 상황에는 except(예외처리)를 해야한다

exception을 잘 활용해야 한다(셀레니움)

목표: 설령 막히더라도 전체 프로그램이 구동되기 위해서



##  셀레니움

마우스 우클릭 - 검사 - xpath로 찾기





비동기통신: ajax

화면은 움직이지 않으면서 화면이 변화



## plotly

차트 & 데이터프레임 함께 저장(차트 스튜디오에)

동적 차트

https://chart-studio.plotly.com/settings/api#/

차트스튜디오에 만들어서, 필요한 곳(웹)에 embed할 것(html)

또한 데이터베이스에 바로 수정도 가능함

plotly 차트를 bootstrap 대쉬보드에 끼워넣는 것은 가능함( 대쉬보드끼리 섞을 수는 없음)









# 3월 3일

SAMPLE_df_json_str.py

serialization : 직렬화

데이터프레임을 덤프나 로드를 하려면 json으로 바꿔야 하는 이유!!

ex) id: pw --->  딕트나 리스트 등 한 줄로 핀다음 그 모양의 글자를 만들어야 함

load: 스트링 ---> json: 다시 딕트화

* 딕셔너리로 바꾸기 위해서는 to_json()
* 싱글쿼트, 더블쿼트 주의
* 리스트에 리스트를 append하면 가장 데이터프레임하기 쉬운 구조





# 포트폴리오 정리법

프로젝트명: 제주관광 빅데이터 GIS시각화 서비스 개발

### 적용기술

- IDE: colab, pycharm, jupyter lab
- DB: oracle 11g exp.
- API: 비짓제주관광 openAPI / 공공데이터포털 openAPI
- python pkg
  - pandas, numpy, json, sqlalchemy, Cx_Oracle, Beautifulsoup4, selenium, flask
- 시각화 pkg:
  - matplotlib, plotly, seaborn
- web Framework
  - Flask, Bootstrap



### 주요개발기술

- RESTful 서비스
- GIS 시각화
- Oracle DataFrame 연동 빅데이터 처리
- 데이터 수집(정형: RDB, 준정형: REST JSON, 비정형: 웹 크롤링)







# [머신러닝]

sklearn: 회귀, 분류, 군집

pycharm: tensorflow(2.11.0), keras 설치

머신러닝은 대량의 데이터 학습 불가, 딥러닝은 가능

## 머신러닝: 사람이 규칙을 만들어 학습 → 적은데이터로 학습가능, but 기계에게 직접 지정해줘야

### 특징추출

특징 추출을 위한 알고리즘을 인간이 직접 제공

해당분야에 대한 지식 및 직관, 알고리즘 구축을 위한 상당한 노력 필요

### 분류

컴퓨터 학습의 영역

### 출력



## cf. 딥러닝: 종단간 기계학습

데이터로부터 학습(예를 들어 계속 사진을 보여주면, 컴퓨터가 스스로 패턴을 찾아냄, neural network)

어린 아이에게 학습시키는 느낌(black box)





## supervised   vs   unsupervised

정답을 알려주고 학습 vs 정답을 알려주지 않고 학습

답이 있는 경우: <u>회귀(모델링이 어려움, 통계필요) / 분류</u>

답이 없는 경우: <u>클러스터링(군집)</u>





## 회귀와 분류

회귀와 분류의 공통점: 답안이 있다

회귀: 연속된 수치, 또는 부동소수점수(실수)를 예측하는 것 예측 출력값 사이에 연속성이 있다

분류: 이산형(분리된) / 미리 정의된 여러 클래스 레이블 중 하나를 예측하는 것

**★★타겟이 명확해야 회귀인지 분류인지를 결정할 수 있다**

ex) 경마 우승말이 무엇일지?  /  1번 말의 우승확률이 얼마인지?



## 일반화

모델로 훈련 데이터셋을 훈련하고 테스트데이터셋을 정확하게 예측하는 것

적절한 fitting 하여 학습하는 것이 필요하다

과대적합: 가진 정보를 모두 사용한 너무 복잡한 모델, 훈련데이터셋에 너무 가깝게 맞춰져서 테스트데이터셋에 일반화되기가 어려움

과소적합: 너무 간단한 모델, 데이터의 다양성을 잡아내지 못하고 훈련데이터셋도 잘 맞지 않음

과대와 과소 모두 피해서 일반화 성능이 최대가 되도록 분석



## 과정

![image-20230306121455625](C:\Users\ASIA\Desktop\이미지\image-20230306121455625.png)

1. Problem Definition: 문제부터 정의해라
2. Problem Feature: 피쳐선정
3. Select Framework:뭐로 개발할지 정의
4. EDA: 데이터 살펴보자(수집, 가공, 시각화, 전처리)
5. Feature Engineering: 피쳐 가공 → 데이터 원본에 변형을 가하는 것 ex) 결측을 채우는 것, 결측을 지우는 것, 남여를 0,1로 채워라 → 모두 숫자로 채우기
6. Model Selection: 모델 선정하기, 대여섯가지 가져다 돌려봐야 
7. Apply learning: 모델을 사용하여 데이터를 학습시키기
8. Model Maintenance: (유지보수) 검증, 이전데이터에 앞으로의 데이터를 계속 학습, 이에 모델결과는 달라질 수 있다

※ 예를 들어 17일치 학습이 끝난 데이터는 모델을 저장한다 / 그 다음 다시 학습 



## 머신러닝은 확률게임이다

정규분포화 및 확률밀도함수 개념

보통 현업에서는 예측도 43%가 1등 (70~80%면 overfitting 가능성이 높다)



## EDA

1. Data Collection: 변수확인

2. Data Preprocessing: 데이터 전처리

   - 데이터 셋 확인

   - 결측값(missing value) 처리: 삭제(행지우기, 컬럼지우기)

     / 다른 값으로 대체(평균, 최빈값, 중간값 - 외도가 발생할수도)   

     / 예측값 삽입(현업에서 가장 많이 사용) → 회귀분석, KNN

   - 이상값(outlier treatment) 처리

     - 이상값 찾기: Boxplot, Histogram, var-n, 4분위(IQR), Scatter(가장 확실)
     - 이상값 처리하기: 단순 삭제(오타 등), 보통 다른 값으로 대체
     - 변수화: 자연 발생한 이상값의 경우, 바로 삭제하지 말고 세부적인 파악이 필요
     - 리샘플링: 이상값을 분리해서 모델을 만드는 방법
     - **★★케이스를 분리하여 분석★★**: 이상값 포함 모델, 이상값 제외 모델을 모두 만들고 각 모델에 대해 설명

​		분류: input 값의 갭차이에 큰 영향이 없다

​		회귀: 들어오는 값(input)에 따라 y가 영향을 받기 때문에 인코딩 중요

3) Data Cleaning

   - Transforming Features(피쳐 가공)
     - A. Scaling: 단위 변경, 편향된 분포 변경(표준화, 정규화, 로그함수화)
     - B. Binning: 나이 구간화(연속형 → 범주형)
     - C. Dummy: 바이닝과 반대, OneHotEncoding
     - D.Transform: 변수의 성질을 이용해 다른 변수(FC, PCA)를 만드는 것 → 두 개 feature를 하나의 feature로 바꾸는 것(단순 덧셈 X)

   - Feature Encoding

4) Visualization



cf. SGD: 머신러닝 중 그나마 큰  데이터를 처리할 수 있는 모델

cf. label이 없으면 군집(clustering)

cf. 예측의 개념은 accuracy(정확도)와 +a가 필요하다





w(가중치): 기울기(가중치)가 크면 클수록 주요피쳐이다

x값 변화시 y의 변화량이 커지기 때문

주어진 데이터를 가장 잘 설명하는 함수: x를 가장 잘 부합하는 w를 찾는 것

비용(loss): 분산, 표준편차와 비슷한 개념 → 비용함수를 통해 구한다

## ★★분산 및 표준편차가 최소화되는 좋은 모델이다 

**→ 이 모델의 기울기, bias를 찾아야 한다** : 회귀의 주 목적

→ 모델 파라미터: w와 bias를 찾는 것(기울기, y절편)



cf. 다차함수: 모델이 복잡할 가능성이 높음 → overfitting가능성



## fitting: 학습하다

### test가 맞는지 검정하려면?

train을 7:3, 8:2(x_train, x_test, y_train, y_test)로 잘라낸다(train_test_split, 양, 비율 모두 가능) → 답지를 숨긴다 → 잘라낸 3, 2를 맞춘다 → 떼어논 답안지를 통해 내 예측모델의 점수 확인



코드) from sklearn.model_selection import train_test_split

x_train, x_test, y_train, y_test = train_test_split(x, y, random_state=0)

분석을 위한 알고리즘 선택

#### ★★알고리즘은 분석에 파라미터=보정값(하이퍼 파라미터)를 tuning하여 최적의 점수를 도출해야 함

from sklearn._____ import ~algorithm~

~~ = ~~ algorithm ~~

train 문제/답 훈련

~~.fit(x_train, y_train)

test 문제

pred = ~~.predict(x_test)

test 문제에 대한 정답률: 일반화 정확도

~~.score(y_test, pred)

★★ 모델의 파라미터를 tuning하려면 수학, 통계적 지식이 필요







## 1. Supervised learning

1. Linear Model: 선형모델

2. support vector: y=x와 떨어진 거리를 통해 분류(회귀 분류 모두 지원)

3. Nearest neighbor: 이웃노드, 이웃을 보고 '나'를 판단

4. Naive Bayes: 모 아니면 도, 사망 or 생존

5. Decision Trees: 의사결정트리, 분류에 주로 사용, 평균에서 편차가 얼마인지 판단해서 분류하기도 함, 분산을 활용하기도

6. Ensemble methods: 저급의 모델을 묶어서 강한 모델을 만드는 것

   ex) 의사결정시 여러 모델을 두고 투표하는 방식을 사용하기도 함(강력한 모델에 weight가중치를 많이 두고)

7. Feature selection

8. Neural network

9. Semi-supervised: 데이터가 극단적으로 imbalanced인 경우 →경계치에 있는 값도 포함시켜서 반복 학습



## 2. Unsupervised learning(비지도학습)

1. Neural network
2. Clustering
3. Density Estimation: 밀집도 기반으로 clustering



## 3. model selection and evaluation

validation: 검증 - 모델을 학습, 평가 후  '검증' 단계를 거쳐야 함 → 내 평가모델의 당위성을 확보하기 위한 작업



## 6. Dataset transformation

imputation: ex) 결측에 평균 등을 넣어라 등 가능

pipeline



## 7. Dataset loading utilities

Toy: 연습을 위한 예시데이터

Real world: 실 데이터



## 8. Computing with Scikit-learn

빅데이터 처리는 현실적으로 어렵다



## cf. 분류 vs 군집

답지가 있는지 없는지?(supervised vs unsupervised)



------

회귀:  MSE(분산), rMSE(표준편차) → 편차의 1 차이에도 패널티가 큰 편이다

ex) (4-2)^2 → (4-3)^2

분류: Binary cross-entropy, Categorical cross-entropy

# 회귀

**목표: accuracy가 아니라 loss를 계산해야 한다**(accuracy는 정확히 그 값이 일치하는가의 개념이니까 accuracy로 접근하면 걍 빵점 나오겠지)



## MSE

- loss 가 조금만 나도 더 error의 크기가 크게 출력될 수 밖에

- 예측과 실제값의 차이를 제곱하여 "뻥튀기된다"

​		→ 이상치 데이터에 민감하게 반응하는 metrics

​			(정규분포 내 데이터는 문제 없음)

​		→ 공모전에서 평가에 활용될 수 밖에

- 순간기울기를 구할 수 있다(2차 함수 그래프니까)

​		(RMSE는  미분값을 구할 수 없음)

- RMSE는 이상치 데이터에 상대적으로 덜 민감하게 반응할 것(즉 공모전에서는 보기좋은 점수)

- MSE의 단점? loss가 너무 크니까 머신러닝이 그 피쳐에 민감해진 상태로 학습이 시작 → loss를 줄이려다 보니 오버피팅이 발생할 수 있음
- MAE와 달리 페널티(가중치를 반영하여)를 반영하기 때문에 outlier에 더 민감하게 반응하게 된다



## MAE

- 절대값의 개념
- outlier에 상대적으로 덜 민감
- 오차에 대한 페널티를(가중치를 부여해서 반영 x) 동등하게 반영



## 결론

 RMSE와 MAE가 유사하다면, 예측을 많이 정확하게 한 것(outlier가 있다 하더라도 근사치로 정확하게 맞출 수 있다)



## RMSLE

- 에러에 로그스케일링 후 제곱 후 평균 후 루트

cf. 예측함수에 log → 값이 크기 때문에 스케일링

​     +1 하는 이유? 0에 너무 수렴하면 under flow현상 발생

------

수학기호 넣으려면 Latex 문법

시계열 데이터는 regdate를 인덱스로 그대로 넣어도 된다

인덱스는 학습에 영향 미치지 않음

reset인덱스를 할 때는 한번만!

------

타겟 피쳐: train과 test의 차이나는 피쳐 → regcount만 (casual + registered)

다중공선(겹치는 데이터로 인해 상관성이 1로 거의 일치할 경우)은 없애야 한다

~~OneHotEncoding~~: 숫자화 + 평준화 작업 → '차등'이 들어간 개념에서는 쓰면 안된다

중앙값은 outlier의 영향을 받지 않는다(평균은 받는다)

------

분류는 상관없지만 

회귀는 스케일링이 필수!

------

ref: https://hsm-edu.tistory.com/756

적률: 그래프의 모양을 나타내는 척도

평균: 1차적률

분산: 2차적률

왜도(Skewness): 좌우로 찌그러진(outlier), 왼쪽으로 쏠리면 (+값), 오른쪽 쏠리면(-값) [3차적률] 쏠림이 클 수록 수치가 커진다(outlier가 많다), 0~10은 정상범주

첨도(Kurtosis): 날카로운 정도 [4차 적률]

https://blog.naver.com/s2ak74/220616766539

첨도>0 이라면 왜 꼬리가 두껍고 길어지는가?

분산의 크기를 통해 평균으로부터 얼마나 더 쏠렸는지 확인 가능

중앙값과 평균값의 관계를 통해서도 얼마나 쏠렸는지 확인 가능

중앙값과 평균의 거리가 멀다, 왜도의 (+)(-) 확인 → (평균을 올리는) 매우 큰 데이터가 있을 확률(+ 중앙값과 평균과의 비교)이 높다

------

### 확률분포(도)를 따른다

확률에 따라서 기대값을 말하는 것

이산형: cdf

연속형: pdf



프로파일링: 데이터 분석 및 시각화, 통계테이블

------

## 트리모델

![분류회귀](C:\Users\ASIA\Desktop\이미지\분류회귀-1678863038834-2.PNG)

분류 vs 회귀

depth(가지치기의 단계단계) 하나 당 feature 하나

1)

2) 가지치기를 어디까지? → 분류가 잘 된건지 아닌지(hyper-parameter tuning)

엔무지순: 엔트로피 무질서 지니계수 순도(순도가 높으면 지니계수가 낮다, 순도가 낮은 쪽에서 가지치기가 발생)

즉, tree는 지니계수를 낮추기 위해

지니계수가 낮을수록 순수하다



엔트로피 수치가 크다 = 정보량이 많다 → 분류가 가능하다

매번 일어나는 사건의 정보량은 적다

잘 일어나지 않는 사건의 정보량은 크다(특이성 데이터가 섞여 있으면 정보량이 많다)



결론) 엔트로피, 지니수치가 큰 것은 분류대상, tree는 작게 만들려고 함

------

RF, LGBM 등 bagging 기존 트리모형은 속도는 빠름(병렬)

- 하나의 모델에 다양한 데이터



XGBOOST등 boosting원리: loss를 최소화하려하면 over fitting

- 앙상블효과, 순차적, 동일한 데이터에 점점 쎈 모델 적용(트리를 바꿔가며, 가중치를 바꿔가며)



Voting

- hard: 가중치가 없는 투표, 특별한 경우
- soft: 가중치를 둔 투표(가중평균), 대부분 투표에서는 soft가 디폴트

------

# pycaret

앙상블모델: 배깅, 부스팅

블렌드모델: voting포함

cf. 트리모델: 잘 맞춘 것은 weight 는 줄이고, 잘못 맞춘 가중치는 더 키운다(패널티를 더 준다)

★ predict_model( ) → 사용하지 말 것



## PCA:Principal Component Analysis

주 성분 분석: 행렬의 내적 분해(고유값 분해) → 2차원 벡터선(고유선) 도출 가능

#### 차원축소: 분산이 최대화되도록 벡터선 도출

직교(90도)되는 선을 계속 도출하여 찾는다 → 보통 PC3 까지만

분산이 최대화되도록 방향을 설정해야 가능한 많은 점이 찍힐 것 → 각 점들을 최대한 많이 거치도록 방향을 설정해야 함

투영: 고차원에서 저차원 공간으로 위치시켜 데이터 셋을 만드는 것

차원을 줄인다는 것은 예측력을 올리는 것, 모델의 복잡도가 떨어져서 점수 자체가 낮게 도출될 수 있음

일반화 = loss를 줄일 수 있다

grid: 바둑판, 격자

catboost model: 카테고리성 데이터 부스트

커널: 확률밀도함수를 구하는 함수

# logistic regression 분류(classifier)!!!

파이캐럿에서 튜닝된 점수, 모델을 사용하여 직접 타이핑하면 더 빠르게 확인 가능(장점)

------

macro평균: 산술평균

micro평균: 가중평균

통계와 머신러닝은 실제-예측 축이 다름을 주의하자

------

precision과 recall의 점수가 다 높아야 조화평균인 f1값이 높아진다

------

random_state라도 바꿔서 낼 것

dacon: 분류 사례임

------

공모전 하다가 안된다면?

#### 임계치 수정

#### 랜덤시드 바꾸기

dacon lec07은 카드패턴을 보고, annormally detection(이상치 탐지)

불균형 데이터를 처리하는 것이 목표



PCA → outlier처리가 되었다는 의미

time, amount 피쳐는 버리는게 나음



## 판별함수 조건

loss가 일정

분산이 어느정도 일정(선형관계여야)

다중공선이 없어야

아웃라이어 제거





## over sampling vs under sampling

둘다 해봐야 된다 그래야 뭐가 맞는지 아니까



------

데이콘 전화데이터

타겟: 전화해지여부

decribe()로 나열해서 비교하기

#### 항상 0과 1 고객을 같이 비교해야

EDA

목표

1) '저녁통화시간min에서 차이가 커보임' → 해지하는 놈들만 하는 짓이 있다(20건 이상의 이상한 짓) → 고객센터 전화...?

→ 통화시간과 통화요금은 상관성이 있는데, '밤'시간이 더 기울기가 급격하다



2) pairplot 확인

3) SMOTE이나 undersampling, oversampling 전후 확인하기

threshold=0.26

------

교수님 방법

1) eda
2) sampling(10가지 정도 하면서 최적 값 도출 )
3) pycaret 모델 셀렉션, 튜닝

양성비율(churn rate): 고객이탈율 = 전체 고객 중 고객이 이탈할 확률(전체 중 p가 얼마?)

고객이탈율을 지수화해서 시간 변화시 어떻게 변화하는지 보고자 함

가입기간을 년, 월, ~~주, 일~~로 쪼개보기

통화요금: 주간 / 저녁/밤의 패턴이 다르다 → 요금제가 다를듯?, 요금이 고객이탈요인 중 핵심

cf.통신사 변경 사유: 요금!! 1등

요금제와 관련한 고객들이 이탈가능성이 높음

박스플롯 그려보면 상담전화건수가 확인하게 다르다

가입을 오래 한 애들은 0에 많음 

가입기간 년수가 높으면 0이 많다(장기고객 가능성, 다중공선 가능성)

장기고객은 아웃라이어로 버리면 안된다

단위시간당 요금이 필요함 → 주간 / 저녁 / 밤 쪼개고 나머지는 drop



정 안되면 군집을 시도해야 함

PCA를 통해 군집화한 후 차원축소를 하면 분리된 모습을 만날 수 있다

총 통화시간이 낮은데 시간당 요금이 높다? → 게임을 하던지 다른 서비스를 이용할 가능성이 높은 사람

로테이션(5~6개월 당 해지가 발생)









주간바보 → 저녁, 밤에도 바보인가



가입일을 3구간으로 나누기



가입일(구간나누기)과                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        

1) 주간, 저녁, 밤 요금  
2) 상담전화건수

1번 csv: 소중한우리피쳐 {'criterion': 'gini', 'max_depth': 50, 'n_estimators': 200} threshold=0.28 [10]

------



점수를 잘 받는 근본적인 방법은 제대로 파생변수를 생성하는 것

→ 예측력이 확연히 좋아지는 가장 빠른 방법

but 그게 안되니까 모델 고르고 이런짓 저런짓 하는 것

------

군집을 만들경우 좋은점수 받으려면?

1) 그룹간 거리가 멀어야
2) 그룹내 응집도가 높아야

오버피팅만 피하자!!(원본이 점수가 좋은 상황... 오버 샘플링은 좋지 않다)

------

통화(특정피쳐만)에 대한 건만 클러스터링

아웃라이어 처리

비율 맞춰서 오버샘플링

------

화이트 vs 레드와인, 타겟: 퀄리티

화이트와 레드와인의 분포가 다른 것처럼 보인다? → 화이트와 레드와인을 구분하고, 모델을 다르게 학습하자

------

시계열 회귀: x축이 날짜데이터, y축은 그 이외

(평균, 표준편차)가 매우 중요

ex) price를 예측해볼래?, 강수, 에너지 전기 등등

header: 0번째 라인 의미

------

타겟: RM(0.75) LSTAT(-0.75)
TAX, RAD: 다중공선 관계 → MEDV와의 관계를 보자 → TAX(더 영향이 높다) 삭제 OR 피쳐를 하나로 통합

------

R^2: 결정계수, 모델 설명력

r^2: 1에 가까우면 설명력이 좋음, 0에 가까우면 설명력 낮음

1이라면 SST = SSE  즉 전체를 모델이 설명 다 할 수 있다

------

RMSE는 큰  오류값 차이에 대해 크게 패널티를 주는 이점이 있다

------







# NLP 과정

-----------------------------

전처리(8) 
-----------------------------

 - 문장/단어 토큰화
 - 정제/정규화 : 불용어, 렘마, 스티밍
 - 원핫/라벨 인코딩 : 수치메트릭스,패딩

-----------------------------

워드임베딩 (단어->숫자 벡터화)
-----------------------------

 : W2Vec,    Glove, Elmo,     (Wiki...)


-----------------------------

언어모델  **
-----------------------------

 - n-gram
 - 빈도기반 : BoW, TF-IDF
 - cos 유사도

-----------------------------

RNN  **
-----------------------------

  - RNN, LSTM(BiLSTM), GRU

[실습 : 감성(긍부정), 토픽(유사도), 요약(X), 챗봇(X)]

- KO네이버 리뷰(쇼핑,영화,댓글)
- EN로이터 뉴스(캐글..  NEWS... )
- CNN : 필터링


-----------------------------

심화과정
-----------------------------

 : Seq2Seq
 : Attention
 : Transfomer
 : BERT

------

# NLP

## 자르기

```python
word_tokenize(sent)

# 단어 단위로 자르기
from nltk.tokenize import word_tokenize

# 문장 단위로 자르기
from nltk.tokenize import sent_tokenize
----------------------------------------------------
# 품사 단위로 자르기
from nltk.tag import pos_tag

w토큰list =  word_tokenize(sent)
pos_tag(w토큰list)
```



## 한글관련

- morphs: 형태소 추출(의미단위의 최소단위)
- pos: 품사
- nouns: 명사
- KoNLpy



## 복합토큰화

```python
# zip
zip([1,2,3],[a,b,c]) ---> 같은 번째끼리 묶음
(1,a)(2,b)(3,c)
zip([문장토큰1, 문장토큰2, 문장토큰3, 원본문장])

from keras.preprocessing.text import Tokenizer
text_list = ['홍길동은 딥러닝을 공부합니다.',
        '홍길동은 딥러닝이 어렵습니다.', 
        '아무개도 딥러닝을 시작합니다.']

token = Tokenizer()
token.fit_on_texts(text_list)
print(' keras : \n')
print('문장 카운트 : ',token.document_count) 

print("==="*20)
print("문장 토큰화(texts_to_sequences) : ")
text_seq_list = token.texts_to_sequences(text_list)
print(text_seq_list)

print("==="*20)
for seq, sent in zip (text_seq_list,text_list) :
    print(sent, seq)

print("==="*20)      
print('단어 카운트 : \n', token.word_counts) 

print("==="*20)
print('각 단어에 매겨진 인덱스 값 :  \n',token.word_index)

print("==="*20)
print('각 단어가 몇 개의 문장에 포함되어 있는가 :  \n',token.word_docs) 
```

- from kefas.preprocessing.text import text_to_word_sequence

  - 문장s/토큰화(정수인코딩), 빈도수 체크

  

- from kefas.preprocessing.text import 

- token.fit_on_text(text_list)

  - token.document_count: 문장 카운트
  - token.texts_to_sequences(text_list)
  - token.word_counts
  - token.word_index: 자르고 인덱스 순, 문장을 자르면서 앞에서부터 차례대로 카운트
  - token.word_docs

  

- zip()





## 불용어 처리 + 정수인코딩, 사전만들기

```python
# 문장 토큰화
sentences = sent_tokenize(raw_text)
print(sentences)

['A barber is a person.', 'a barber is good person.', 'a barber is huge person.', 'he Knew A Secret!', 'The Secret He Kept is huge secret.', 'Huge secret.', 'His barber kept his word.', 'a barber kept his word.', 'His barber kept his secret.', 'But keeping and keeping such a huge secret to himself was driving the barber crazy.', 'the barber went up a huge mountain.']

vocab = {}
preprocessed_sentences = []
stop_words = set(stopwords.words('english'))

for sentence in sentences:
    # 단어 토큰화
    tokenized_sentence = word_tokenize(sentence)
    result = []

    for word in tokenized_sentence: 
        word = word.lower() # 모든 단어를 소문자화하여 단어의 개수를 줄인다.
        if word not in stop_words: # 단어 토큰화 된 결과에 대해서 불용어를 제거한다.
            if len(word) > 2: # 단어 길이가 2이하인 경우에 대하여 추가로 단어를 제거한다.
                result.append(word)
                if word not in vocab:
                    vocab[word] = 0 # 빈도수0
                vocab[word] += 1
    preprocessed_sentences.append(result) 
print(preprocessed_sentences)
# 결과
[['barber', 'person'], ['barber', 'good', 'person'], ['barber', 'huge', 'person'], ['knew', 'secret'], ['secret', 'kept', 'huge', 'secret'], ['huge', 'secret'], ['barber', 'kept', 'word'], ['barber', 'kept', 'word'], ['barber', 'kept', 'secret'], ['keeping', 'keeping', 'huge', 'secret', 'driving', 'barber', 'crazy'], ['barber', 'went', 'huge', 'mountain']]

print('단어 집합 :',vocab)
단어 집합 : {'barber': 8, 'person': 3, 'good': 1, 'huge': 5, 'knew': 1, 'secret': 6, 'kept': 4, 'word': 2, 'keeping': 2, 'driving': 1, 'crazy': 1, 'went': 1, 'mountain': 1}
```



cf. 시제표현을 잘 수행하지 못함 ---> RNN의 필요성

용언(동사, 형용사)에 대해서만 정규화 작업을 한다

정규표현식 ^"[A{1,2}]"$

정수인코딩: sklearn

차원의 저주: 데이터의 수보다 라벨의 수가 많은 경우 ---> 학습이 비효율적

해결? 워드임베딩



cf. 추천모델

- 유사도 (구글)
- 컨텐츠based(단점: 신규가입자면 뭐 추천?)
- 협업 필터링: 이 사람과 비슷한 사람들이 그 동안 무엇을 선호했는지 (넷플릭스)
- 군집









# 수정종가

배당락이 결정된 경우, 기업의 주가조정시 배당락을 반영하여 종가를 계산하는 지표

------

# pykrx 사용법



`get_index_name` 함수를 사용해서 티커의 이름을 얻을 수 있습니다.

```python
for ticker in stock.get_index_ticker_list():
    print(ticker, stock.get_index_ticker_name(ticker))
```

1001 코스피
1028 코스피 200
1034 코스피 100
1035 코스피 50
1167 코스피 200 중소형주
1182 코스피 200 초대형제외 지수
1244 코스피200제외 코스피지수
1150 코스피 200 커뮤니케이션서비스
1151 코스피 200 건설
1152 코스피 200 중공업
1153 코스피 200 철강/소재
1154 코스피 200 에너지/화학
1155 코스피 200 정보기술
1156 코스피 200 금융
1157 코스피 200 생활소비재
1158 코스피 200 경기소비재
1159 코스피 200 산업재
1160 코스피 200 헬스케어
1005 음식료품
1006 섬유의복
1007 종이목재
1008 화학
1009 의약품
1010 비금속광물
1011 철강금속
1012 기계
1013 전기전자
1014 의료정밀
1015 운수장비
1016 유통업
1017 전기가스업
1018 건설업
1019 운수창고업
1020 통신업
1021 금융업
1022 은행
1024 증권
1025 보험
1026 서비스업
1027 제조업
1002 코스피 대형주
1003 코스피 중형주
1004 코스피 소형주
1224 코스피 200 비중상한 30%
1227 코스피 200 비중상한 25%
1232 코스피 200 비중상한 20%

ex) 코스피 상위 10개? 1035에서 10개 슬라이싱

get_index_listing_date() 함수는 인덱스의 상장일 및 기준비수 정보를 조회합니다.

```python
df = stock.get_index_listing_date("KOSPI")
print(df.head())
```

## 공매도 정보

보통 분석시, 1달 전 자료로 확인

KRX는 (T+2)일 이후의 데이터를 제공합니다. 최근 영업일이 20190405라면 20190403일을 포함한 이전 데이터를 얻을 수 있습니다.

#### 2.1.3.1 종목별 공매도 현황

get_shorting_status_by_date() 메서드는 시작일/종료일/티커 세 개의 파라미터를 입력받아 공매도 현황을 DataFrame으로 반환합니다.

```python
df = stock.get_shorting_status_by_date("20181210", "20181212", "005930")
print(df)
```

당일 잔고와 (전일 잔고 + 당일 공매도 - 당일 상환) 수량은 정확하게 일치하지 않을 수 있습니다. 이는 투자자가 보유한 공매도잔고 비율이 상장주식수의 0.01% 미만인 경우 보고의 의무가 없기 때문에 집계되지 않을 수 있습니다.

              공매도     잔고   공매도금액      잔고금액
    20190401  154884  3403293  6981626250  153318349650
    20190402  186528  3435390  8529586850  157169092500
    20190403  211750  3380137  9837895500  157514384200

## 구조화상품: ETX API

`get_etf_ticker_list` 함수는 입력된 날짜에 존재하는 ETF 티커를 리스트로 반환합니다.

```python
tickers = stock.get_etf_ticker_list("20200717")
print(tickers[:10])
['346000', '342140', '342500', '342600', '342610', .... ]
```

20021014에는 `KODEX 200(069500)`과 `KOSEF 200(069660)` 두 개의 종목이 상장돼 있었습니다.

```python
get_etf_ticker_list("20021014")
print(tickers)
 ['069500', '069660']
```

## 채권 API

### 채권 수익률

#### 장외 채권수익률 - 전종목

KRX가 제공하는 11 종류의 장외 채권수익률을 출력합니다.

```python
df = bond.get_otc_treasury_yields("20190208")
print(df)


                       수익률   등락폭
장외 일자별 채권수익률
국고채 1년              1.743   -0.008
국고채 3년              1.786   -0.015
국고채 5년              1.853   -0.023
국고채 10년             1.965   -0.030
국고채 20년             2.039   -0.022
국고채 30년             2.034   -0.021
국민주택 1종 5년        1.935   -0.023
회사채 AA-(무보증 3년)  2.234   -0.015
회사채 BBB-(무보증 3년) 8.318   -0.014
CD(91일)                1.860    0.000
```

