# 💳 WiseCard

**WiseCard**는 사용자가 보유한 카드의 혜택 정보를 기반으로, 현재 위치에서의 **최적의 할인 매장**과 사용자의 소비 계획에 따른 **최적의 혜택 카드**를 안내해주는 **카드 혜택 추천 모바일 애플리케이션**입니다.

📍 **핵심 기능**
- 카드사 혜택 정보 크롤링 🔍
- Naver Map 연동으로 주변 매장 시각화 및 매장 추천 🗺️
- 사용자의 결제 상황에 맞춘 최적의 카드 자동 추천 💡

---

## 🚀 구현 기능

### 📌 카드 혜택 및 기간 한정 이벤트 정보 크롤링

카드사의 혜택 정보를 자동으로 수집하고 정리하는 크롤링 시스템을 구현하였습니다. 전체 과정은 다음과 같습니다:

1. **카드 목록 수집**

   * `extract_card_info_from_url(list_url, card_url_prefix)` 함수를 사용하여 카드 목록을 수집합니다.
   * 수집된 정보는 `_creditcardInfos.csv` 파일에 저장됩니다.
   * CSV 파일에는 다음 항목이 포함됩니다:

     * 카드 이름
     * 카드 이미지 URL
     * 카드 상세정보 URL

2. **카드 상세 혜택 정보 크롤링**

   * `_creditcardInfos.csv`에 저장된 각 카드의 상세 페이지 URL을 기반으로, 혜택 정보를 추가로 크롤링합니다.
   * 크롤링된 정보는 `credit_benefit.csv`에 저장되며, 아래와 같은 항목을 포함합니다:

     * 카드 ID
     * 카드 이름
     * 카드 이미지 URL
     * 혜택 정보
     * 크롤링 일자 (생성 일자)
     * 카드 종류 (신용/체크 여부)

### 🧩 카드 목록 추출 함수

#### ✅ 기능 요약

* 카드 이름 수집
* 카드 코드 추출 → 이미지 URL 및 상세 페이지 URL 생성
* 크롤링한 정보를 각각 `card_names`, `card_imgs`, `card_urls` 리스트에 저장

#### 🧪 주요 로직 예시

```python
# 카드 이름 추출
name = li.find('span', class_='h4_b_lt')
if name:
    card_names.append(name.text.strip())

# 카드 코드 추출 → 이미지 및 상세 페이지 URL 생성
a_tag = card_div.find('a')
if a_tag and 'onclick' in a_tag.attrs:
    onclick_text = a_tag['onclick']
    match = re.search(r"goCardDetail\('([^']+)'\)", onclick_text)
    if match:
        card_code = match.group(1)
        card_imgs.append(f'https://img.hyundaicard.com/img/com/card/card_{card_code}_h.png')
        card_urls.append(f'{card_url_prefix}{card_code}&eventCode=00000')
```

### 🧩 카드 상세 혜택 정보 크롤링

#### ✅ 기능 요약

* 각 카드 상세 URL에 접속 카드 혜택 관련 텍스트만 선별 추출 (카드사마다 상이함)
* 혜택 텍스트를 `benefits` 리스트에 저장

#### 🧪 주요 로직 예시

```python
# 웹 페이지 접속 및 파싱
driver.get(card_urls[i])
html = driver.page_source
soup = BeautifulSoup(html, 'html.parser')

# 팝업 컨테이너 중 혜택 관련된 모달만 선별
popup_containers = soup.find_all('div', class_='modal_pop')
exclude_ids = {'popup_call_reservation', 'cardApplyPop', ...}

for container in popup_containers:
    if container.get('id') in exclude_ids:
        continue
    for el in container.find_all(['h1', 'p', 'li']):
        texts.append(el.get_text(strip=True))
benefits.append(texts)
```

* `selenium`, `BeautifulSoup`, `pandas`를 활용하여 카드사 웹페이지를 자동으로 순회하며 상세 정보를 수집합니다.
* 팝업 ID 필터링을 통해 혜택과 무관한 요소는 제외합니다.

---

### 📌 Gemini를 활용한 카드 혜택 정보 정제 
- Google AI - Gemini 2.0 Flash 모델

웹 크롤링을 통해 수집한 카드 혜택 문장을 입력값으로 활용하여, Gemini가 해당 문장을 이해한 후 카테고리, 할인율, 포인트, 캐시백, 해당 매장 등으로 자동 분류하고 JSON 형태로 구조화합니다.

```python
for _, card in df.iterrows():
   benefit_text = card["benefits"]

   prompt = (
       f"{benefit_text}\n\n"
        '''
        JSON 예시를 포함한 프롬프트
        '''
   )

   try:
       response = model.generate_content(prompt)
       json_response = response.text.strip().replace("```json","").replace("```","")
       updated_benefits.append(json_response)
       print(json_response)
   except Exception as e:
       print(f"Error processing card {card['name']}: {e}")
       updated_benefits.append("")
```

