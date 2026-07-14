# Amplitude 확인 기준

## 1. 현재 수집 중인 이벤트

### 자동 수집 이벤트

| 이벤트          | 의미                      |
| --------------- | ------------------------- |
| `Start Session` | 새로운 방문 세션이 시작됨 |
| `End Session`   | 방문 세션이 종료됨        |
| `Page Viewed`   | 페이지가 조회됨           |

### 직접 만든 이벤트

| 이벤트             | 의미                          |
| ------------------ | ----------------------------- |
| `portfolio_viewed` | 포트폴리오 방문 흐름이 시작됨 |
| `section_viewed`   | 특정 주요 섹션에 도달함       |

---

# 2. 정상 작동 확인 순서

## 포트폴리오 첫 진입

페이지를 새로고침했을 때 다음 이벤트가 들어와야 한다.

```
Start Session
Page Viewed
portfolio_viewed
section_viewed
```

첫 번째 `section_viewed`의 속성은 다음이어야 한다.

```
section_name: intro
section_order: 1
```

## Capability 구간 진입

Capability 구간까지 스크롤하면 새로운 `section_viewed`가 발생해야 한다.

```
section_name: capability
section_order: 2
```

## Contact 구간 진입

페이지 끝까지 내려가면 다시 `section_viewed`가 발생해야 한다.

```
section_name: contact
section_order: 3
```

---

# 3. Live Events에서 확인할 흐름

정상적인 이벤트 흐름은 대략 다음과 같다.

```
Start Session
Page Viewed
portfolio_viewed
section_viewed { intro }
section_viewed { capability }
section_viewed { contact }
```

표시 순서가 약간 다를 수는 있지만, 세 섹션 이벤트가 모두 존재하면 된다.

---

# 4. `section_viewed` 속성 확인 기준

각 `section_viewed` 이벤트를 선택한 뒤 아래 속성을 확인한다.

| 속성               | 의미                     | 정상 값 예시                     |
| ------------------ | ------------------------ | -------------------------------- |
| `section_name`     | 도달한 섹션 이름         | `intro`, `capability`, `contact` |
| `section_order`    | 페이지 내 섹션 순서      | `1`, `2`, `3`                    |
| `device_type`      | 화면 기준 기기 분류      | `mobile`, `tablet`, `desktop`    |
| `viewport_width`   | 브라우저 화면 너비       | `390`, `768`, `1512`             |
| `pathname`         | 이벤트가 발생한 경로     | `/`                              |
| `visibility_ratio` | 감지 당시 섹션 노출 비율 | `0.01` 등                        |

## `visibility_ratio`가 작아도 되는 이유

현재 스크롤 스테이지는 일반 섹션보다 매우 길다.

또한 이벤트가 다음 기준으로 발생한다.

```
섹션이 화면 중앙 감지 영역에 진입
→ section_viewed 발생
```

따라서 전체 섹션 대비 노출 비율은 `0.01`처럼 작을 수 있다. 오류가 아니다.

현재 분석에서 더 중요한 값은 다음 네 가지다.

```
section_name
section_order
device_type
viewport_width
```

---

# 5. 중복 이벤트 판단 기준

## 정상적인 중복

다음 상황에서는 이벤트가 다시 발생해도 정상이다.

```
페이지 새로고침
브라우저 탭 다시 열기
새로운 세션 시작
```

예:

```
portfolio_viewed
section_viewed { intro }
```

가 새로 들어올 수 있다.

## 비정상적인 중복

같은 페이지를 새로고침하지 않은 상태에서:

```
Capability 진입
→ 위로 이동
→ 다시 Capability 진입
```

했을 때 같은 `section_viewed`가 반복 발생하면 안 된다.

현재 훅은 `hasTracked`로 각 Stage를 한 번만 기록한다.

```
consthasTracked=useRef(false);
```

---

# 6. `Unexpected` 표시 의미

Amplitude에서 다음처럼 표시될 수 있다.

```
Unexpected
```

이것은 오류가 아니다.

의미:

```
이벤트와 속성은 정상 수집됨
하지만 Tracking Plan에 아직 등록되지 않음
```

확인할 수 있는 상태:

```
이벤트 목록에 section_viewed가 존재함
속성 목록에 section_name 등이 존재함
Live Events에서 실제 값이 보임
```

위 세 가지가 충족되면 수집은 정상이다.

---

# 7. 이벤트별 용도

## `Page Viewed`

```
페이지가 몇 번 조회됐는가
```

페이지 새로고침도 포함될 수 있다.

## `portfolio_viewed`

```
포트폴리오 핵심 사용자 흐름의 시작점
```

향후 퍼널의 첫 단계로 사용한다.

## `section_viewed`

```
방문자가 실제로 어느 주요 구간까지 내려왔는가
```

