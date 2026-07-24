# FSU EAFC FUT Web Enhancer 개선 프로젝트

GreasyFork에 공개된 `【FSU】EAFC FUT WEB 增强器`를 분석하고, FC26 환경에서 안전하게 유지보수·개선하기 위한 작업 저장소다.

## 프로젝트 상태

- 현재 포크 버전: `26.09.8`
- 원본 기준 버전: `26.09`
- userscript: [`fsu-eafc-fut-web.user.js`](./fsu-eafc-fut-web.user.js)
- 원본 SHA-256: `A2E0BEB018921CDD334D68BD4AF9BEE843F9F391A98E01E3B27D67E74D6B9634`
- 원본 출처: [GreasyFork 코드 페이지](https://greasyfork.org/en/scripts/431044-fsu-eafc-fut-web-%E5%A2%9E%E5%BC%BA%E5%99%A8/code)

`6e3cc9f` 커밋은 내려받은 원본을 수정하지 않은 기준점이다.

Club 허브의 `My Players` 대시보드에서는 Club과 SBC Storage의 선수를 카드 그리드로 보고, 안정적으로 판별 가능한 속성으로 필터링하며, 열 수를 2~4개로 조절할 수 있다. 선수 카드를 누르면 모바일 대응 대형 상세 팝업에서 기본 정보, PlayStyle/PlayStyle+, 6대 능력치와 세부 능력치를 확인할 수 있다.

진화 선수 검색 화면에는 기본 `검색` 버튼 아래 `Browse Eligible Players` 진입점이 추가된다. 현재 진화 조건을 통과한 Club 선수만 같은 필터·카드 그리드로 탐색할 수 있고, 카드 선택은 EA의 기존 진화 미리보기 및 오류 처리 경로로 전달된다.

## Edge / Tampermonkey 설치

1. 기존 GreasyFork판 FSU가 설치되어 있다면 먼저 **비활성화**한다.
2. Edge 주소창에서 [이 저장소의 userscript 설치 링크](https://raw.githubusercontent.com/jonghyunseo/fsu-fc26-enhancer/main/fsu-eafc-fut-web.user.js)를 직접 연다.
3. Tampermonkey 설치 화면에서 버전이 `26.09.8` 이상인지 확인하고 **설치**를 누른다.
4. 같은 이름의 FSU 스크립트가 두 개 동시에 활성화되지 않게 한다.

Tampermonkey의 **도구 → URL에서 가져오기** 화면에서 `디코딩 중` 상태가 오래 지속되면 해당 작업을 취소하고 위 링크를 주소창에서 직접 여는 방식을 먼저 사용한다. 이 포크는 설치 지연을 줄이기 위해 원본에 중복 선언되어 있던 Lodash CDN 의존성 하나를 제거했다.

## 문서

- [코드 분석 및 목표 설계](./docs/ANALYSIS.md)
- [My Players 대시보드 설계](./docs/CLUB_PLAYERS_DASHBOARD.md)
- [진화 선수 선택기 설계](./docs/ACADEMY_PLAYER_SELECTOR.md)
- [원본 출처 및 라이선스](./docs/UPSTREAM.md)

분석 문서에는 다음 내용이 포함된다.

- 실제 활성·비활성 기능 구분
- EA Web App 후킹 구조와 초기화 흐름
- 전역 상태, 영속 저장소, 외부 API 구조
- 선수 카드, 가격, 이적시장, SBC, 팩, 목표, 진화 기능
- 확인된 로직 오류와 보안·운영 위험
- 목표 모듈 구조와 단계별 개선 로드맵
- 테스트 전략

## 저장소 구조

```text
.
├─ fsu-eafc-fut-web.user.js  # GreasyFork 26.09 기반 개선 userscript
├─ docs/
│  ├─ ANALYSIS.md            # 분석 및 개선 설계 기준서
│  ├─ CLUB_PLAYERS_DASHBOARD.md
│  ├─ ACADEMY_PLAYER_SELECTOR.md
│  └─ UPSTREAM.md            # 원본 출처와 라이선스
├─ LICENSE
└─ README.md
```

## 개선 우선순위

1. 선수 필터, 오류 코드, 미정의 변수, 완료되지 않는 Promise 등 P0 결함 수정
2. 구매·판매·SBC 제출·팩 개봉에 확인, 취소, 상한과 dry-run 적용
3. `unsafeWindow` 노출과 `@connect *` 축소
4. `HookManager`, `ApiClient`, `SettingsStore`, `OperationScope`, `EaAdapter` 추출
5. 읽기 전용 기능부터 기능별 모듈로 이전
6. 한국어, 가격 신뢰도, 케미스트리와 SBC 템플릿 기능 개선

## 작업 원칙

- 원본 기준선은 직접 수정하지 않고 변경 목적별 커밋을 남긴다.
- 코드에 존재하는 기능과 실제 도달 가능한 기능을 구분한다.
- EA 비공개 API 변경은 capability 검사로 격리한다.
- 재고나 코인을 변경하는 기능은 기본적으로 안전한 방향으로 동작한다.
- 외부 API 응답은 timeout과 스키마 검증을 거친다.
- 기능 개선에는 회귀 테스트 또는 재현 가능한 검증 근거를 함께 추가한다.

## 주의

이 코드는 EA FC Web App의 비공개 내부 객체를 후킹하며 구매, 판매, SBC 제출과 팩 개봉을 자동화할 수 있다. 계정 및 아이템에 영향을 주는 기능은 테스트 계정과 제한된 대상에서 먼저 검증해야 한다.

## 라이선스와 출처

원본 userscript는 `Futcd_kcka`가 MIT 라이선스로 공개한 코드다. 원본 출처와 기준 버전은 [UPSTREAM.md](./docs/UPSTREAM.md), 라이선스 전문은 [LICENSE](./LICENSE)를 참고한다. 이 저장소는 원본 작성자의 공식 저장소가 아니다.
