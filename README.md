# 우리 동네 알아보기

# 1. 프로젝트의 목적

우리가 생활하며 흔히 접하게 되는 가게들이 어디에 많이 분포해 있고, 이 점포들은 왜 해당 지역에 많이 있는지 알아보기 위해 다음과 같은 목적성을 두고 프로젝트를 진행하게 되었다. 

- 각 자치구 내 어떤 업종이 많이 발달되어 있는 지?
- 전년도 동분기와 비교했을 때 상권 변화가 있는 지?
- 어떤 변수가 상권 분포에 영향을 미치는 지?

위의 질문들에 답하기 위해서 상권 정보 데이터를 주 데이터로 활용, 어떤 변수가 상권분포에 영향을 주었는지 확인하기 위해 사업체 종사자 수, 거주인구 데이터 등을 활용하였다.

또한, 태블로 대시보드를 제작해 원하는 지역에 분포하고 있는 상권들을 쉽게 알아볼 수 있도록 하였다.

# 2. 데이터 전처리

- **[소상공인시장진흥공단_상가(상권)정보](https://www.data.go.kr/data/15083033/fileData.do) 전처리**
    
    ```sql
    -- 상호명과 지점명을 함께보기 위해 두 컬럼의 문자열을 하나로 합쳐줌
    SELECT 상호명
          ,지점명
          ,IF(상호명 REGEXP 지점명 OR 지점명 IS NULL, 상호명, CONCAT(상호명, 지점명)) 상호지점명
      FROM market_info_202209
    ```
    
- **[서울시 사업체현황 종사자수(종사상 지위별/동별/성별)](https://www.data.go.kr/data/15046680/fileData.do) 통계 전처리**
    
    ![Untitled](https://wonil.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fff5283e6-fa2a-44c7-9397-826f3d5457c3%2FUntitled.png?id=efa5c6c7-40b2-4aa8-85e5-66acd43c3f41&table=block&spaceId=6b2368ed-25f6-40aa-89b1-aad3f3ad9929&width=1310&userId=&cache=v2)
    
    필요없는 합계컬럼과 구별로 소계의 행을 정리함. 시군구명과 행정동명으로 조인을 하기위해 컬럼 피쳐명을 변경.
    
    ![Untitled](https://wonil.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F52f0aea0-84f3-4d46-9073-c9b7bd6169f2%2FUntitled.png?id=0291096c-2c23-42c6-99f2-9fb4c2bdb96f&table=block&spaceId=6b2368ed-25f6-40aa-89b1-aad3f3ad9929&width=1160&userId=&cache=v2)
    
- **[서울시 사업체 현황 - 종사자규모별/동별](https://www.data.go.kr/data/15046671/fileData.do?recommendDataYn=Y) 전처리**
    
    ![Untitled](https://wonil.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F4ea37c3a-4cde-4300-8add-27dd1289db1e%2FUntitled.png?id=6e9683ed-7b15-46c7-8650-63ec0033b408&table=block&spaceId=6b2368ed-25f6-40aa-89b1-aad3f3ad9929&width=1340&userId=&cache=v2)
    
    농업 임업 및 어업 / 광업 등의 상세한 사업체 정보를 포함한 내용을 삭제하고, 사업체 수와 종사자 수 합계만 남김.
    
    ![Untitled](https://wonil.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F02e28a5a-73f3-43aa-83e4-51adcaf282fd%2FUntitled.png?id=d760862d-942a-433a-a64c-04238d507bd5&table=block&spaceId=6b2368ed-25f6-40aa-89b1-aad3f3ad9929&width=670&userId=&cache=v2)
    

# 3. EDA

## 1) 구, 동 별 상권 파악

### 1-1. 자치구별 상권업종 TOP 1

![Untitled](https://wonil.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Ff9c4513d-3a02-4cf5-ada6-5420add146ae%2FUntitled.png?id=e0335662-c112-44b4-b684-97c9654d8700&table=block&spaceId=6b2368ed-25f6-40aa-89b1-aad3f3ad9929&width=1340&userId=&cache=v2)

- SQL query
    
    ```sql
    -- 구별로 가장 많은 업종 확인하기
    WITH gu_rank AS (
    	SELECT 시군구명
    		    ,상권업종대분류명
            ,상권업종중분류명
            ,COUNT(*) AS cnt
    	  FROM market_info_202209
    	 GROUP BY 1,2,3
       ORDER BY cnt DESC
    )
    
    SELECT *
      FROM (
    	SELECT 시군구명
            ,상권업종대분류명
            ,cnt
    		    ,RANK() OVER(PARTITION BY 시군구명 ORDER BY cnt DESC) AS ranking
    	  FROM gu_rank
      ) AS a
      WHERE ranking = 1
      ORDER BY cnt DESC;
    ```
    

> **분석 결과** :  대부분의 자치구에서는 음식점이 가장 많이 분포되어 있다. 종로구 같은 경우 상점들이 몰려있는 상가단지(낙원상가, 세운상가, 귀금속상가 등)가 많이 분포되어 있어 소매점이 가장 많은 것으로 볼 수 있다.
> 

### 1-2. 22년 9월 기준 전년도 동분기 업종별 점포수 비교

![Untitled](https://wonil.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fa2517488-9fa2-4b42-975d-780891298af2%2FUntitled.png?id=34e35568-6262-4d6c-b3f6-b2108f801ddb&table=block&spaceId=6b2368ed-25f6-40aa-89b1-aad3f3ad9929&width=1250&userId=&cache=v2)

- SQL query
    
    ```sql
    WITH m_22 AS
    (SELECT 상권업종대분류명 
           ,COUNT(*) cnt_22
       FROM market_info_202209
      GROUP BY 상권업종대분류명)
     ,m_21 AS(
     SELECT 상권업종대분류명 
    		   ,COUNT(*) cnt_21
       FROM market_info_202109
      GROUP BY 상권업종대분류명)
      
    SELECT m_22.상권업종대분류명
          ,cnt_22
          ,cnt_21
          ,cnt_22 - cnt_21
     FROM m_22
          INNER JOIN m_21 ON m_22.상권업종대분류명 = m_21.상권업종대분류명
    ```
    

> **분석결과** : 모든 업종대분류에서 2021년도보다 2022년도의 점포수가 많다. 여기서 가장 큰 폭을 나타낸 업종은 생활서비스이다. 생활서비스 세부 업종을 파악해보니, 사업경영상담(컨설팅)이 많이 생긴 업종이었다.
> 

### 1-3. 시군구, 연령별 거주 인구수 TOP 1

![Untitled](https://wonil.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F76b43c42-1ed1-4cf1-921c-c47552459693%2FUntitled.png?id=3f1afead-c1d6-475d-b5cd-48134952e632&table=block&spaceId=6b2368ed-25f6-40aa-89b1-aad3f3ad9929&width=1340&userId=&cache=v2)

- SQL query
    
    ```sql
    WITH cte AS (
    SELECT 시도명
    	  , 시군구명
    	  , 연령대
    	  , SUM(연령대_cnt) AS sum_age
      FROM residents
    GROUP BY 시군구명, 연령대)
    
    SELECT *
      FROM cte
    WHERE sum_age IN (
    	SELECT max(sum_age)
        FROM cte
        GROUP BY 시군구명
    )
    ```
    

> **분석결과 :** 대부분의 자치구에서 40 ~ 50대의 비율이 가장 많이 나타났으며, 대학가 중심(관악구, 동작구 광진구, 서대문구 등)으로 20대가 많이 분포하고 있다.
> 

## 2) 구, 동 별 헬스장 수 TOP5 확인하기

| 시군구명 | 행정동명 | cnt |
| --- | --- | --- |
| 강남구 | 역삼1동 | 70 |
| 영등포구 | 여의동 | 37 |
| 강남구 | 청담동 | 37 |
| 강서구 | 가양1동 | 35 |
| 강남구 | 논현2동 | 33 |
- SQL query
    
    ```sql
    SELECT 시군구명, 행정동명, COUNT(상호명) cnt
      FROM market_info_202209
     WHERE 상권업종소분류명 = '헬스클럽'
     GROUP BY 1,2
     ORDER BY cnt DESC
     LIMIT 5
    ```
    

> 가장 많은 헬스클럽의 점포를 가지고 있는 동은 역삼 1동과 여의동이다. 
구별로 보자면 강남구가 헬스클럽(업종소분류) 점포의 수 top5에 상당수 차지하고 있음을 알 수 있다.
> 

```
❓ 강남구의 역삼 1동과 영등포구 여의도동에 헬스클럽이 왜 많을까?
```

```
❓ 가설 : 대표적으로 강남구 와 영등포구는 회사가 밀집되어있는 지역을 고려하여 직장인이 
         많은 지역이니 건강을 많이 신경쓰는 직장인 상대로 한 헬스클럽이 밀집되어 있을 것이다.
```


**검증**: **[서울시 사업체현황 종사자수(종사상 지위별/동별/성별) 통계](https://data.seoul.go.kr/dataList/10598/S/2/datasetView.do)** 에서 각 구, 동 별 사업체수의 top5를 뽑아 보았다.

| 시군구명 | 행정동명 | 사업체수 | 총종사자수 |
| --- | --- | --- | --- |
| 금천구 | 가산동 | 24,572 | 186,332 |
| 강남구 | 역삼1동 | 24,219 | 175,876 |
| 종로구 | 종로1.2.3.4가동 | 18,578 | 115,769 |
| 영등포구 | 여의동 | 15,305 | 175,325 |
| 영등포구 | 영등포동 | 14,749 | 44,697 |
- SQL query
    
    ```sql
    SELECT m.시군구명, m.행정동명, w.사업체수, w.총종사자수
      FROM market_info_202209 m 
    	     INNER JOIN seoul_work_info w ON m.행정동명 = w.행정동명 
     GROUP BY 1,2
     ORDER BY w.사업체수 DESC
     LIMIT 5;
    ```
    

> 구,동 별 사업체 수의 top 5에 역삼1동과 여의동이 있는 것으로 보아 가설의 타당성이 있는 것으로 확인된다.
> 

## 제안
>❓ 사업체데이터와 헬스장데이터를 보았을때 거주인구데이터에서 연령별로 확인하여 **어떤 연령대에 맞춤**으로 헬스클럽에 홍보를 중점으로 두어야 하는지 알고 싶다.
>

**[행정안전부_지역별(법정동) 성별 연령별 주민등록 인구수](https://www.data.go.kr/data/15099158/fileData.do)** 데이터를 활용하여 **사업체 데이터와 헬스장 데이터의 상위 1위 행정동명**과 겹치는 **여의도동**에 대한 연령대별 인구수  top3를 뽑아 보았다.

| 시군구명 | 행정동명 | 연령대 | cnt | ranking |
| --- | --- | --- | --- | --- |
| 강남구 | 역삼동 | 40대 | 14721 | 1 |
| 강남구 | 역삼동 | 30대 | 14429 | 2 |
| 강남구 | 역삼동 | 20대 | 11707 | 3 |
| 금천구 | 가산동 | 20대 | 7043 | 1 |
| 금천구 | 가산동 | 30대 | 6109 | 2 |
| 금천구 | 가산동 | 50대 | 2811 | 3 |
| 영등포구 | 여의도동 | 40대 | 5267 | 1 |
| 영등포구 | 여의도동 | 50대 | 5199 | 2 |
| 영등포구 | 여의도동 | 30대 | 4638 | 3 |
- SQL query
    
    ```sql
    SELECT m.시군구명
    	  ,m.법정동명
          ,r.연령대
          ,r.연령대_cnt
          ,r.ranking
      FROM market_info_202209 m
    	INNER JOIN (
    		SELECT *
    			  , DENSE_RANK() OVER(partition by 읍면동명 ORDER BY 연령대_cnt DESC) AS ranking
            FROM residents) r ON m.법정동코드 = r.법정동코드
    WHERE m.시군구명 REGEXP '금천구|강남구|영등포구' AND m.법정동명 REGEXP '가산동|여의도동|역삼동' AND ranking between 1 AND 3
    GROUP BY 1,2,3
    ORDER BY m.시군구명, m.법정동명, r.연령대_cnt DESC
    ```
    

전체적으로 연령대의 주류가 **30대에서 50대로 인것으로 확인**이 되었다.

추가적으로 2022-01월 부터 2023-01월까지의 헬스장과 관련된 **검색키워드(헬스,다이어트,헬스장)** 를 검색한 연령대를 보아도 **30~50대가 주를 이루었던 것으로 확인**이 된다.

![연령별 검색량 그래프.png](https://wonil.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F8147b5d9-0868-4164-b07e-6b30aa4e3624%2F%25EC%2597%25B0%25EB%25A0%25B9%25EB%25B3%2584_%25EA%25B2%2580%25EC%2583%2589%25EB%259F%2589_%25EA%25B7%25B8%25EB%259E%2598%25ED%2594%2584.png?id=78119511-bc0d-43ca-b952-ccc0127c5342&table=block&spaceId=6b2368ed-25f6-40aa-89b1-aad3f3ad9929&width=1340&userId=&cache=v2)

[카카오데이터트랜드](https://datatrend.kakao.com/result?q=%ED%97%AC%EC%8A%A4&q=%EB%8B%A4%EC%9D%B4%EC%96%B4%ED%8A%B8&q=%ED%97%AC%EC%8A%A4%EC%9E%A5&from=20220111&to=20230111&area=seoul)

**30~50대의 연령대를 타켓으로 적극적으로 홍보를 하는것이 효과적인 홍보방법으로 보인다.**

## 3) 구, 동 별 5대 PC방 수 TOP5 확인하기

### **1) 법정동별 PC방 수**

| 시군구명 | 법정동명 | cnt |
| --- | --- | --- |
| 관악구 | 신림동 | 50 |
| 강서구 | 화곡동 | 34 |
| 강남구 | 역삼동 | 25 |
| 관악구 | 봉천동 | 24 |
| 중랑구 | 면목동 | 20 |
- SQL query
    
    ```sql
    WITH cte AS
    (SELECT m.시군구명
           ,법정동명
           ,COUNT(*) cnt
       FROM market_info_202209 m
            LEFT JOIN pop_by_age p ON m.법정동명 = p.읍면동명
      WHERE 상권업종소분류명 REGEXP 'pc방'
      GROUP BY 1, 2)
     
     SELECT 시군구명
           ,법정동명
           ,cnt
    	     ,DENSE_RANK() OVER(ORDER BY cnt DESC) rk
       FROM cte
      LIMIT 10
    ```
    

> 신림동, 화곡동 순으로 PC방이 가장 많이 분포하고 있다.
시군구명으로 확인했을 때는, 각기 다른 구에서 분포하고 있음을 보인다.
> 

**신림동, 화곡동 등에는 왜 PC방이 많이 분포되어 있는 것인가?**

**가설 :**  위 5개의 행정동 주변에는 대학교가 있거나(신림동, 봉천동, 면목동), 주로 20대, 30대가 머무르는 원룸, 오피스텔이 많이 있어 **PC방이 많이 밀집**되어있을 것이다. 

**검증 : [행정안전부_지역별(법정동) 성별 연령별 주민등록 인구수](https://www.data.go.kr/data/15099158/fileData.do)** 데이터를 이용하여 연령대 인구분포를 확인해 보았다.

### 2) 법정동별 연령대 비율(%)

| 시군구명 | 법정동명 | 10대 미만 | 10대 | 20대 | 30대  | 40대 | 50대 | 60대 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 관악구 | 신림동 | 3.33 | 5.03 | 22.54 | 17.58 | 13.25 | 14.11 | 12.79 |
| 강서구 | 화곡동 | 5.12 | 6.73 | 16.08 | 17.15 | 15.51 | 14.76 | 14.06 |
| 강남구 | 역삼동 | 4.66 | 10.09 | 16.33 | 20.13 | 20.54 | 13.55 | 7.80 |
| 관악구 | 봉천동 | 4.28 | 6.04 | 23.10 | 17.69 | 13.25 | 13.01 | 11.66 |
| 중랑구 | 면목동 | 4.94 | 5.58 | 13.64 | 14.98 | 14.26 | 17.90 | 16.02 |
- SQL query
    
    ```sql
    SELECT 시군구명
          ,읍면동명
          ,ROUND((10대_미만/all_sum) * 100, 2) 10대_미만
          ,ROUND((10대/all_sum) * 100, 2) 10대
          ,ROUND((20대/all_sum) * 100, 2) 20대
          ,ROUND((30대/all_sum) * 100, 2) 30대
          ,ROUND((40대/all_sum) * 100, 2) 40대
          ,ROUND((50대/all_sum) * 100, 2) 50대
          ,ROUND((60대/all_sum) * 100, 2) 60대
      FROM pop_by_age
     WHERE 읍면동명 REGEXP '신림동|화곡동|역삼동|봉천동|면목동'
    ```
    

> 법정동별 연령대 비율로 확인한 결과, 20 ~ 30대의 비율이 상당히 높은 것으로 확인할 수 있으며, 
가설의 타당성이 있는 것으로 판단된다.
> 

## 4) 구, 동 별 카페 수 TOP5 확인하기

### 1)  상황별 카페 수

- **구별 커피점/카페 TOP 5 확인하기**

| 시군구명 | 점포 수 | 순위 |
| --- | --- | --- |
| 강남구 | 2191 | 1 |
| 마포구 | 1637 | 2 |
| 송파구 | 1243 | 3 |
| 서초구 | 1205 | 4 |
| 강서구 | 1134 | 5 |
- SQL query
    
    ```sql
    WITH coffee_gu AS
    ( 
    	SELECT 상권업종중분류명
    			, 시군구명
    			, COUNT(*) AS cnt
    			FROM market_info_202209
    			WHERE 상권업종중분류명 = '커피점/카페'
    			GROUP BY 상권업종중분류명, 시군구명 
    			ORDER BY 상권업종중분류명, cnt DESC
    )
    SELECT * 
    FROM (
    	SELECT 상권업종중분류명
    			, 시군구명
    			, cnt
    			, DENSE_RANK() OVER(PARTITION BY 상권업종중분류명 ORDER BY cnt DESC) AS Ranking
    	FROM coffee_gu
    ) AS ft 
    ORDER BY cnt DESC;
    ```
    

- **법정동별 커피점/카페 TOP 5**

| 시군구명 | 법정동명 | 점포 수 | 순위 |
| --- | --- | --- | --- |
| 강남구 | 역삼동 | 575 | 1 |
| 서초구 | 서초동 | 424 | 2 |
| 마포구 | 서교동 | 368 | 3 |
| 강남구 | 신사동 | 363 | 4 |
| 관악구 | 봉천동 | 357 | 5 |
- SQL query
    
    ```sql
    WITH coffee_dong AS
    ( 
    	SELECT 상권업종중분류명
    			, 시군구명
    			, 법정동명
    			, COUNT(*) AS cnt
    			FROM market_info_202209
    			WHERE 상권업종중분류명 = '커피점/카페'
    			GROUP BY 상권업종중분류명, 법정동명 
    			ORDER BY 상권업종중분류명, cnt DESC
    )
    SELECT * 
    FROM (
    	SELECT 상권업종중분류명
    			, 시군구명
    			, 법정동명
    			, cnt
    			, DENSE_RANK() OVER(PARTITION BY 상권업종중분류명 ORDER BY cnt DESC) AS Ranking
    	FROM coffee_dong
    ) AS ft 
    ORDER BY cnt DESC;
    ```
    

> 구 단위로 확인했을 때는 **강남구, 마포구**가 다른 곳에 비해 **다소 많은** 카페를 가지고 있는 것으로 확인되었다. 법정동 단위로 세분화한 결과, **강남구 내 2개의 법정동**이 순위권 내에 포함되어 있는 것을 추가적으로 확인할 수 있다.
> 

### 2) 가설 수립 및 검증


💡 **강남구, 마포구, 서초구에는 왜 카페가 많은 것일까?**

**가설 1**  :  흔히 언급하는 **커피 프랜차이즈점**이 많아 집계된 점포수가 많을 것이다.
**가설 2**  :  회사와 같은 **사업체 수, 주변 음식점**의 영향일 것이다. 

**검증** : [서울시 사업체 현황 - 종사자규모별/동별](https://data.seoul.go.kr/dataList/104/S/2/datasetView.do) , [소상공인시장진흥공단_상가(상권)정보](https://www.data.go.kr/data/15083033/fileData.do) 데이터를 활용하여 위의 가설들을 검증해보고자 한다.



### 검증 1)  6대 커피 프랜차이즈 분포 현황

- `공정거래위원회의 2021년 12월 기준 프랜차이즈 카페 브랜드별 매장 수 통계` / `2022년 브랜드 평판 지수` / `와이즈앱의 2022년 7월 기준 커피 전문점 단독 결제율 현황 정보`를 조합하여 **6개의 브랜드(메가커피/빽다방/스타벅스/이디야커피/컴포즈커피/투썸플레이스)**를 최종 선정

| 시군구명 | 점포 수 |
| --- | --- |
| 강남구 | 173 |
| 강서구 | 128 |
| 송파구 | 121 |
| 중구 | 107 |
| 마포구 | 98 |
| 영등포구 | 97 |
| 서초구 | 80 |
| 구로구 | 80 |
| 강동구 | 79 |
| 성북구 | 77 |
- SQL query
    
    ```sql
    SELECT 시군구명, 
            COUNT(*) AS cnt
    FROM market_info_202209
    WHERE 상권업종중분류명 = '커피점/카페' 
    	AND 상호명 REGEXP '스타벅스|이디야|투썸|메가|빽다방|컴포즈' 
    GROUP BY 1;
    ```
    

> 선정한 6대 브랜드 점포 수를 구 별로 조회한 결과, 기존에 집계했던 카페 점포 수 TOP5를 모두 포함하고 있는 것으로 확인되어 **커피 프랜차이즈 점 또한 많을 것**이라는 가설이 어느정도 타당하다는 사실을 알 수 있다.
> 

- **TOP 5 구 내에 위치한 프랜차이즈 커피점 분포**

![Untitled](https://wonil.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fcce9dfaa-2034-4662-87fa-bfd7a592a52f%2FUntitled.png?id=6c618772-2fe1-4650-9093-021efd1be8a2&table=block&spaceId=6b2368ed-25f6-40aa-89b1-aad3f3ad9929&width=1340&userId=&cache=v2)

- SQL query
    
    ```sql
    -- 프랜차이즈 커피점/카페 조회
    
    SELECT 시군구명, 
    		상호명,
            COUNT(*) AS cnt,
    		DENSE_RANK() OVER(PARTITION BY 시군구명 ORDER BY COUNT(*) DESC) AS ranking
    FROM market_info_202209
    WHERE 상권업종중분류명 = '커피점/카페' 
    	AND 상호명 REGEXP '스타벅스|이디야|투썸|메가|빽다방|컴포즈' 
    	AND 시군구명 IN ('강남구','마포구','송파구','서초구','강서구')
    GROUP BY 1, 2;
    
    -- 강남구 
    
    SELECT 시군구명, 
    	   법정동명,
           상호명,
           COUNT(*) AS cnt,
           DENSE_RANK() OVER(PARTITION BY 법정동명 ORDER BY COUNT(*) DESC) AS ranking
    FROM market_info_202209
    WHERE 상권업종중분류명 = '커피점/카페' 
    	AND 상호명 REGEXP '스타벅스|이디야|투썸|메가|빽다방|컴포즈' 
    	AND 시군구명 = '강남구'
    GROUP BY 1,2,3
    HAVING cnt >= 1;
    
    -- 이하 동일 
    ```
    

> 보다 세분화하여 어떤 프랜차이즈가 **인기가 많은 지** 살펴보았다. **강남구**와 **서초구**는 **스타벅스**의 **점포 수가** 압도적으로 **많았으며** 그에 반해 **강서구**에서는 이디야커피 - 메가커피 - 컴포즈커피 순서로 **저가형 브랜드** 점포가 많은 것을 알 수 있다.
> 

### 검증 2) 서울시 행정구역별 사업체 / 음식점 현황

- **구 별 사업체 및 종사자 수**

![Untitled](https://wonil.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fb3ff705b-24ca-411f-ae17-8df7f4a58708%2FUntitled.png?id=f4203788-3ba6-48e2-a8e7-3f2132758627&table=block&spaceId=6b2368ed-25f6-40aa-89b1-aad3f3ad9929&width=1340&userId=&cache=v2)

- SQL query
    
    ```sql
    SELECT 시군구명, 
    		SUM(사업체수) AS 사업체수,
    		SUM(종사자수) AS 종사자수
    FROM company
    GROUP BY 1
    ORDER BY SUM(사업체수) DESC;
    ```
    

> **강남구**가 사업체의 수가 **압도적으로 높아** 자연스럽게 종사자 수 또한 많은 것으로 확인되었다. 그 밖에 순위를 살펴보자면 `송파구 - 중구 - 서초구 - 영등포구` 순서로 사업체 수가 많으며, 카페가 많은 곳으로 집계 되었던 마포구와 강서구는 다소 적은 사업체 수를 보인다.
> 



# 4. 데이터 설명

소상공인시장진흥공단의 상가 정보 데이터셋을 주 데이터로 활용, 어떤 변수가 상권분포에 영향을 주었는지 대해서 사업체 종사자 수, 생활인구, 주민등록 인구수 데이터셋 등을 이용해 EDA 및 대시보드 제작에 사용하였다.

### **데이터 :**

- **[소상공인시장진흥공단_상가(상권)정보](https://www.data.go.kr/data/15083033/fileData.do)**
- [**서울시 사업체현황 종사자수(종사상 지위별/동별/성별) 통계**](https://data.seoul.go.kr/dataList/10598/S/2/datasetView.do)
- [**행정안전부_지역별(법정동) 성별 연령별 주민등록 인구수**](https://www.data.go.kr/data/15099158/fileData.do)

