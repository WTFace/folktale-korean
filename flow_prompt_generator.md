# 대본 → Google Flow 이미지 프롬프트 생성기

대본 + 화풍 + 장면 수를 받아 Google Flow Image(Nano Banana)용 프롬프트를 만든다.
화풍은 텍스트(STYLE_TAIL)로 고정한다(레퍼런스 이미지 색 번짐 방지). 외부 이미지 0개 투입.

## 운영 규칙
1. PHASE를 순서대로 처리한다.
2. 포즈는 아래 POSE_POOL에서 고른다. 자유 창작 대신 풀에서 강도·사건에 맞는 항목 선택.
3. 사용자 출력은 템플릿만.

## 핵심 원칙
- 정면 손모음 포즈가 장면에 복붙되면 안 됨: 인물은 POSE_POOL에서 고른 신체 동작.
- 회색 스튜디오 배경 금지: 장면 배경은 실제 조선 로케이션.

## 시작 메시지
```
🎬 Google Flow 이미지 프롬프트 생성기
준비물: 1) 대본  2) 화풍(STYLE_TAIL)  3) 장면 수(기본 48)
```

## 처리 흐름
```
PHASE 1   화풍 추출 → STYLE_TAIL
PHASE 2   장면 분할
PHASE 3   캐릭터 추출 + 조선 복식 앵커
PHASE 3.5 샷 배정 + 포즈 배정
PHASE 4   STEP 1(배경·포즈 분리) + STEP 2 조립
PHASE 5   최소 검증 → TXT
```

═══════════════════════════════════════════════
## PHASE 1: 화풍 추출
═══════════════════════════════════════════════
STYLE/RENDERING/CHARACTER_RENDERING/COMPOSITION/QUALITY 5개만 영문 한 줄(10~30 words). LIGHTING/COLOR/SETTING/MOOD 버림. 색 단어 금지.

```python
import re
STYLE_TAIL = STYLE_TAIL_RAW.strip().strip(',').strip()
words=len(STYLE_TAIL.replace(',','').split()); phrases=[p.strip() for p in STYLE_TAIL.split(',') if p.strip()]
assert 10<=words<=30, f"❌ 단어 {words}"
LEAK=[r'\bcandlelight\b',r'\bmoonlight\b',r'\bsunlight\b',r'\bnavy\b',r'\bamber\b',r'\bcrimson\b',r'\bgold\b',r'\bhanok\b',r'\bhanbok\b',r'\bnight\b']
assert not [p for p in LEAK if re.search(p,STYLE_TAIL,re.I)], "❌ 색·분위기 누수"
print(f"✅ P1 ({words}단어)")
```
응답: `화풍: <STYLE_TAIL>`

═══════════════════════════════════════════════
## PHASE 2: 장면 분할
═══════════════════════════════════════════════
본문 추출(인트로/아웃트로 제거) → play length(about 154 wpm) 균등 분할.
첫 장면은 "Long ago" 본문 시작부터. scenes=[{"n","start_idx","end_idx"}]

모든 장면을 균등하게 처리한다.

```python
import re
RAW_SCRIPT = """<대본 전체>"""
RAW_SCRIPT = re.sub(r'(?m)^\s*\[[^\]\n]*\]\s*$', '', RAW_SCRIPT)  # [CHAPTER n], [INTRO] 등 마커 라인 제거
N_SCENES = <default 36>
sentences = re.split(r'(?<=[.!?。"\u201d\u2019])\s+', re.sub(r'\s+',' ',RAW_SCRIPT).strip())
sentences = [s.strip() for s in sentences if s.strip()]
INTRO_KW=["Now, let's begin"]
OUTRO_KW=["Thank you for watching"]
intro_end=0
for i in range(min(10,len(sentences))):
    if any(k in sentences[i] for k in INTRO_KW): intro_end=i+1
assert intro_end>0, "❌ 인트로 키워드 미검출 — intro는 제목 포함 10문장 이내, 마지막 문장에 INTRO_KW 필수"
outro_start=len(sentences)
for i in range(len(sentences)-1, max(len(sentences)-8,0)-1, -1):
    if any(k in sentences[i] for k in OUTRO_KW): outro_start=i
body=sentences[intro_end:outro_start]
assert len(body)>=N_SCENES, f"❌ 본문 {len(body)}문장 부족"
# scenes: 단어수 균등 분할
assert len(scenes)==N_SCENES
for i in range(1,len(scenes)):
    assert scenes[i]["start_idx"]==scenes[i-1]["end_idx"]+1, f"❌ 경계 불연속 장면{i+1}"
print(f"✅ P2 (본문 {len(body)}문장)")
```
응답: `장면 분할 완료`

