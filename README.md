# Life Is Maple — 코드 구조 가이드

메이플스토리 월드(MSW) 기반 보드게임 프로젝트.  
주사위를 굴려 헥사 타일 위를 이동하고, 칸 이벤트를 처리하는 **싱글/멀티 지원 턴제 보드게임**입니다.

---

## 목차

1. [전체 구조 한눈에 보기](#1-전체-구조-한눈에-보기)
2. [턴 시스템](#2-턴-시스템)
3. [주사위 & 이동](#3-주사위--이동)
4. [타일 시스템](#4-타일-시스템)
5. [데이터 & 스탯 시스템](#5-데이터--스탯-시스템)
6. [UI 시스템](#6-ui-시스템)
7. [파일 위치 요약](#7-파일-위치-요약)
8. [팀원 작업 가이드](#8-팀원-작업-가이드)

---

## 1. 전체 구조 한눈에 보기

```
[플레이어가 주사위 버튼 클릭]
        ↓
   UnderButton (UI 컴포넌트)
        ↓  RequestRollDice()
   SingleTurnManager / MultiTurnManager  ←─  GameModeManager가 모드 결정
        ↓  Roll()
   DiceManager  →  @Sync로 눈값 동기화
        ↓  PlayDiceAnimation()
   RollDiceAnimationUI  →  연출 끝나면 OnDiceAnimationEnd() 콜백
        ↓  AdvanceState("MOVE")
   PlayerMovement  →  MoveSteps(N)  →  MoveToTile() 반복
        ↓ (분기 있으면 BranchSelectUI 표시)
   HexaTileManager  →  타일 그래프/엔티티 제공
        ↓  OnMoveComplete()
   SingleTurnManager / MultiTurnManager  →  TILE → TURN_END → 다음 턴
```

---

## 2. 턴 시스템

### 2-1. GameModeManager
**파일:** `RootDesk/MyDesk/Turn/GameModeManager.mlua`  
**역할:** 싱글/멀티 모드를 결정하고, 해당 턴 매니저를 시작·종료하는 **진입점**.

| 메서드 | 설명 |
|---|---|
| `StartSinglePlayer()` | 싱글 모드 시작. `_SingleTurnManager:Start()` 호출 |
| `StartMultiPlayer(playerCount)` | 멀티 모드 시작 (2~4인). `_MultiTurnManager:Start(N)` 호출 |
| `EndGame()` | 현재 활성 모드의 턴 매니저를 종료 |

**@Sync 프로퍼티:**
- `_activeMode` : `"NONE"` / `"SINGLE"` / `"MULTI"`
- `_isGameActive` : 게임 진행 중 여부

> **현재 상태:** `OnBeginPlay`에서 자동으로 `StartSinglePlayer()`를 호출 (테스트용).  
> 실제 서비스 시 모드 선택 UI 연동으로 교체 필요.

---

### 2-2. SingleTurnManager
**파일:** `RootDesk/MyDesk/Turn/SingleTurnManager.mlua`  
**역할:** 싱글플레이 턴 상태머신. 플레이어 1인의 턴 흐름을 관장.

#### 상태 순환
```
IDLE → TURN_START → DICE → MOVE → TILE → TURN_END → TURN_START → ...
```

| 상태 | 동작 |
|---|---|
| `TURN_START` | 즉시 DICE로 전진 |
| `DICE` | 주사위 버튼 입력 대기. `RequestRollDice()` 호출 받으면 진행 |
| `MOVE` | `PlayerMovement:MoveSteps(N)` 실행. 이동 완료 시 `OnMoveComplete()` 수신 |
| `TILE` | 칸 이벤트 처리 (현재 미구현, 즉시 TURN_END로 진행) |
| `TURN_END` | 즉시 다음 TURN_START로 순환 |

**외부에서 호출하는 메서드:**
- `RequestRollDice()` — UnderButton이 호출. DICE 단계에서만 유효
- `OnMoveComplete()` — PlayerMovement가 이동 완료 시 호출
- `OnDiceAnimationEnd()` — 주사위 연출 완료 시 클라이언트가 콜백

---

### 2-3. MultiTurnManager
**파일:** `RootDesk/MyDesk/Turn/MultiTurnManager.mlua`  
**역할:** 멀티플레이 턴 상태머신. SingleTurnManager와 동일한 상태 순환이지만 **플레이어 순서 관리**가 추가됨.

#### SingleTurnManager와의 차이점

| 기능 | Single | Multi |
|---|---|---|
| 플레이어 수 | 1명 고정 | 2~4명, 순서 관리 |
| 주사위 검증 | 없음 | `senderUserId == _currentPlayerUserId` 확인 |
| 이동 대상 | 전체 유저 첫 번째 | `_currentPlayerUserId`와 일치하는 유저만 |
| 퇴장 처리 | 없음 | `OnUserLeave` — 목록 재정렬, 인덱스 보정 |

**@Sync 프로퍼티:**
- `_currentPlayerUserId` : 현재 턴 플레이어의 userId (클라이언트에서 "내 차례" 판단에 사용)
- `remainDiceTotal` : 마지막 주사위 합계

---

## 3. 주사위 & 이동

### 3-1. DiceManager
**파일:** `RootDesk/MyDesk/DiceManager.mlua`  
**역할:** 주사위 두 개를 **서버에서** 굴려 결과를 반환. 싱글/멀티 공용.

```
Roll() → d1(1~6) + d2(1~6) = total(2~12)
```

**@Sync 프로퍼티:** `_dice1`, `_dice2`, `_total`  
→ 클라이언트의 연출 UI(`RollDiceAnimationUI`)가 이 값을 읽어 눈금 스프라이트를 표시.

> 주사위는 반드시 서버에서 생성해야 함. 클라이언트가 굴리면 치팅 가능.

---

### 3-2. PlayerMovement
**파일:** `RootDesk/MyDesk/PlayerMovement.mlua`  
**역할:** 플레이어 캐릭터의 타일 이동을 담당하는 `@Component`. 플레이어 엔티티에 부착.

#### 이동 흐름
```
MoveSteps(N)
  └─ MoveNextStep() 반복 호출
       ├─ 남은 칸 = 0 → 이동 완료, OnMoveComplete() 호출
       ├─ 다음 타일 = 0개 → End 타일, 정지
       ├─ 다음 타일 = 1개 → MoveToTile() (Tween 0.5초 이동) → 도착 후 MoveNextStep() 재귀
       └─ 다음 타일 ≥ 2개 → ShowBranchUI() 표시 후 대기
                               └─ 플레이어 선택 → OnBranchSelected(tileName) → MoveToTile()
```

**@Sync 프로퍼티:**
- `currentTile` : 현재 위치한 타일 이름 (예: `"HexaTile_3"`)
- `remainingSteps` : 남은 이동 칸 수 (UI 표시용)

**서버/클라이언트 역할 분리:**
- **서버:** 이동 경로 계산, 분기 판정, 완료 신호
- **클라이언트:** `MoveToTile()` — Tween 애니메이션, 방향 전환, 애니메이션 상태 변경

---

## 4. 타일 시스템

### 4-1. HexaTileManager
**파일:** `RootDesk/MyDesk/HexaTile/HexaTileManager.mlua`  
**역할:** 보드의 **논리 연결 그래프**와 **씬 엔티티**를 관리하는 `@Logic`.

#### 현재 타일 구성 (11개)
```
HexaTile_0(Start) → 1(Hunt) → 2(Hunt) → 3(Hunt) → 4(Unlucky) → 5(Hunt) → 6(Boss)
                                                                            ↓
                                                           HexaTile_7A(Village/왼쪽) → 8(End)
                                                           HexaTile_7B(Guild/오른쪽)  → 9(Hunt) → 10(End)
```

#### 타일 타입 목록
| 타입 | 설명 |
|---|---|
| `Start` | 시작 칸 |
| `Hunt` | 사냥 칸 (이벤트 미구현) |
| `Unlucky` | 세금 칸 (이벤트 미구현) |
| `Boss` | 보스 칸 / 분기점 |
| `Village` | 마을 칸 |
| `Guild` | 길드 칸 |
| `End` | 끝 칸 |

**주요 메서드:**
- `GetNextTiles(tileName)` → 다음 이동 가능 타일 목록 (0개=End, 1개=직선, 2개=분기)
- `GetTileType(tileName)` → 칸 종류 문자열
- `GetTileEntity(tileName)` → 씬의 실제 Entity (위치 정보용)
- `GetTileBranch(tileName)` → 분기 버튼에 표시할 이름 ("왼쪽"/"오른쪽")

#### 맵 설정 (map01.map)
각 타일 엔티티에는 다음 컴포넌트가 있어야 함:
- `SpriteRendererComponent` — 타일 스프라이트 표시
- `TriggerComponent` — 충돌 그룹 `HexaTile`
- `TagComponent` — 태그 `"HexaTile"` (HexaTileManager가 수집 시 사용)

---

## 5. 데이터 & 스탯 시스템

### 5-1. CombatPowerManager
**파일:** `RootDesk/MyDesk/Data/CombatPowerManager.mlua`  
**역할:** 플레이어의 전투력(스탯·공격력·대미지) 계산.

**전투력 공식:**
```
전투력 = 스탯 × 공/마 × 대미지 × 크리티컬 대미지

스탯    = (주스탯 + 스탯 가산) × (1 + 스탯%) × 4.5 / 100
공/마   = 기본공마 × (1 + 공마%) + 공마 가산
대미지  = 1 + (추가대미지% + 보스대미지%) / 100
```

**@Sync 프로퍼티:** `MainStat`, `MainAttack`, `ExtraDamage`, `BossDamage`, `CombatPower`

---

### 5-2. BuffManager
**파일:** `RootDesk/MyDesk/Data/BuffManager.mlua`  
**역할:** 활성 버프 목록 관리 및 합산.

버프 구조체:
```lua
{ id="버프ID", statFlat=0, statPct=0, attackFlat=0, attackPct=0, damagePct=0, bossDamagePct=0 }
```

- `AddBuff(buff)` — 버프 추가
- `RemoveBuff(id)` — id로 버프 제거
- `GetModifiers()` — 전체 버프 합산 modifier 반환

---

### 5-3. ExtraTableManager
**파일:** `RootDesk/MyDesk/Data/ExtraTableManager.mlua`  
**역할:** 여러 소스(버프, 길드, 유니온 등)의 modifier를 하나로 합쳐 `CombatPowerManager`에 전달.

`GetExtra()` 호출 시 현재 활성된 모든 modifier를 합산한 `extra` 테이블 반환.  
→ 새 시스템 추가 시 `AddModifiers(extra, _NewManager:GetModifiers())` 한 줄만 추가하면 됨.

---

### 5-4. JobDataSetManager
**파일:** `RootDesk/MyDesk/Data/JobDataSetManager.mlua`  
**역할:** 직업별 주스탯·공격 타입을 DataSet CSV에서 조회.

지원 직업: `Warrior`, `Archer`, `Mage`, `Thief`, `Pirate`

- `GetMainStatType(job)` → STR/DEX/INT/LUK
- `GetMainAttackType(job)` → "공격력" / "마력"

DataSet 파일: `DataSet_MainStat`, `DataSet_MainAttack` (CSV)

---

## 6. UI 시스템

### UnderButton
**파일:** `RootDesk/MyDesk/UI/UnderButton.mlua`  
**역할:** 하단 HUD — 주사위 패널 열기/닫기, 주사위 굴리기 버튼.

**핵심 로직:**
- `CanOpenDicePanel()` — DICE 단계이고 내 차례일 때만 패널 열림
  - 싱글: `_SingleTurnManager._currentState == "DICE"`
  - 멀티: 위 조건 + `myUserId == _MultiTurnManager._currentPlayerUserId`
- `OnClickRoll()` — `_SingleTurnManager:RequestRollDice()` / `_MultiTurnManager:RequestRollDice()` 동시 호출 (각 매니저가 비활성이면 내부에서 무시)

---

### RollDiceAnimationUI
**파일:** `RootDesk/MyDesk/UI/RollDiceAnimationUI.mlua`  
**역할:** 주사위 굴리기 연출 + 이동 중 남은 칸 수 표시.

- `Show(d1, d2, total, onComplete)` — 주사위 눈 스프라이트 교체 후 타이머 대기, 완료 시 `onComplete` 콜백 실행
- `ChangeDiceBuff(remaining)` — 이동 중 남은 칸 수를 화면에 표시/숨김

**주사위 눈 스프라이트:** `diceSprites` 테이블에 1~6 눈에 해당하는 RUID 6개 등록.

---

### BranchSelectUI
**파일:** `RootDesk/MyDesk/UI/BranchSelectUI.mlua`  
**역할:** 분기 타일 도착 시 방향 선택 팝업 (최대 5개 버튼).

- `Show(tileNames, branchTexts)` — 분기 수만큼 버튼 활성화, 텍스트 표시
- 버튼 클릭 → `SelectBranch(tileName)` → `player.PlayerMovement:OnBranchSelected(tileName)` 호출

---

### PlayerInfoPanel
**파일:** `RootDesk/MyDesk/UI/PlayerInfoPanel.mlua`  
**역할:** 플레이어 닉네임 표시 HUD 컴포넌트.

---

## 7. 파일 위치 요약

```
RootDesk/MyDesk/
├── Turn/
│   ├── GameModeManager.mlua      # 싱글/멀티 모드 진입점
│   ├── SingleTurnManager.mlua    # 싱글 턴 상태머신
│   └── MultiTurnManager.mlua     # 멀티 턴 상태머신
├── HexaTile/
│   └── HexaTileManager.mlua      # 타일 그래프 + 엔티티 관리
├── Data/
│   ├── CombatPowerManager.mlua   # 전투력 계산
│   ├── BuffManager.mlua          # 버프 목록 관리
│   ├── ExtraTableManager.mlua    # 전체 modifier 합산
│   └── JobDataSetManager.mlua    # 직업별 스탯/공격 타입 조회
├── UI/
│   ├── UnderButton.mlua          # 주사위 HUD 버튼
│   ├── RollDiceAnimationUI.mlua  # 주사위 연출 + 이동 버프 표시
│   ├── BranchSelectUI.mlua       # 분기 선택 팝업
│   └── PlayerInfoPanel.mlua      # 플레이어 정보 HUD
├── DiceManager.mlua              # 서버 주사위 생성
└── PlayerMovement.mlua           # 플레이어 타일 이동 컴포넌트

map/
└── map01.map                     # 헥사 타일 엔티티 배치

ui/
├── DefaultGroup.ui               # 메인 HUD (UnderButton, RollDiceAnimationUI 등)
├── BranchSelectGroup.ui          # 분기 선택 팝업 UI
└── JobSelectGroup.ui             # 직업 선택 UI
```

---

## 8. 팀원 작업 가이드

### 새 칸 이벤트 추가하기
`SingleTurnManager`와 `MultiTurnManager`의 `OnTile()` 메서드에 이벤트 로직을 구현합니다.  
이벤트 완료 후 `AdvanceState()` (싱글) 또는 `AdvanceToState("TURN_END")` (멀티)를 호출해야 다음 턴으로 넘어갑니다.

```lua
-- 예시: 헌트 칸에서 전투력 계산 후 보상 지급
method void OnTile()
    local tileType = _TileManager:GetTileType(_PlayerMovement.currentTile)
    if tileType == "Hunt" then
        -- 칸 이벤트 처리...
        -- 완료되면:
        self:AdvanceState()
    end
end
```

### 타일 추가하기
1. `HexaTileManager.mlua`의 `tileGraph`에 타일 정보 추가
2. `map01.map`에 엔티티 배치 (`TriggerComponent` + `TagComponent("HexaTile")` 필수)
3. 타일 엔티티 이름이 `tileGraph`의 key와 **정확히 일치**해야 함

### 버프 시스템 연동하기
```lua
-- 버프 추가
_BuffManager:AddBuff({ id="buff_01", statFlat=100, attackPct=5 })

-- 전투력 재계산
local power = _CombatPowerManager:CalculateCurrentCombatPower()
```

### 새 스탯 소스 추가하기 (길드, 유니온 등)
`ExtraTableManager.mlua`의 `GetExtra()`에 한 줄 추가:
```lua
self:AddModifiers(extra, _GuildManager:GetModifiers())
```

### 주의사항

- **`.codeblock` 파일은 절대 직접 수정하지 마세요.** Maker가 자동 생성하는 메타파일입니다.
- **`Global/`과 `Environment/` 폴더는 읽기 전용입니다.** MSW 에디터에서만 수정 가능합니다.
- **서버/클라이언트 역할 구분:** 게임 로직(주사위, 이동 판정)은 서버에서, 연출(Tween, 스프라이트 교체)은 클라이언트에서 실행됩니다. `@ExecSpace` 어노테이션을 확인하세요.
- **타일 이름은 정확히 일치해야 합니다.** `HexaTileManager`는 엔티티 `.Name`을 key로 사용하므로, 맵 엔티티 이름이 `tileGraph` key와 다르면 이동이 작동하지 않습니다.
