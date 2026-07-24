# 진화 선수 선택기 설계

## 검토 결론

기존 `My Players` 카드 목록과 필터를 진화 선수 선택 화면에 재사용할 수 있다. EA Web App의 진화 흐름에는 이미 다음 두 단계가 분리되어 있기 때문이다.

1. 현재 진화 슬롯에 맞는 선수를 찾는 검색 단계
2. 선택한 아이템 ID로 진화 미리보기를 요청하고 기존 진화 상세 화면에 반영하는 단계

FSU의 카드 그리드는 1번 화면만 대체한다. 실제 선택 이후에는 EA의 `UTAcademyPlayerFromClubViewController.prototype.onCellSelected()`를 현재 컨트롤러 문맥으로 호출한다. 따라서 `getSlotPreview(slotId, itemId)`, 모바일 뒤로 이동, 데스크톱 선택 알림, 서버 오류 안내는 EA의 기존 처리를 그대로 사용한다.

이 기능은 선수를 자동으로 진화에 등록하거나 비용을 지불하지 않는다. 카드 선택 후 나타나는 미리보기와 최종 확인은 기존 EA 화면에서 진행한다.

## 확인한 EA 구조

2026-07-24 현재 공개 EA Web App 번들에서 확인한 구조는 다음과 같다.

- `UTAcademyClubSearchView`
  - `_searchFilters.getSearchButton()`으로 기본 검색 버튼을 보유한다.
  - 기본 버튼의 `TAP`은 `UTAcademyClubSearchView.Events.SEARCH`로 전달된다.
- `UTAcademyClubSearchViewController`
  - `viewmodel.getSelectedSlot()`으로 현재 진화 슬롯을 얻는다.
  - 기본 검색 시 `UTAcademyPlayerFromClubViewController`를 내비게이션 스택에 추가한다.
- `UTAcademySlotEntity.meetsRequirements(player)`
  - 슬롯의 모든 `eligibilityRequirements`에 대해 EA의 클라이언트 판정기를 실행한다.
- `UTAcademyPlayerFromClubViewController.onCellSelected`
  - `viewmodel.getSlotPreview(slot.id, item.id)`를 호출한다.
  - 성공 시 모바일은 진화 상세 컨트롤러로 돌아가고 데스크톱은 `ACADEMY_ITEM_SELECTED`를 발행한다.
  - 시간 제한, 이미 적용됨, 기능 비활성화 등 EA 오류를 기존 팝업과 알림으로 처리한다.

참고한 공개 코드:

- [EA FC Ultimate Team Web App](https://www.ea.com/ea-sports-fc/ultimate-team/web-app/)
- [EA Web App 진화 UI 번들](https://www.ea.com/ea-sports-fc/ultimate-team/web-app/js/compiled_4.js?_=10821)
- [EA Web App 진화 엔티티 번들](https://www.ea.com/ea-sports-fc/ultimate-team/web-app/js/compiled_2.js?_=10821)

번들 파일명과 캐시 번호는 EA 배포에 따라 바뀔 수 있다. 구현은 클래스 존재 여부와 메서드 형태를 검사하고, 호환되지 않으면 재시도 안내만 표시한다.

## 진입점

`UTAcademyClubSearchView.init()` 후 다음 구조로 보조 버튼을 삽입한다.

```text
UTClubSearchFiltersView
  ├─ ...기본 진화 검색 필터
  ├─ .button-container
  │    └─ EA 기본 검색 버튼
  └─ .fsu-academyClubPlayersEntryRow
       └─ Browse Eligible Players
```

버튼을 기본 검색 버튼의 부모 컨테이너 뒤에 삽입하므로 같은 버튼 행 안에서 밀리거나 폭을 나눠 갖지 않는다. 모바일에서는 높이 52px, 데스크톱에서는 48px이며 부모 콘텐츠 폭을 그대로 사용한다.

중복 초기화를 막기 위해 컨트롤과 행을 `UTAcademyClubSearchView._fsu`에 보관한다. 화면이 닫힐 때 기존 `fsuDispose` 수명주기로 버튼 컨트롤을 dealloc하고 DOM 행을 제거한다.

## 데이터 범위와 적격성

진화 선택기에는 `Club`의 실제 보유 선수만 포함한다.

```text
getClubPlayerDashboardRecords()
  └─ location === "club"
       ├─ 실제 선수이며 Concept가 아님
       ├─ loans === -1
       ├─ endTime === -1
       ├─ 이미 Academy에 등록된 선수가 아님
       └─ academySlot.meetsRequirements(player) === true
```

`SBC Storage`는 제외한다. EA의 진화 미리보기는 클럽 선수의 아이템 ID를 전제로 하며, 저장소 아이템을 같은 경로에 넘기는 동작은 공개 클라이언트에서 보장되지 않는다.

적격성 판정 중 EA 데이터가 없거나 메서드가 예외를 내면 해당 카드는 목록에서 제외한다. 잘못된 선수를 적격으로 표시하는 것보다 누락시키는 fail-closed 정책이다. 생년월일이나 세부 능력치처럼 지연 메타데이터에 의존하는 조건에서는 아직 메타데이터가 없는 카드가 누락될 수 있다. 전체 클럽 메타데이터를 한꺼번에 요청하면 URL 크기와 모바일 성능 문제가 생길 수 있어 이번 버전에서는 수행하지 않는다.

카드를 누르기 직전에 같은 적격성 조건을 다시 검사한다. 목록을 연 뒤 선수 상태가 바뀌었다면 EA 미리보기 요청을 보내지 않고 변경 안내를 표시한다.

## 재사용 구조

기존 `clubPlayersControllerView`는 옵션 객체를 받는 공용 카드 브라우저가 되었다.

| 옵션 | 일반 `My Players` | 진화 선택기 |
|---|---|---|
| `records` | Club + SBC Storage를 내부에서 조회 | 현재 슬롯 적격 Club 레코드를 주입 |
| `onSelect` | 없음: 선수 상세 팝업 열기 | EA 진화 선택 처리기로 전달 |
| `scopeText` | `Club + SBC Storage` | `Club · Eligible only` |
| `emptyText` | 일반 필터 결과 없음 | 현재 진화 적격 선수 없음 |
| `hideLocationFilter` | `false` | `true` |
| `mode` | 일반 | `academy` |

선수명, OVR, 포지션, Quality, 국가, 리그, 클럽, 희귀도, 개인기, 약한 발, 주발, 성별, 거래 상태와 정렬은 기존 필터를 그대로 사용한다. 진화 선택기에는 Club만 들어오므로 Location 필터는 표시하지 않는다.

열 수 2·3·4와 `열 수 × 6` 단위 추가 렌더링도 공용 구현을 사용한다. 저장된 열 수는 일반 목록과 진화 목록에서 공유한다.

## 선택 흐름

```text
Browse Eligible Players TAP
  └─ 현재 UTAcademyClubSearchViewController 탐색
       ├─ viewmodel.getSelectedSlot()
       ├─ Club 레코드 적격성 판정
       └─ academyClubPlayersController push
            └─ 선수 카드 TAP
                 ├─ 적격성 재검사
                 └─ EA onCellSelected를 현재 컨트롤러로 호출
                      └─ getSlotPreview(slot.id, item.id)
                           ├─ 성공: 기존 진화 상세 미리보기로 복귀
                           └─ 실패: EA 기본 오류 팝업/알림
```

선택 컨트롤러는 EA의 `showTimedWarningPopup()`도 위임한다. 같은 카드를 연속으로 눌러 미리보기 요청이 중복되는 것을 줄이기 위해 1.5초 선택 잠금을 둔다.

## UI와 모바일 안전성

- 기본 검색과 시각적으로 구분되는 보조 액션을 사용한다.
- 진화 목록에서는 카드의 잠금 버튼을 숨긴다. 전체 카드 탭이 진화 선택 액션이므로 두 동작이 겹치지 않게 한다.
- 카드 하단 첫 배지는 저장 위치 대신 `Eligible`을 표시한다.
- 기존 거래 가능/불가 배지는 유지한다.
- 터치 액션은 기존 EA `UTButtonControl`과 `EventType.TAP`을 사용한다.
- 일반 `My Players` 모드의 카드 탭과 Location 필터는 그대로 유지한다.

## 자동 검증

다음 항목을 확인했다.

- userscript 전체 JavaScript 문법 파싱
- 검색 버튼 컨테이너 다음에 새 행을 삽입하는 정적 계약
- 390px 모바일 모형에서 기본 검색과 새 버튼의 좌우 정렬 및 세로 순서
- 모바일 2·3·4열에서 가로 스크롤이 생기지 않음
- Club 적격 선수는 포함되고 SBC Storage, 임대, 만료, 이미 진화 중인 선수는 제외됨
- `meetsRequirements()`가 거부하거나 예외를 내면 제외됨
- 카드 선택 시 EA 네이티브 처리기의 세 번째 인자에 동일한 선수 아이템이 전달됨
- 연속 탭이 1회만 네이티브 처리기로 전달됨
- 일반 모드는 기존 데이터 조회와 상세 팝업 콜백을 유지함

## 실제 계정에서 확인할 항목

1. 진화 상세에서 선수 검색 화면을 열면 기본 `검색` 버튼 바로 아래에 새 버튼이 보이는가
2. 새 버튼을 누르면 현재 진화 조건을 만족하는 Club 선수만 표시되는가
3. 국가·리그·클럽·OVR·포지션 등 필터와 2·3·4열이 정상 동작하는가
4. 선수를 누르면 EA의 진화 미리보기 화면으로 돌아가는가
5. 미리보기 수치와 비용이 기본 EA 검색으로 같은 선수를 선택했을 때와 같은가
6. 조건이 복잡한 진화와 생년월일 제한 진화에서 누락된 카드가 없는가
7. Android Edge에서 새 버튼, 카드 탭, 뒤로 가기와 `Load More`가 모두 동작하는가

## 후속 개선 후보

- EA 서버의 `academySlotId` 검색 결과를 안전하게 전 페이지 수집하는 어댑터
- 생년월일/세부 능력치 조건에 필요한 메타데이터의 제한적 배치 로딩
- 적격성 판정에서 제외된 이유를 카드별로 설명하는 디버그 모드
- 일반 목록과 진화 목록의 필터 상태를 별도 프리셋으로 저장