---

### 📌 Kakao Map API - 장소 검색 API

### 🧩 혜택 적용 매장 탐색
사용자의 GPS 기반 위치를 기준으로, 크롤링을 통해 수집한 카드 혜택의 적용 매장이 주변에 존재하는지를 확인하는 용도로 사용하였습니다. 
  
검색 키워드와 위치 좌표를 함께 전송하면, 결과로 해당 장소의 이름, 주소, 거리, 위도/경도 등의 정보를 반환합니다.

```python
x_coord = "126.94"
y_coord = "37.55"

results = []

for cafe in cafe_targets_list:
   params = {
       "query": cafe,
       "x": x_coord,
       "y": y_coord,
       "radius": 1000,
       "size": 5
   }
   response = requests.get(url, headers=headers, params=params)
   if response.status_code != 200:
       print(f"{cafe} 검색 실패: {response.status_code}")
       continue

   places = response.json().get("documents", [])


   for place in places:
       place_info = {
           "place_name": place.get("place_name"),
           "road_address_name": place.get("road_address_name"),
           "x": place.get("x"),
           "y": place.get("y"),
           "카페명": cafe,
           "거리": int(place.get("distance", "0")),  # 문자열을 정수로 변환
           "카드별_혜택": {}
       }

       # 3. 이 place의 카페명으로 어떤 카드에서 혜택 있는지 확인 후 혜택 첨부
       for card_name, info in card_cafe_benefits.items():
           if cafe in info["대상"]:
               place_info["카드별_혜택"][card_name] = deepcopy(info["혜택"])  # 깊은 복사
       results.append(place_info)
```

### 🧩 카드 내역의 매장 카테고리 분류 및 활용
사용자 카드 내역 파일(카드내역.csv)을 불러와, 결제된 매장의 카테고리를 자동으로 추출하는 과정에서 Kakao Map API를 사용합니다.

추출한 카테고리 정보를 기반으로 해당 분야(예: 카페)에서의 결제 건별 평균 소비 금액을 도출할 수 있으며, 이는 이후 예상 할인 금액을 계산하고 카드 추천 시스템에 활용하는 과정에 적용할 수 있습니다.

#### 1. 카드 내역 불러오기
```python
card_history_df = pd.read_csv('카드내역2.csv')
```

#### 2. 카카오 API 설정
```python
headers = {"Authorization": "KakaoAK API-KEY"}
url = "https://dapi.kakao.com/v2/local/search/keyword.json"
```

#### 3. 카테고리 코드 매핑
```python
category_map = {
   "MT1": "대형마트", "CS2": "편의점", "PS3": "어린이집, 유치원", "SC4": "학교",
   "AC5": "학원", "PK6": "주차장", "OL7": "주유소, 충전소", "SW8": "지하철역",
   "BK9": "은행", "CT1": "문화시설", "AG2": "중개업소", "PO3": "공공기관",
   "AT4": "관광명소", "AD5": "숙박", "FD6": "음식점", "CE7": "카페",
   "HP8": "병원", "PM9": "약국"
}
```

#### 4. 카테고리별 누적 정보 저장용 dict
```python
category_stats = {}
```

#### 5. 검색 및 수집
```python
for idx, row in card_history_df.iterrows():
   store_name = row['가맹점명']
   try:
       amount = int(str(row['승인금액']).replace(",", "").replace("원", "").strip())
   except:
       amount = 0

   store_name = store_name.replace("( 주 )", "").replace("주식회사", "")
   params = {"query": store_name}

   try:
       response = requests.get(url, headers=headers, params=params)
       data = response.json()

       print(f"\n[{idx}] 가맹점명: {store_name} | 승인금액: {amount}원")
       if data.get("documents"):
           doc = data["documents"][0]
           category_code = doc.get("category_group_code", "")
           category_name = category_map.get(category_code, "기타")

           print(f"  • 이름: {doc['place_name']}")
           print(f"    카테고리: {category_name} ({category_code})")
           print(f"    ----------------------")

           if category_name not in category_stats:
               category_stats[category_name] = {"total_amount": 0, "count": 0}
           category_stats[category_name]["total_amount"] += amount
           category_stats[category_name]["count"] += 1
       else:
           print("  → 검색 결과 없음")

       time.sleep(0.3)

   except Exception as e:
       print(f"  오류 발생: {e}")
```

#### 6. 카테고리별 총합과 평균 출력
```python
print("\n카테고리별 총합 및 평균 금액:")
for category, stats in category_stats.items():
   total = stats["total_amount"]
   count = stats["count"]
   avg = total / count if count else 0
   print(f"  - {category}: 총 {total:,}원 / {count}회 / 평균 {avg:,.0f}원")
```
---