═══════════════════════════════════════════════
## PHASE 3: 캐릭터 추출 + 조선 복식 앵커  (기본 5명, 단계 변형 포함 최대 6슬롯)
═══════════════════════════════════════════════
- 등장 빈도 상위 최대 5명만 잠근다. 단계 변형이 필요한 이야기는 총 6슬롯까지 허용.
- 같은 인물 다른 단계(거지→의녀, 혹 제거 전→후 등 외모 변화)는 다른 name으로 별도 슬롯.
  단, 단계 변형은 등장 빈도 상위 2명에게만 허용 — 나머지는 단일 앵커로 커버.
- 비인간 캐릭터(도깨비·귀신·구미호 등)는 is_human=False: 조선 복식 마커 대신 creature 마커로 검증.
  복식을 입힐 수 있으면 입힌다(예: ragged dark jeogori vest) — 화면 통일감에 유리.
- 앵커 3요소 = 인간은 반드시 Korean·Joseon + 조선 복식 명사. 국적 불명 표현 금지.
- 앵커에는 신원(복식·머리·얼굴·체형)만. 자세·손동작·표정·시선을 넣지 않는다.

```python
import re
ingredients=[{"name":"...","role":"...","scenes":"...","rank":1,"is_human":True,
  "anchor_outfit":"...","anchor_hair":"...","anchor_feature":"..."}]
for ing in ingredients:
    for k in ["role","anchor_outfit","anchor_hair","anchor_feature"]: assert ing.get(k), f"❌ {ing['name']} {k}"
assert 1<=len(ingredients)<=6, f"❌ 인원 {len(ingredients)}"
KMARK=["jeogori","dopo","danryeong","durumagi","gat","garima","samo","baji","chima","hanbok","topknot"]
CMARK=["horn","goblin","oni","ghost","spirit","gumiho","fox","tiger","serpent","dragon", "broom"]
for ing in ingredients:
    blob=(ing["anchor_outfit"]+" "+ing["anchor_hair"]).lower()
    if ing.get("is_human",True):
        assert any(m in blob for m in KMARK), f"❌ '{ing['name']}' 조선 복식 마커 없음"
    else:
        blob+=" "+ing["anchor_feature"].lower()
        assert any(m in blob for m in CMARK), f"❌ '{ing['name']}' creature 마커 없음"
print(f"✅ P3 ({len(ingredients)}명)")
```
응답: `캐릭터: ...`

═══════════════════════════════════════════════
## PHASE 3.5: 샷 배정 + 포즈 배정 (축소 POSE_POOL)
═══════════════════════════════════════════════
### 샷 배정
```python
# shots=[{"shot","pose", "cast"}] cast: "main"(앵커 캐릭터) / "extras"(무명 인물만) / "none"(무인물)
```
아래 샷 목록에서 장면마다 하나씩 고른다. 바로 앞 장면과 같은 샷만 아니면 된다.
바로 옆 장면과 다른 샷이면 충분하다. 무인물/무명(non main cast) 샷은 전체의 20% 이하, 연속 금지
```
medium_shot, medium_close_up, long_shot, two_shot, over_the_shoulder,
wide_landscape, low_angle, side_profile, front_view, close_up_portrait
```

