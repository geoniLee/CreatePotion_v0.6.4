# CreatePotion

필드에서 재료를 채집하고 공방에서 포션을 제작해 NPC의 요청을 해결하는
힐링 / 어드벤처 게임입니다.

- **개발 기간:** 2026.08.03 ~ 2026.10.15
- **개발 인원:** 9명
- **Engine:** Unreal Engine 5.8
- **Language:** C++ / Blueprint
- **Play Video:** [Google Drive](https://drive.google.com/file/d/1wYYHEF186lLVIjGBgIATkxz-wvopihos/view?usp=sharing)
- **Packaged Build:** [Google Drive - v0.6.4-beta](https://drive.google.com/file/d/1rq6BZ-FbZ2OYr345ob4vFyU3ls3QBtkA/view?usp=sharing)

---

## 게임 진행

플레이어는 필드에서 재료를 발견하고 채집해 도감을 채웁니다.
공방에서는 재료의 GameplayTag를 조합해 포션을 제작하고,
완성된 포션을 NPC에게 전달해 요청 결과와 보상을 확인합니다.

---

## 개인 기여

아래 내용은 Git 작성자 `geoniLee`, `geoni-ara`의 커밋과 현재 코드를 기준으로 정리했습니다.

### 1. 채집물 데이터 · 도감 시스템

채집물의 정적 데이터와 플레이 중 변경되는 도감 상태를 분리했습니다.

- `UCPForageableItemData`를 추가해 표시명, GameplayTag, 도감 텍스트 Key와 이미지 등 정적 데이터를 관리
- `UCPCodexSubsystem`을 구현해 채집물 등록, 조사 단계와 누적 채집 횟수를 런타임 상태로 관리
- 누적 채집 횟수 3회, 6회, 10회를 기준으로 최대 4단계까지 도감 설명을 해금
- 채집물 도감의 Array 기반 상태를 Map 기반 조회 구조로 변경
- NPC 도감 등록, Quest 진행에 따른 상세 설명, String Table 연동 추가
- 발견한 포션 효과 Tag와 조합식을 표시하는 Tag 조합 도감 구현
- 채집물·NPC·발견 Tag 도감 상태를 저장 및 불러오기 흐름에 연동

도감 UI는 Subsystem의 상태 변경 Delegate를 구독한 뒤 현재 Entry를 다시 조회합니다.
이를 통해 데이터 소유는 Subsystem에 두고 UI는 표시와 갱신만 담당하도록 구성했습니다.

### 2. GameplayTag 데이터 · 연금술 시스템

포션 제작에 사용하는 Tag 식별자와 화면에 표시할 정보를 분리했습니다.

- `FCPTagDefinitionRow`와 DataTable을 추가해 Tag 표시명, 조합 결과, 포션 색상 데이터를 관리
- `UCPDTSettings`를 통해 공통 Tag Definition DataTable을 참조
- 가마솥에 들어온 재료 인스턴스와 GameplayTag를 수집하고 정의된 조합 관계를 순차 적용
- 중화 Tag가 다른 효과와 동시에 남지 않도록 조합 규칙 처리
- 투입한 재료 수만큼 최종 Effect Tag 조합을 가진 포션 생성
- 새로 완성한 조합 Tag를 도감 발견 상태에 반영
- 여러 Effect Tag의 색상 데이터를 사용해 포션 액체와 충돌 VFX를 표현

가마솥의 재료, 현재 효과, 조합 결과는 `UCPCauldronSubsystem`에 보관해
레벨 이동과 저장·불러오기 이후에도 유지되도록 구성했습니다.

### 3. 공방 · Quest / NPC 통합

담당한 공방과 포션 제작 흐름을 팀의 Quest / NPC 시스템에 연결했습니다.

- 공방 벨과 NPC Spawn / Despawn 흐름 및 상호작용 Prompt 연동
- 공방 요청 상태와 NPC Quest 판정 책임 분리
- 포션이 전달한 Effect Tag를 Quest NPC의 납품 판정으로 연결
- 납품 결과의 등급과 보상을 결과 UI에 표시
- 성공 보상을 플레이어 인벤토리에 실제 지급
- Quest 완료 이후 대화와 NPC 도감 설명 갱신 연결

Quest와 NPC 시스템 전체를 구현한 것이 아니라,
공방·포션 시스템과 기존 팀 시스템 사이의 인터페이스와 결과 처리 흐름을 담당했습니다.

### 4. 상호작용 · Prop 구조 리팩터링

기존 투척 Prop과 포션 구현을 프로젝트 공용 상호작용 구조에 맞게 정리했습니다.

- Lab 상호작용 Actor를 공용 `UCPInteractableComponent` 기반 구조로 전환
- `UCPThrowableComponent`를 구현해 집기 가능 여부와 Pickup Gameplay Event 전달을 담당
- 채집물 DataAsset을 기준으로 일반 재료 Prop과 Potion의 데이터 초기화 방식 통합
- Prop과 Potion 사이의 공통 API와 Effect Tag 전달 경로 정리
- 기존 `UCPPotionImpactComponent`가 전달한 Effect Tag를 Quest NPC 납품 처리에 연동

투척 물리와 상태 관리의 최초 구현, 충돌 범위 검사 및 GAS Gameplay Event 처리,
동물 NPC의 포션 반응은 다른 팀원의 담당 범위입니다.

### 5. 게임 상태 저장 · 불러오기 연동

팀에서 만든 `UCPSaveSubsystem`에 각 시스템의 저장 데이터 수집과 복원 흐름을 연결했습니다.

- 월드 시간
- 인벤토리
- 채집물 상태
- 채집물·Tag·NPC 도감
- NPC 및 Quest 진행 상태
- 가마솥 재료와 조합 상태
- 현재 레벨과 플레이어 위치·방향

새 게임 시작 시 각 Subsystem의 런타임 상태를 초기화하고,
불러오기 시 저장된 시스템 상태와 플레이어 Transform을 적용하도록 구성했습니다.

### 6. 담당 기능 UI

- 채집물 도감의 상세 설명, 채집 횟수, Tag 표시 갱신
- NPC 도감의 등록, String Table 설명, Quest 진행 내용 표시
- 발견 여부를 반영한 Tag 조합 도감
- 가마솥의 현재 Effect Tag를 보여주는 상호작용 UI
- 재료 운반 UI와 포션 명명 표시
- 납품 결과 등급과 최종 보상을 표시하는 결과 UI

### 7. 패키징 안정화

- Ray Tracing 관련 기능을 비활성화하고 쿠킹 대상을 정리
- GPU 충돌 로그를 기준으로 Shadow 설정을 조정해 테스트 가능한 패키징 환경 확보
- 패키징 대상 Map과 시작 영상 설정
- Unreal Engine 5.8 Development 패키징 오류와 대상 설정 수정

최종 실행 확인용 `v0.6.4-beta` 패키징 빌드를 별도 링크로 제공합니다.

---

## 주요 기여 코드

| 기여 영역 | 코드 위치 | 담당 내용 |
| --- | --- | --- |
| 채집물 정적 데이터 | `Source/CreatePotion/Public/Data/CPForageableItemData.h` | 채집물 DataAsset 구조 추가 및 도감·Prop 데이터 연동 |
| 도감 런타임 상태 | `Source/CreatePotion/Public/GameInstance/Subsystem/CPCodexSubsystem.h` | 채집물·NPC·Tag 발견 상태와 저장 데이터 관리 |
| 도감 처리 | `Source/CreatePotion/Private/GameInstance/Subsystem/CPCodexSubsystem.cpp` | 등록, 채집 횟수, 단계 해금, 조회 및 복원 |
| Tag 정의 데이터 | `Source/CreatePotion/Public/Data/CPTagDefinitionTypes.h` | 표시명·조합·색상 DataTable 구조 |
| DataTable 설정 | `Source/CreatePotion/Public/Settings/CPDTSettings.h` | 공통 Tag Definition 참조 |
| 가마솥 연금술 | `Source/CreatePotion/Public/Lab/Component/Interact/CPCauldronComponent.h` | 재료 수집, Tag 조합, 포션 생성 |
| 가마솥 상태 유지 | `Source/CreatePotion/Public/GameInstance/Subsystem/CPCauldronSubsystem.h` | 레벨 이동 및 저장·불러오기 상태 유지 |
| 투척 Prop 상호작용 | `Source/CreatePotion/Public/Lab/Component/Interact/CPThrowableComponent.h` | 공용 상호작용 및 Pickup Event 연결 |
| Quest NPC 납품 연동 | `Source/CreatePotion/Private/Items/Potion/CPPotionImpactComponent.cpp` | 포션 Effect Tag를 Quest NPC 결과 처리에 전달 |
| 저장 데이터 연동 | `Source/CreatePotion/Private/GameInstance/Subsystem/CPSaveSubsystem.cpp` | 각 Subsystem 데이터 수집·복원 및 초기화 |
| Tag 조합 도감 | `Source/CreatePotion/Private/UI/Widgets/Common/Codex/CPTagCodexWidget.cpp` | 조합식 구성과 발견 상태 표시 |
| 가마솥 UI | `Source/CreatePotion/Private/UI/Widgets/Lab/CPCauldronWidget.cpp` | 현재 Effect Tag 실시간 표시 |
| 납품 결과 UI | `Source/CreatePotion/Private/UI/Widgets/Lab/CPLabResultWidget.cpp` | 결과 등급·보상 표시 및 지급 흐름 |

---

## Controls

| 동작 | 입력 |
| --- | --- |
| 이동 | WASD |
| 시점 조작 | Mouse |
| 상호작용 | F |
| 아이템 사용 / 투척 | Left Click |
| 조준 | Right Click |
| 인벤토리 | I |
| 퀘스트 | J |
| 도감 | B |

---

## Build / Play

### Source Project

이 저장소의 `CreatePotion.uproject`는 Unreal Engine 5.8 프로젝트입니다.
C++ 소스코드는 `Source/CreatePotion`에서 확인할 수 있습니다.

### Packaged Build

[**Create Potion v0.6.4-beta 다운로드**](https://drive.google.com/file/d/1rq6BZ-FbZ2OYr345ob4vFyU3ls3QBtkA/view?usp=sharing)

### Play Video

[**Create Potion 플레이 영상**](https://drive.google.com/file/d/1wYYHEF186lLVIjGBgIATkxz-wvopihos/view?usp=sharing)
