# My Players 대시보드 설계

## 목적

EA FC Web App의 클럽 허브에서 `In Packs`와 유사한 카드 그리드로 보유 선수를 탐색한다. 이번 기능은 읽기 전용이며 선수 이동, 판매, SBC 제출 같은 재고 변경 동작을 수행하지 않는다.

## 데이터 범위

현재 화면이 포함하는 소스는 다음 두 곳이다.

- Club: `repositories.Item.club.items.values()`
- SBC Storage: `repositories.Item.getStorageItems()`

각 아이템은 `id`로 중복 제거하고 `isPlayer()`가 참인 실제 카드만 사용한다. Concept 카드는 제외한다.

따라서 화면의 `My Players`는 정확히 `Club + SBC Storage`를 뜻한다. 다음 위치는 완전한 로딩을 보장할 수 없어 포함하지 않는다.

- Unassigned
- Transfer List
- Transfer Targets

이 범위는 화면 상단의 scope 배지에도 표시한다. 향후 다른 pile을 포함하려면 각 저장소의 완전한 새로고침과 페이지네이션을 먼저 보장해야 한다.

## 진입과 데이터 흐름

```text
UTClubHubView.clearTileContent
  └─ My Players 타일 + 현재 카드 수
       └─ events.goToClubPlayers
            └─ clubPlayersController
                 └─ clubPlayersControllerView
                      ├─ getClubPlayerDashboardRecords
                      ├─ filterClubPlayerDashboardRecords
                      ├─ UTItemViewFactory.createLargeItem
                      └─ 열 수 × 6장 단위 추가 렌더링
```

클럽 허브 타일은 기존 `Locked Players` 타일과 같은 `UTTileView` 패턴을 사용한다. 상세 화면은 기존 `In Packs` 화면처럼 `EAViewController`와 `EAView` 조합으로 구성하며, 선수 카드는 EA의 `UTItemViewFactory`로 렌더링한다.

## 필터 선정 기준