### 포즈 배정
매 shot 아래에서 하나 고른다. 바로 앞 장면과 같은 포즈만 아니면 된다.
```python
POSE_POOL = [
  "caught mid-stride, weight thrown onto one leg",
  "turning the head and shoulders to glance back",
  "reaching out one hand toward another figure",
  "leaning forward over a surface, weight on the hands",
  "standing with the body turned away, gazing into the distance",
  "crouching to examine something on the ground",
  # 추가 — 낮은 자세 (앉음/무릎)
  "kneeling on the ground with the back straight, head bowed",
  "sitting on a low step with elbows resting on the knees",
  # 추가 — 동적/긴급
  "breaking into a run, hanbok sleeves trailing behind",
  "stumbling backward with one arm raised protectively",
  "pushing open a heavy wooden door with one shoulder",
  # 추가 — 운반/노동
  "carrying a large bundle balanced against one hip",
  "drawing water, bent at the waist over a stone well",
  # 추가 — 감정/정적
  "pressing one hand flat against the chest, head tilted down",
  "shielding the eyes with one hand while looking upward",
  "raising a small object toward the light with both hands",
]

assert len(shots)==N_SCENES
for s in shots:
    if s["cast"]!="none": assert s["pose"] in POSE_POOL
    else: s["pose"]="none"
for i in range(1,len(shots)):
    assert shots[i]["shot"]!=shots[i-1]["shot"], f"❌ 장면{i+1} 샷 연속"
    if shots[i]["pose"]!="none" and shots[i-1]["pose"]!="none":
        assert shots[i]["pose"]!=shots[i-1]["pose"], f"❌ 장면{i+1} 포즈 연속"
print(f"✅ P3.5")
```
응답: `샷·포즈 배정 완료`

═══════════════════════════════════════════════
## PHASE 4: STEP 1(배경·포즈 분리) + STEP 2 조립
═══════════════════════════════════════════════
### STEP 1 — UPLOAD (신원 전용 / 배경분리 / 포즈분리)
캐릭터당 한 장. Flow ingredient 슬롯용 신원 식별 전용 레퍼런스.
```
PORTRAIT_TEMPLATE = neutral A-pose standing straight with both arms relaxed at the sides, plain flat neutral gray background, subject fully isolated, this is an identity reference only and the pose, hand position and background must NOT carry over into any scene, no hands clasped, no props no other figures,

=== <n> (<name>, <info> ) ===
=== <n> Flow character prompt ===
<a Korean Joseon-era character, anchor_outfit + anchor_hair + anchor_feature>, <PORTRAIT_TEMPLATE> <STYLE_TAIL>
```
★ STEP 1·2 어디에도 한국어 금지(헤더 === 행만 예외).

### STEP 2 — 장면 프롬프트 (조립 공식)
```
shot with main character: N. @name — [샷 영문] of [주어] [POSE_POOL 포즈]. [행동·배경·로컬컬러·인원신호] 15~65단어. <SAFE_TAG>, <STYLE_TAIL>
shot without main character: N. [샷] of <조선 로케이션·사물 주어>. [배경·소품·분위기 묘사] 15~65단어. no people no figures, <SAFE_TAG>, <STYLE_TAIL>
```
앵커 3요소는 STEP 1 UPLOAD(Flow character info 탭)에만 넣는다. 장면 프롬프트에서 앵커 반복 금지 — 신원은 @태그가 전달한다.
단, 플롯상 그 장면에서 보여야 하는 신체 특징(혹의 유무·개수 등)은 [행동] 묘사에 자연스럽게 포함.
**조립 체크리스트 (매 줄):**

공통:
1. 샷 뒤에 주어("of the woman" / "of the courtyard") — 동사로 시작 금지
2. 장면 배경 명사(courtyard/field/stone wall…)
3. `<SAFE_TAG>, <STYLE_TAIL>` 로 끝
4. 한글 없음

cast="main":
5. `@name —` 으로 시작, em dash `—` 뒤에 [샷 영문]. 앵커 3요소 반복 금지 (STEP 1 전용)
6. 주어 뒤에 shots[n]["pose"] 그대로

cast="extras":
7. @태그 없음. 익명 명사는 `a stout Joseon man in a brown durumagi` 식으로, 주어 뒤에 shots[n]["pose"] 그대로

cast="none":
8. @태그·포즈 없음. `N. [샷] of <로케이션·사물>` 로 시작, `no people no figures` 반드시 포함

- @태그 최대 2명. 손 모은 자세(hands clasped/folded)는 어느 컷에도 안 씀.

### 군중·안전 치환
```python
import re
SAFE_SUBS={r'\btied\b':'low-knotted', r'\bbound\b':'wrapped', r'\bgripping\b':'holding',
  r'\bdragged\b':'led', r'\bweeping\b':'with tears on the face', r'\bclutching\b':'holding'}
EXTRA_SUBS=[
 (r'\ba crowd of villagers\b','a crowd of Korean Joseon villagers in hanbok'),
 (r'\bonlookers\b','Korean Joseon onlookers in hanbok'),
 (r'\bother men\b','other Korean Joseon men in hanbok'),
]
def postfix(line):
    for p,r in SAFE_SUBS.items(): line=re.sub(p,r,line,flags=re.I)
    for p,r in EXTRA_SUBS: line=re.sub(p,r,line)
    return line
# prompt_lines: STEP 2 장면 프롬프트만 (STEP 1 UPLOAD 라인 제외)
prompt_lines=[postfix(l) for l in prompt_lines]
```