### 📌 혜택 적용 시스템
사용자가 받을 수 있는 카드 혜택과 앞으로 받을 수 있는 혜택 한도를 계산합니다. 

사용자의 카드 결제 내역 하나를 기준으로, 해당 결제에 대해 혜택을 받을 수 있는 카드들을 식별한 뒤, 할인 > 적립 > 캐시백 순의 우선 순위에 따라 가장 먼저 적용된 하나만 사용합니다. 

혜택이 적용될 수 있는 최소 조건이 있다면 해당 조건을 만족하는지 우선 판단하고, 조건을 만족했을 때 혜택 계산을 진행합니다.

```python
import re

# 혜택이 적용되는 최소 조건(결제 금액 등)을 만족하는 지 판단하는 함수
def extract_minimum_amount(condition_str):
    """
    '5만원 이상', '1만 원 이상', '5천원 이상' 등의 조건에서 최소 금액을 원 단위로 추출
    """
    if not condition_str:
        return None
    
    pattern = r'(\d+)(만|천)?\s*원?\s*이상' # 숫자 + (만|천) + 원 + 이상 패턴
    match = re.search(pattern, condition_str)
    
    if match:
        num = int(match.group(1))
        unit = match.group(2)
        
        if unit == '만':
            return num * 10000
        elif unit == '천':
            return num * 1000
        else:
            return num # 원 단위
    
    return None
```

```python
# 한 건의 결제내역에 대해 어떤 혜택이 적용되는지를 계산하는 함수
def apply_benefit(row_data):
    store_name = str(row_data["가맹점명"]) # 가맹점명을 문자열로 가져옴
    amount = row_data["이용금액"]
    applied_amount = amount
    benefit_detail = None # 어떤 혜택이 적용됐는지 (예: 할인, 적립, 캐시백)
    point_or_cashback = 0.0 # 적립/캐시백 금액 기본값 0원
    
    for idx, benefit_row in df.iterrows():
        benefits = benefit_row['benefits_dict']
        for category, info in benefits.items():
            if info is not None and "대상" in info:
                targets = info["대상"] # 가맹점명이 혜택 대상에 포함되는지 확인
                if targets and any(target in store_name for target in targets):
                    # 현재 가맹점명이 혜택 대상에 포함되는 경우
                    
                    # ───── 할인 처리 시작 ─────
                    할인_목록 = info.get("혜택", {}).get("할인", [])
                    for d in 할인_목록:
                        조건문 = d.get("조건", "")
                        최소_금액 = extract_minimum_amount(조건문) # 혜택이 적용되는 조건에서 최소금액 추출
                        if 최소_금액 and amount < 최소_금액: # 금액이 조건보다 작으면 건너뜀
                            continue
                        if d.get("할인율(%)"): # 할인율이 있는 경우
                            rate = d["할인율(%)"]
                            if rate:
                                applied_amount = amount * (1 - rate / 100)
                                benefit_detail = "할인"
                                return applied_amount, benefit_detail, 0.0
                        elif d.get("금액(원)"): # 정액 할인 금액이 있는 경우
                            discount_amt = d["금액(원)"]
                            if discount_amt:
                                applied_amount = max(0, amount - discount_amt)
                                benefit_detail = "할인"
                                return applied_amount, benefit_detail, 0.0

                    # ───── 적립 처리 시작 ─────
                    적립_목록 = info.get("혜택", {}).get("적립", [])
                    for d in 적립_목록:
                        if d.get("적립율(%)"): # 적립율이 있는 경우
                            rate = d["적립율(%)"]
                            if rate:
                                benefit_detail = "적립"
                                point_or_cashback = amount * rate / 100
                                return amount, benefit_detail, point_or_cashback # 지불 금액은 변함없음
    
                    # ───── 캐시백 처리 시작 ─────
                    캐시백_목록 = info.get("혜택", {}).get("캐시백", [])
                    for d in 캐시백_목록:
                    if d.get("캐시백 비율(%)"): # 비율 캐시백
                        cashback_rate = d["캐시백 비율(%)"]
                        if cashback_rate:
                            benefit_detail = "캐시백"
                            point_or_cashback = amount * cashback_rate / 100
                            return amount, benefit_detail, point_or_cashback
                    elif d.get("금액(원)"): # 정액 캐시백
                        cashback_amt = d["금액(원)"]
                        if cashback_amt:
                            benefit_detail = "캐시백"
                            point_or_cashback = cashback_amt
                            return amount, benefit_detail, point_or_cashback
    
    # 어떤 혜택도 적용되지 않은 경우
    return amount, None, 0.0
```