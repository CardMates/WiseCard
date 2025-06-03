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