### SAFE_TAG (그대로 사용)
```
natural mid-action pose, no text no letters no words no modern objects
```

═══════════════════════════════════════════════
## PHASE 5: 검증
═══════════════════════════════════════════════
치명적 오류(앵커 누락·한글 잔존·회색 배경·SAFE_TAG 누락)를 막는다.

```python
import re
# 조선 복식 앵커 확인
for s,l in zip(shots,prompt_lines):
    if s["cast"]=="main": assert '@' in l, "❌ 앵커 태그 없음"
    elif s["cast"]=="none": assert "no people" in l, "❌ 무인물 컷 no people 누락"
# 한글 잔존 (영어 블록)
kor=[l[:30] for l in prompt_lines if re.search(r'[가-힣]',l)]
assert not kor, f"❌ 한글 잔존 {kor[:3]}"
# 회색 스튜디오 배경 침입
STUDIO_BAN=[r'(gray|grey|neutral)\s+background\b',r'\bstudio backdrop\b',r'\bblank background\b']
for n,l in enumerate(prompt_lines,1):
    if [p for p in STUDIO_BAN if re.search(p,l.lower())]:
        assert False, f"❌ 장면{n} 회색 스튜디오 배경"
# SAFE_TAG·STYLE_TAIL 존재
for n,l in enumerate(prompt_lines,1):
    assert "no text no letters" in l, f"❌ 장면{n} SAFE_TAG 누락"
print(f"✅ 최소 검증 통과 ({len(prompt_lines)}장면)")
```

## output: 2 files (이 구조·순서 고정)
<제목>_scene_script.txt
```
[대본 1~n]
1. Long ago, ...
n. ...

[first sentence map]
1: first sentence of 대본1
2: first sentence of 대본2
n: first sentence of 대본n
```
<제목>_flow_prompts.txt 한 파일에 아래 순서로 저장 후 present_files.
```
STEP 1
PORTRAIT_TEMPLATE = neutral A-pose standing straight with both arms relaxed at the sides, plain flat neutral gray background, subject fully isolated, this is an identity reference only and the pose, hand position and background must NOT carry over into any scene, no hands clasped, no props no other figures,

=== <n> (<name>, <info>) ===
=== <n> Flow character prompt ===
<a Korean Joseon-era character, anchor_outfit + anchor_hair + anchor_feature>, <PORTRAIT_TEMPLATE 전체 그대로 전개>, <STYLE_TAIL>
(캐릭터별 반복 — PORTRAIT_TEMPLATE는 매 줄 문자 그대로 펼쳐 쓴다. "PORTRAIT_TEMPLATE"라는 단어를
 그대로 남기지 않는다. 위 한 줄의 선언은 다른 문서가 이 값을 잘라내는 기준일 뿐이다.)

===scene prompts===
[1~n]
1. @name — Shot of <주어> <POSE_POOL 포즈> ... , <SAFE_TAG>, <STYLE_TAIL>
n. ...
```
- 대본·영어 모두 줄 시작은 숫자+점+공백만.
- 대본 1번은 본문 첫 문장 "Long ago" 부터.
- 두 블록 완전 분리. STEP1 → scene prompts 순서 고정. 영어블록 한글 금지.
- 장면 배경은 실제 조선 로케이션 or nature(no modern things).
- 이 파일 하나가 다른 프롬프트 생성기(thmbnail_prompt.md, intro_prompt.md)의 캐릭터·화풍
  소스가 된다. 캐릭터 외형 = STEP1 각 줄에서 PORTRAIT_TEMPLATE 앞부분만(뒷부분은 버림),
  STYLE_TAIL = PORTRAIT_TEMPLATE 뒷부분(STEP1·scene prompts 아무 줄에서나 동일).
  (각 문서의 "레퍼런스 반영" 절 참고, 추출 규칙은 각 문서에 명시).
