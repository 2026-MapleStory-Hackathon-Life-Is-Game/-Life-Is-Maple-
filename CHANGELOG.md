# 변경사항 (버프 패널 + 전투력 연동 + 확률 테이블)

> 작성: 2026-06-16 / 팀 공유용

## 한 줄 요약
진영·타일 버프가 **좌상단 아이콘 카드 + 호버 툴팁**으로 표시되고, 버프·스탯 변경 시 **전투력이 즉시 재계산되어 패널에 반영**됩니다. 게임 내 **확률을 전부 CSV 테이블로 분리**해 코드 수정 없이 밸런싱할 수 있게 했습니다.

---

## 1. 버프 패널 UI (진영/타일 버프 표시)
- **무엇**: 버프를 얻으면 좌상단에 **버프 아이콘 카드**가 가로로 나열되고, 카드에 **마우스를 올리면 효과 툴팁**(이름/설명/올스탯 +10% 등)이 뜨고 떼면 사라짐.
- **데이터 흐름**: 서버 `BuffManager`가 버프 적용 → 해당 클라이언트로 활성 버프 목록 push → 클라 `BuffPanelUI`가 카드 생성.
- **신규**: `RootDesk/MyDesk/UI/BuffPanelUI.mlua`
- **수정**: `RootDesk/MyDesk/Battle/BuffManager.mlua`, `ui/InGameUIGroup.ui`
- **기술 결정(중요)**:
  - PC 호버 감지는 `ButtonComponent`의 **`ButtonStateChangeEvent`** 사용 (UITouchEnter는 호버용 아님).
  - 버프 카드는 **자식 없는 단일 버튼**(아이콘=버튼 배경)으로 구성 — MSW에서 버튼의 자식 UI, builder remove+재생성한 자식이 refresh로 런타임 재생성되지 않는 이슈를 우회.

## 2. 올스탯(AllStat) 버프 반영
- **무엇**: 기존엔 BuffData의 `AllStat_*` 컬럼을 안 읽어 진영 버프의 **올스탯 +10%가 적용되지 않던 상태** → 올스탯을 주스탯 항목에 합산하도록 수정.
- **수정**: `BuffManager.mlua` (`SetBuff`)

## 3. 전투력 즉시 재계산 + 패널 표시 연동
- **무엇**: 버프 적용/해제 시 **전투력 즉시 재계산**(`OnBuffsChanged`)되고, 좌상단 패널의 **"전투력: …" 텍스트가 실시간 갱신**.
- **이유**: 기존엔 버프를 저장만 하고 재계산을 안 해 게임 종료 판정에 버프가 누락될 수 있었고, 패널 전투력은 정적 placeholder였음.
- **수정**: `BuffManager.mlua`(재계산 트리거), `RootDesk/MyDesk/UI/PlayerInfoPanel.mlua`(전투력 표시 연동). `CombatPowerManager`는 **읽기만** 함(변경 없음).
- **검증**: 기본 607.5 → 올스탯 +10% → 668.25(+10%) 즉시 반영 확인.

## 4. 확률 시스템 → CSV 테이블화 (밸런싱 용이)
모든 확률을 코드에서 분리해 CSV로 관리. **확률 변경 시 코드 수정 불필요, CSV 숫자만 수정.**

- **신규 폴더 `RootDesk/MyDesk/Probability/`** (각 `.csv` + `.userdataset` 페어):

  | 파일 | 용도 | 기본값 |
  |---|---|---|
  | `VillageProbability.csv` | 마을 룰렛 | 대성공 20 / 성공 50 / 실패 30 |
  | `GuildProbability.csv` | 길드 임무 | 성공 70 / 실패 30 |
  | `LoveProbability.csv` | 연애 데이트 | 성공 60 / 실패 40 |
  | `BossDropRate.csv` | 보스 장신구 드롭률 | 보스별·장신구별 가중치 |

- **공용 로직**: `HexaTile:RollOutcome(테이블명)`, `BossTile:RollDrop(보스명)` — Weight 가중치로 결과 추첨.
- **수정**: `HexaTile.mlua`, `VillageTile.mlua`, `GuildTile.mlua`, `UnionTile.mlua`, `BossTile.mlua` (하드코딩 확률 → 테이블 읽기로 교체).
- **설계 결정**: **보스 성공 판정(대성공/성공/실패)은 전투력 기반 유지**(확률 테이블 제외) — 전투력 빌드의 의미 보존.
- **검증(1000회 표본)**: Village 18/53/28, Guild 69/30, Union 58/41, 보스 드롭 Zakum 51/33/15 · Horntail 40/31/18/9 — 전부 CSV 가중치와 일치.

