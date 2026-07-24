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
                      ├─ 열 수 × 6장 단위 추가 렌더링
                      └─ 카드 TAP
                           └─ openClubPlayerDetail
                                ├─ ensureClubPlayerDetailMetadata
                                ├─ getClubPlayerDetailStatSource
                                ├─ getClubPlayerDetailData
                                └─ EADialogViewController
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

표의 `아니요`는 대량 목록의 필터로 사용하지 않는다는 의미다. PlayStyles, 세부 능력치, 신체 정보는 선택한 한 선수의 상세 팝업에서 제한적으로 표시한다.

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
- 선수 카드 TAP: 내부 스크롤이 있는 대형 선수 상세 팝업
- 카드 하단: Club/SBC Storage와 거래 가능 여부 배지
- 빈 결과: 명시적인 no-results 메시지

기존 FSU와 EA Web App의 어두운 색상, 타이포그래피, 카드 렌더러를 재사용한다. FUT.GG의 코드는 복제하지 않고 필터 분류만 참고했다.

## 선수 상세 팝업

[FUT.GG 선수 상세](https://www.fut.gg/players/231747-kylian-mbappe/26-151226691/)의 정보 계층과 [PlayStyles 분류](https://www.fut.gg/playstyles/)를 참고하되, 화면과 데이터 어댑터는 FSU 안에서 새로 구현했다. 외부 FUT.GG 선수 API를 호출하거나 클럽 데이터를 전송하지 않는다.

팝업은 다음 순서로 구성한다.

1. EA `UTItemViewFactory`가 렌더링한 실제 보유 카드
2. 이름, OVR, 주 포지션, 희귀도
3. 6대 카드 능력치
4. 국가, 클럽, 리그, 주발, 개인기, 약한 발, 부 포지션, 신체 정보, 저장 위치, 거래 상태, EA/아이템 ID
5. PlayStyles+와 일반 PlayStyles
6. 필드 선수 29개 또는 골키퍼 8개 세부 능력치

PlayStyle+는 `getPlusPlayStyles()`, 일반 PlayStyle은 `getBasicPlayStyles()`를 우선 사용한다. 표시 이름은 EA 객체와 현지화 값을 먼저 사용하고, 없으면 `PlayerTrait` 열거형 이름, 마지막에는 trait ID로 안전하게 대체한다.

선수 메타데이터가 이미 EA 저장소에 있으면 `repositories.PlayerMeta`에서 읽어 현재 보유 아이템에 `setMetaData()`로 연결한 뒤 팝업을 만든다. 아직 없으면 해당 선수 한 명만 `PlayerMetaData.updateItemPlayerMeta()`로 요청하고, 응답 후에도 같은 연결 단계를 거친다. 응답이 없더라도 2.5초 후 사용 가능한 데이터로 계속 열기 때문에 메타데이터 실패가 화면 전체를 무기한 막지 않는다.

세부 능력치는 먼저 메타데이터와 아이템의 업그레이드 정보를 반영한 `getSubAttribute()` 값을 사용한다. 각 대표 능력치와 EA 가중치로 다시 계산한 세부 능력치가 1보다 크게 어긋나면, 특수·진화 카드에서 EA가 업그레이드된 대표값만 노출한 경우로 판단한다. 이때 기존 FSU의 `fgCreateVirtualPlayers()`를 사용해 명시적인 세부 능력치 override는 보존하고 나머지만 대표값에 맞게 보정한다. 보정값을 사용한 경우 팝업 하단에 추정값임을 표시한다. 즉, 원본 세부값이 일치하는 일반 카드는 변경하지 않는다.

EA 현지화 서비스가 없는 키 앞에 `*`를 붙여 반환하면 이를 번역 결과로 사용하지 않고 안정적인 영문 세부 능력치 이름으로 대체한다.

반응형 구조는 다음과 같다.

- 데스크톱: 최대 1040px 팝업, 카드와 요약을 2열로 배치하고 세부 능력치를 3열로 표시
- 태블릿: 세부 능력치를 2열로 축소
- 모바일: `100dvh` 기반으로 실제 브라우저 표시 영역 안에 팝업을 고정하고, 카드와 요약을 1열로 쌓으며 PlayStyle 및 세부 능력치를 1열로 표시
- 닫기: 모든 화면에서 오른쪽 상단에 44×44px `×` 버튼을 고정하고 EA 다이얼로그의 기본 `TAP` 종료 동작에 연결
- 레이어: 상세 팝업의 EA 모달 컨테이너를 전용 레이어로 올리고, 열려 있는 동안 뒤쪽 sticky 필터 툴바의 레이어를 낮춰 필터가 팝업을 관통하지 않게 함
- 모바일 제목: EA 다이얼로그 header를 52px 고정 영역으로 유지하고 선수명에 한 줄 말줄임을 적용해 flex 축소로 글자가 위쪽에서 잘리지 않게 함
- 선수 잠금: 카드 확대율과 분리된 86×36px 버튼을 카드 스테이지 오른쪽 상단에 배치하고 `Lock/Unlock` 상태와 접근성 레이블을 함께 갱신
- Android Edge를 포함한 터치 환경: 네이티브 `EventType.TAP`, `touch-action: manipulation`, 팝업 내부 관성 스크롤 사용

기존 하단 확인 버튼은 긴 본문과 모바일 브라우저 도구 모음 때문에 화면 밖으로 밀릴 수 있어 이 상세 팝업에서만 숨긴다. 닫기 동작은 항상 보이는 상단 `×`가 담당하며 다른 FSU 팝업의 버튼 구성에는 영향을 주지 않는다.

선수 잠금은 원본 FSU의 카드 후크가 제공하는 로컬 제외 목록 기능이다. `lock_26` 설정만 변경하며 EA 서버의 선수 아이템을 이동하거나 거래 상태를 바꾸지 않는다. 상세 팝업에서는 전역 버튼 스타일과 카드 확대가 겹치지 않도록 크기와 위치만 별도로 격리한다.

## 수명주기와 안전성

- 화면 종료 시 생성한 각 `UTItemView`에 `dealloc()`을 호출한다.
- 필터 변경 또는 화면 종료 시 카드 TAP용 `UTButtonControl`도 `dealloc()`한다.
- 상세 팝업 종료 시 팝업 안의 대형 `UTItemView`를 함께 `dealloc()`한다.
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
- 필드 선수 상세 데이터 6개 그룹과 29개 세부 능력치 생성
- 골키퍼 상세 데이터 6개 그룹과 8개 골키퍼 세부 능력치 생성
- 대표 PAC 92, 원본 세부값 76/78 모형을 기존 FSU 방식으로 91/93에 보정하고 가중 대표값이 92가 되는지 확인
- 이미 일치하는 일반 선수는 보정 함수를 호출하지 않고, 골키퍼 대표·세부 능력치도 별도 구성으로 일치 판정
- 저장소에 이미 있는 메타데이터와 비동기 로드한 메타데이터가 모두 현재 아이템에 연결되는지 확인
- PlayStyle+와 일반 PlayStyle 분리 및 중복 제거
- 상세 DOM의 정보, PlayStyle, 세부 능력치 섹션 생성
- 카드와 상세 팝업 `UTItemView`, TAP 컨트롤의 수명주기 정리
- 상단 닫기 버튼의 44×44px 터치 영역, EA 기본 종료 이벤트 연결, 모바일 동적 뷰포트 규칙 확인
- 상세 모달이 뒤쪽 sticky 필터보다 높은 레이어를 유지하고 종료 시 임시 body/modal 클래스가 제거되는지 확인
- 모바일 제목 header가 52px 아래로 축소되지 않으며 긴 선수명도 세로로 잘리지 않는지 확인
- 상세 카드 잠금 버튼의 고정 크기·오른쪽 상단 배치와 잠금 상태 `aria-pressed` 동기화 확인
- 상세 기능이 FUT.GG 또는 다른 외부 API 요청을 만들지 않음

EA Web App에서 수동으로 확인해야 할 항목:

1. Club 허브에 `My Players` 타일과 수량이 표시되는가
2. 타일에서 뒤로 가기가 가능한 독립 화면으로 이동하는가
3. Club 및 SBC Storage 카드가 올바른 위치 배지를 갖는가
4. 각 필터의 실제 결과 수가 EA 기본 클럽 검색 결과와 일치하는가
5. 2·3·4열 변경 후 카드가 잘리거나 겹치지 않는가
6. 모바일에서 4열까지 선택 가능하고 화면이 가로로 넘치지 않는가
7. 모바일에서 `Load More`를 누를 때 표시 수가 열 수 × 6만큼 증가하는가
8. 반복 필터링과 화면 재진입 후 카드 뷰 또는 이벤트가 누적되지 않는가
9. 일반 선수와 골키퍼 카드를 TAP하면 각각 올바른 능력치 그룹의 상세 팝업이 열리는가
10. PlayStyle+와 일반 PlayStyle 아이콘 및 이름이 실제 카드 정보와 일치하는가
11. 특수·진화 카드의 대표 능력치와 세부 능력치가 일치하며, 추정값을 사용했다면 하단 안내가 표시되는가
12. 팝업을 반복해서 열고 닫은 뒤 카드 렌더러나 TAP 이벤트가 누적되지 않는가
13. Android Edge 세로 화면에서 팝업이 가로로 넘치지 않고 내부 스크롤과 오른쪽 상단 `×`가 동작하는가
14. 잠금 버튼이 카드 확대율과 관계없이 같은 크기로 표시되고 잠금·해제 후 레이블과 상태가 즉시 바뀌는가
15. 목록을 스크롤한 상태에서 상세 팝업을 열어도 필터가 팝업 위에 나타나지 않고 선수명이 상단에서 잘리지 않는가

## 후속 개선 후보

- Unassigned와 Transfer List를 완전하게 페이지네이션한 뒤 선택적 scope로 추가
- 필터 상태 URL/로컬 프리셋
- 다중 선택 포지션·리그·국가 필터
- 신뢰도와 갱신 시각을 포함한 선택적 가격 필터
- 대형 클럽용 IntersectionObserver 또는 가상 스크롤
- EA Web App 어댑터를 분리한 DOM 통합 테스트