[FUT.GG FC 26 Players](https://www.fut.gg/players/)의 현재 필터 구성을 기준으로 삼되, 보유 카드 객체에서 안정적으로 판별할 수 있는 필드만 채택했다.

| FUT.GG 계열 | 구현 | FSU/EA 데이터 |
|---|---:|---|
| 선수명 검색 | 예 | `_staticData.name` |
| Min/Max OVR | 예 | `rating` |
| 포지션 | 예 | `possiblePositions` |
| Primary only | 예 | `preferredPosition` |
| Quality | 예 | `isBronzeRating()`, `isSilverRating()`, `isGoldRating()`, `isSpecial()` |
| Nation | 예 | `nationId` |
| League | 예 | `leagueId` |
| Club | 예 | `teamId` |
| Rarity | 예 | `rareflag` |
| Skill Moves | 예 | `getSkillMoves()` |
| Weak Foot | 예 | `getWeakFoot()` |
| Strong Foot | 예 | `isLeftFoot()` |
| Gender | 예 | `gender` |
| Price | 아니요 | 모든 보유 카드에 대해 신뢰할 수 있는 최신 시세가 기본 로딩되지 않음 |
| PlayStyles / Roles | 아니요 | 버전과 메타데이터 로딩 상태에 따라 불완전할 수 있음 |
| 세부 능력치 | 아니요 | PlayerMeta 지연 로딩 필요 |
| 키, 체중, Body Type, Accelerate | 아니요 | 외부 또는 지연 메타데이터 의존 |
| SBC/Objective/Market 상태 | 아니요 | 보유 카드 엔티티만으로 현재 배포 상태를 확정할 수 없음 |

보유 현황에 필요한 다음 필터는 별도로 추가했다.

- Location: Club / SBC Storage
- Trade Status: Tradeable / Untradeable

검색은 대소문자와 발음 구별 부호를 무시한다. 예를 들어 `fabian`으로 `Fabián`을 찾을 수 있다.

## 정렬과 열 수

지원 정렬은 다음 네 가지다.

- Rating: High to Low
- Rating: Low to High
- Name: A-Z
- Name: Z-A

열 수는 2, 3, 4 중 선택한다. 선택값은 `GM_setValue("clubplayers.columns", value)`로 저장되어 다음 진입에도 유지된다.

초기 렌더링 수는 `열 수 × 6`이다.

| 열 수 | 최초 표시 |
|---:|---:|
| 2 | 12장 |
| 3 | 18장 |
| 4 | 24장 |

`Load More`를 누르면 같은 수만큼 추가한다. 필터 변경 시 기존 `UTItemView`를 dealloc하고 첫 배치부터 다시 렌더링한다. 이 방식은 수백~수천 장의 카드를 한 번에 DOM에 생성하는 비용을 제한한다.

모바일 EA Web App에서 탭이 누락되지 않도록 열 수, 필터 초기화, `Load More` 액션은 `UTStandardButtonControl`과 `EventType.TAP`을 사용한다. 배치 렌더링은 중복 실행을 잠그며, 특정 카드의 `UTItemView` 생성이 실패해도 평점·이름 fallback을 표시하고 다음 카드 및 페이지로 계속 진행한다.

## UI 구조

- 상단: 결과 수, 데이터 범위, 2~4열 선택, 필터 초기화
- 필터: 데스크톱 6열 기반의 조밀한 작업 영역, 좁은 화면에서 4열·3열로 축소
- 모바일: 2열 필터와 사용자가 선택한 2~4열 카드 그리드
- 카드 하단: Club/SBC Storage와 거래 가능 여부 배지
- 빈 결과: 명시적인 no-results 메시지

기존 FSU와 EA Web App의 어두운 색상, 타이포그래피, 카드 렌더러를 재사용한다. FUT.GG의 코드는 복제하지 않고 필터 분류만 참고했다.

## 수명주기와 안전성

- 화면 종료 시 생성한 각 `UTItemView`에 `dealloc()`을 호출한다.
- 검색 입력은 180ms debounce를 적용한다.
- 현재 보이는 배치만 `events.loadPlayerInfo()`에 전달한다.
- 필터와 정렬은 메모리상의 레코드만 변경하며 EA 재고 API를 호출하지 않는다.
- 새 기능은 선수 이동, 판매, 구매, SBC 제출 요청을 만들지 않는다.

## 검증

자동으로 확인한 항목:

- userscript 전체 JavaScript 문법 컴파일
- 모든 `clubplayers.*` 정적 현지화 키 존재
- 악센트 무시 이름 검색
- OVR 범위
- 주 포지션과 부 포지션 구분
- 성별, 저장 위치, 거래 상태, Weak Foot 복합 필터
- 평점 오름차순
- Club과 SBC Storage 간 동일 아이템 ID 중복 제거

EA Web App에서 수동으로 확인해야 할 항목:

1. Club 허브에 `My Players` 타일과 수량이 표시되는가
2. 타일에서 뒤로 가기가 가능한 독립 화면으로 이동하는가
3. Club 및 SBC Storage 카드가 올바른 위치 배지를 갖는가
4. 각 필터의 실제 결과 수가 EA 기본 클럽 검색 결과와 일치하는가
5. 2·3·4열 변경 후 카드가 잘리거나 겹치지 않는가
6. 모바일에서 4열까지 선택 가능하고 화면이 가로로 넘치지 않는가
7. 모바일에서 `Load More`를 누를 때 표시 수가 열 수 × 6만큼 증가하는가
8. 반복 필터링과 화면 재진입 후 카드 뷰 또는 이벤트가 누적되지 않는가

## 후속 개선 후보

- Unassigned와 Transfer List를 완전하게 페이지네이션한 뒤 선택적 scope로 추가
- 필터 상태 URL/로컬 프리셋
- 다중 선택 포지션·리그·국가 필터
- 신뢰도와 갱신 시각을 포함한 선택적 가격 필터
- 대형 클럽용 IntersectionObserver 또는 가상 스크롤
- EA Web App 어댑터를 분리한 DOM 통합 테스트