## 5. 버프 Tier 시스템 + 연애/유니온 칸 통합
- **Tier 기반 슬롯 교체**: `BuffData.csv`에 `Tier` 컬럼 추가. 같은 카테고리 슬롯은 **새 버프 Tier가 같거나 높을 때만 교체**(더 낮으면 무시 → 상위 버프 강등 방지). `Buff/BuffManager.mlua` `SetBuff`.
- **연애 칸 = 유니온 칸으로 통합**: 첫 데이트(T1) → 웨딩 팡파르(T2) → 유니온의 축복(T3)이 모두 `Union` 카테고리. 칸 방문 성공 시 **한 단계씩 승급**(`BuffManager:UpgradeSlot("Union")`).
  - `LoveTile.mlua` **삭제** → `UnionTile.mlua` 신규. `HexaTileManager` 타일 타입 `Love` → `Union`.
  - 확률 테이블 `LoveProbability` → `UnionProbability`로 이름 정리.
- **검증**: 승급 1→2→3단계 정상, 최고 단계에서 추가 승급 없음. 동일 Tier 진영 재선택 시 교체됨.

## 6. 스탯 모델: 주스탯 1개 → STR/DEX/INT/LUK 4개 개별 + 올스탯 분리
- **이유**: 진영 효과가 "주스탯 10%" → "올스탯 10%"로 변경됨. 기획자 요청으로 4대 스탯을 개별 관리.
- **변경**:
  - `CombatPowerManager`: 스탯 값을 **`STR/DEX/INT/LUK` 4개**로 분리. 직업 주스탯(`MainStatType`)이 그 중 하나를 가리키고(미설정 시 STR), 나머지 3개가 부스탯.
  - 유효스탯 = `주스탯_적용 + (부스탯 3개 합)_적용 ÷ 4` (메이플식 가중).
  - 버프에서 **올스탯(`allStat*`)을 주스탯(`stat*`)과 분리** — 올스탯%는 4스탯 모두, 주스탯%는 주스탯에만. (`BuffManager`/`ExtraTableManager`/`CombatPowerManager`)
  - 사냥/세금 등 게임플레이는 **주스탯 1개만 성장**(기획 c안). 부스탯은 값이 들어오면(장비 등) 합/4로 전투력 기여.
- **검증**: STR100+공100 → 607.5 / 부스탯 DEX40 추가 → 668.25(부스탯 기여 확인) / 올스탯 10% → 735.075(주·부 모두 +10% 적용 확인).

---

## 신규 / 수정 파일 (이번 작업)
- **신규**:
  - `RootDesk/MyDesk/UI/BuffPanelUI.mlua`
  - `RootDesk/MyDesk/HexaTile/Tiles/UnionTile.mlua`
  - `RootDesk/MyDesk/Probability/{VillageProbability, GuildProbability, UnionProbability, BossDropRate}.csv` (+ `.userdataset`)
- **수정**:
  - `RootDesk/MyDesk/Buff/BuffManager.mlua` (Tier 슬롯 교체 + UpgradeSlot)
  - `RootDesk/MyDesk/UI/PlayerInfoPanel.mlua`
  - `RootDesk/MyDesk/HexaTile/HexaTileManager.mlua` (타일 타입 Love→Union)
  - `RootDesk/MyDesk/HexaTile/Tiles/{HexaTile, VillageTile, GuildTile, BossTile}.mlua`
  - `ui/InGameUIGroup.ui`
- **삭제**: `RootDesk/MyDesk/HexaTile/Tiles/LoveTile.mlua`, `Probability/LoveProbability.*`

## 검증 상태
- 빌드 에러 0. 핵심 동작은 플레이 + 로그/스크린샷으로 확인.
- 호버 툴팁 실기 동작(표시/숨김) 확인 완료.

## 후속 / 밸런싱 거리 (팀 논의용)
- 확률값(마을/길드/연애 분포, 보스 드롭 가중치)은 **전부 placeholder** — 기획 확정 시 CSV만 수정.
- `BossData.csv`의 `Stage`·`DropAccessory` 컬럼은 현재 미사용이나, **여명/칠흑 장신구 세트 등 확장 대비로 의도적으로 보존**.
- 길드/웨딩/유니온 버프는 보스 칸의 **오른쪽 분기(길드 → 연애)** 에서만 획득 — 보드 경로 설계 확인 필요.