다음 도달률을 계산하는 데 사용한다.

```
Capability 도달률
Contact 도달률
기기별 섹션 도달 차이
```

---

# 8. 현재 분석할 수 있는 지표

## 전체 방문 규모

Amplitude의 사용자 및 세션 기준으로 본다.

```
고유 방문자 수
세션 수
페이지뷰 수
평균 세션 시간
기기 및 브라우저 분포
유입 정보
```

## Capability 도달률

```
Capability에 도달한 고유 사용자 수
÷ portfolio_viewed 고유 사용자 수
× 100
```

예:

```
포트폴리오 방문자: 100명
Capability 도달자: 78명

Capability 도달률: 78%
```

## Contact 도달률

```
Contact에 도달한 고유 사용자 수
÷ portfolio_viewed 고유 사용자 수
× 100
```

예:

```
포트폴리오 방문자: 100명
Contact 도달자: 32명

Contact 도달률: 32%
```

## 구간별 이탈

```
Intro 방문자: 100명
Capability 도달자: 78명
Contact 도달자: 32명
```

해석:

```
Intro → Capability 이탈: 22명
Capability → Contact 이탈: 46명
```

이 경우 Capability 내부의 길이, 정보량, 인터랙션 흐름에서 이탈이 큰지 확인할 수 있다.

---

# 9. 모바일·데스크톱 비교 기준

`device_type` 기준으로 나눠 본다.

예:

| 기기    | 포트폴리오 방문 | Capability 도달 | Contact 도달 |
| ------- | --------------- | --------------- | ------------ |
| Mobile  | 70              | 42              | 14           |
| Desktop | 30              | 25              | 18           |

도달률:

```
모바일 Capability 도달률
= 42 ÷ 70
= 60%

데스크톱 Capability 도달률
= 25 ÷ 30
= 83%
```

이처럼 모바일 도달률이 현저히 낮으면 모바일 스크롤 구조나 인터랙션을 우선 점검한다.

---

# 10. 데이터 해석 시 주의할 점

## 이벤트 수와 사용자 수를 혼동하지 않기

```
section_viewed 50회
```

가 반드시 50명을 의미하지는 않는다.

같은 사용자가 다른 세션에서 다시 방문할 수 있기 때문이다.

도달률을 계산할 때는 가능하면:

```
고유 사용자 수
```

를 기준으로 본다.

## 표본이 너무 작을 때 단정하지 않기

```
방문자 8명 중 Contact 도달 2명
→ 25%
```

이라고 해도 표본이 너무 작으므로 개선 근거로 단정하기 어렵다.

초기 기준:

```
50명 미만
→ 방향성만 참고

50~100명
→ 큰 차이 위주로 해석

100명 이상
→ 기기 및 구간 비교 시작
```

## 체류 시간이 길다고 무조건 좋은 것은 아님

길게 본 것일 수도 있지만, 인터랙션이 불편해서 오래 걸린 것일 수도 있다.

체류 시간은 반드시 도달률과 함께 본다.

---

# 11. 문제가 있을 때 점검

## 이벤트가 전혀 안 들어올 때

```
.env.local API Key 확인
개발 서버 재시작 확인
브라우저 광고 차단기 확인
Console 오류 확인
Network에서 Amplitude 요청 확인
```

## `portfolio_viewed`만 들어올 때

```
각 Stage에서 useSectionViewTracking을 호출했는지 확인
stageRef가 section 요소에 연결됐는지 확인
```

## Intro만 들어올 때

```
Capability와 Contact까지 실제로 스크롤했는지 확인
IntersectionObserver 설정 확인
Stage가 화면 중앙 감지 영역을 통과하는지 확인
```

## 실제 속성 값이 안 보일 때

이벤트 정의 화면이 아니라 다음에서 확인한다.

```
Live Events
→ 사용자 선택
→ section_viewed 선택
→ Event Properties 확인
```

---

# 12. 완료 체크리스트

```
[ ] 페이지 진입 시 Start Session 발생
[ ] 페이지 진입 시 Page Viewed 발생
[ ] 페이지 진입 시 portfolio_viewed 발생
[ ] Intro 진입 시 section_name: intro 발생
[ ] Capability 진입 시 section_name: capability 발생
[ ] Contact 진입 시 section_name: contact 발생
[ ] section_order가 1, 2, 3으로 구분됨
[ ] device_type이 정상적으로 들어옴
[ ] viewport_width가 실제 화면 너비와 비슷함
[ ] 같은 페이지 실행 중 동일 섹션 이벤트가 반복되지 않음
```

## 한 줄 기준

Live Events에서 방문·세션 이벤트와 intro, capability, contact 섹션 도달 값이 각각 확인되면 정상 작동한 것이다.
