# 인트로(훅) → Google Flow 이미지+영상 프롬프트 생성기

목적: 대본의 인트로(훅) 구간 + 기존 캐릭터 Ingredient + STYLE_TAIL을 받아
Google Flow(Ingredients to Video / Veo 3.1 Lite || Omni Flash)용 클립 프롬프트를 만든다.
본편 장면(default 48)과 완전히 별도. 인트로 전용, 짧다.

캐릭터 Ingredient(character description in [title]_flow_prompt.txt)·STYLE_TAIL을 본편과 공유해 톤을 통일.

---

## 절대 원칙 (충돌 시 최우선)

1. 인트로(훅)만 생성한다. 본편 스포일러(결말·정체·반전) 노출 금지 — script_guide.md의
   "no spoiler" 체크리스트를 그대로 적용한다.
2. 대사(큰따옴표 안 원문)는 한 글자도 수정·요약·의역하지 않는다. lip-sync에 그대로 쓴다.
3. 나레이션 문장 자체도 각색하지 않는다. 다만 **시각 프롬프트(행동·카메라·조명·배경 묘사)는
   나레이션에서 추론해 상세히 보강한다** — 이것은 각색이 아니라 서술의 시각화다.
4. 인물 신원(외모)은 이미 만든 캐릭터 Ingredient 이미지에 있다. 프롬프트에 앵커를 반복하지
   않는다 — `@name`만 쓴다. flow_prompt_generator.md STEP 2와 동일한 원칙.
5. STYLE_TAIL은 flow_prompt_generator.md에서 쓴 것과 동일한 값을 그대로 재사용한다
   (본편과 인트로가 다른 그림체로 보이면 안 된다). 웹툰 셀셰이딩처럼 특정 매체를
   임의로 고정하지 않는다 — 화풍은 본편과 항상 같다.
6. 모든 출력에 텍스트·자막·말풍선 금지.

---

## 문장 수 != 클립 수

본편과 달리 인트로는 문장이 6개 정도. 문장 하나를 그대로 클립 프롬프트로
쓰면 Flow가 배경·소품·조명·인물 동작 같은 디테일을 스스로 지어내며 본편과 다른 장면을
만든다 (실측 확인됨)

- 클립은 **문장이 아니라 감정 비트** 단위로 나눈다 (보통 3~5클립).
- 한 클립에 나레이션 문장 여러 개를 묶어도 된다. 대사(따옴표) 문장은 립싱크 정확도를
  위해 단독 클립으로 둔다.
- **디테일 보강 규칙 (핵심):** 인트로 문장이 묘사하는 순간과 같은 순간을 본편 챕터가
  이미 상세히 묘사하고 있다면 (예: 인트로의 "goblins twice his size"는 챕터 3의
  "blue-gray as a storm cloud, one blunt horn, iron club as long as a fence post"와 같은 순간), 그 감각적 묘사(외형·소품·배경·조명)를 가져와 프롬프트를 보강한다.
  단, 본편에서만 드러나는 결말·정체·다음 전개는 절대 가져오지 않는다 — 이미 인트로
  시점에 드러나 있는 디테일만 확장한다.

---

## Flow 제약 (그대로 따른다)

- 클립당 Ingredient 이미지 최대 3장. 인물 최대 2 + 로케이션/스타일 1을 권장.
- Omni Flash 클립은 보통 4~10초. 한 클립이 10초를 넘지 않게 설계한다.
- 여러 클립은 Flow의 SceneBuilder에서 이어 붙인다 — 각 클립 사이는 "Add Scene"
  (새 컷, 다른 구도), 한 클립을 그대로 늘릴 때만 "Extend"를 쓴다. 이 판단은
  사용자가 SceneBuilder에서 하므로, 여기서는 클립을 **컷 단위로 명확히 분리**해서
  출력한다.

---

## PHASE 1: 인트로 원문 추출

대본에서 인트로 블록만 뽑는다. 제목 줄과 구독 유도(CTA) 문장은 제외 — 이건 시각
클립이 아니라 음성 전환 멘트다.

```python
import re
RAW_INTRO = """<대본의 [INTRO] 블록 전체>"""
RAW_INTRO = re.sub(r'(?m)^\s*\[[^\]\n]*\]\s*$', '', RAW_INTRO)  # [INTRO] 마커 제거
CTA_KW = ["Liking helps us bring you more stories"]
lines = [l.strip() for l in re.split(r'(?<=[.!?"”’])\s+', re.sub(r'\s+',' ',RAW_INTRO).strip()) if l.strip()]
lines = [l for l in lines if not any(k in l for k in CTA_KW)]
assert len(lines) >= 3, "❌ 인트로 문장이 너무 적음 — 대본 확인"
print(f"✅ P1 (훅 문장 {len(lines)}개, CTA 제외)")
```

응답: `훅 원문 {n}문장 추출 완료`

---

## PHASE 2: 클립 분할 (감정 비트 단위)

```python
TOTAL_DURATION = 24  # 기본값, 사용자가 매 실행마다 지정 가능 (최대 30, 하드 캡)
assert TOTAL_DURATION <= 30, "❌ 30초 상한 초과"

# clips=[{"n":1,"lines":[...],"duration":8,"is_dialogue":False}, ...]
clips = [...]  # 감정 비트로 3~5개 그룹핑, 대사 문장은 단독 클립

assert 3 <= len(clips) <= 5, f"❌ 클립 수 {len(clips)} — 3~5 권장"
for c in clips:
    assert c["duration"] <= 10, f"❌ 클립{c['n']} {c['duration']}s — Omni Flash 상한 10s 초과"
assert sum(c["duration"] for c in clips) <= TOTAL_DURATION, "❌ 합계가 TOTAL_DURATION 초과"
for c in clips:
    if c["is_dialogue"]: assert len(c["lines"]) == 1, f"❌ 클립{c['n']} 대사 클립에 문장 2개+"
print(f"✅ P2 (클립 {len(clips)}개, 합계 {sum(c['duration'] for c in clips)}s / {TOTAL_DURATION}s)")
```

응답: `클립 분할 완료 ({len(clips)}개, 총 {sum}s)`

---

## PHASE 3: 캐릭터/로케이션 Ingredient 확인

```python
existing_characters = [...]  # <제목>_flow_prompts.txt STEP1에서 읽는다.
# 각 줄은 <캐릭터 외형>, <PORTRAIT_TEMPLATE> <STYLE_TAIL> 구조 — 인물 외형은
# PORTRAIT_TEMPLATE 앞부분만 사용, PORTRAIT_TEMPLATE 자체(포즈·배경 지시)는 버린다.
# 예: "Deokman","GoblinChief"
cast_in_intro = [...]        # 이번 인트로 클립들에 실제 등장하는 이름만

for name in cast_in_intro:
    assert name in existing_characters, f"❌ '{name}' 기존 Ingredient 없음 — 본편 캐릭터 시트 확인"

need_new_location_ingredient = True  # 여러 클립에서 같은 장소(산신당)가 반복되면 True
```

로케이션이 필요한 경우, STEP 1에 아래 형식으로 **구조만** 고정한 참조 이미지를 추가한다
(날씨·조명·인물·행동은 절대 넣지 않는다 — 그건 각 클립에서 상황에 맞게 바뀐다):

```
=== LOCATION (abandoned mountain shrine) ===
=== LOCATION UPLOAD ===
Location identity reference of a weathered Korean Joseon-era mountain shrine interior,
sagging tiled roof beams, wooden altar table, faded wall paintings of mountain spirits,
one door hanging by a single hinge, neutral even daylight with no weather effects,
completely empty with no characters, no fire, no rain, no props in motion,
this is a structural identity reference only and the lighting, weather and any characters must NOT carry over into any scene, <STYLE_TAIL>
```

응답: `캐릭터 재사용: {cast_in_intro} / 로케이션 신규: {yes/no}`

---

## PHASE 4: 클립 프롬프트 조립

각 클립은 Ingredients-to-Video 한 번 호출로 완성한다 (이미지→영상 2단계 아님 — Flow는
Ingredient 이미지 + 텍스트 프롬프트로 영상을 바로 만든다).

### 조립 공식
```
[원문] 그 클립이 다루는 대본 문장 원문 그대로 (수정 금지)

CLIP n (지속 {duration}s, {대사|나레이션})

[Ingredients] @name1 [, @name2] [, @LOCATION]  ← 최대 3장
[Flow 프롬프트]
<주어(@name or @LOCATION)> <행동> <카메라 언급 1개> <조명·배경·소품 상세>
<로컬컬러>. 25~70단어. no text no letters no words no modern objects, <STYLE_TAIL>
[actual sound]
- dialogue: DIALOGUE: "<원문 대사 그대로>" — English dialogue, precise lip-sync,
  accurate mouth movements matching every word, natural [성별] voice matching the character's age and tone.
- narration: MUTE — no dialogue, no voice, ambient sound only (wind, rain, fire crackle as appropriate to the scene).
```

### 체크리스트 (매 클립)

1. `@name`으로 인물을 지칭하되 외모 앵커(복장·머리·체형 등)는 절대 반복하지 않는다.
2. 카메라 움직임은 최대 1개만 지정한다 (static hold / slow push-in / slight pan 중
   하나). 여러 인물이거나 배경이 복잡한 컷일수록 정적인 쪽을 택해 캐릭터 드리프트를
   줄인다 — 단독 인물 + 단순 배경일 때만 좀 더 과감한 움직임(arc, slow zoom out)을
   허용한다.
3. 대사 클립과 나레이션 클립을 한 클립 안에서 섞지 않는다.
4. 새로 만든 LOCATION Ingredient를 쓰는 클립은 그 구조(문·지붕·제단·벽화)를 유지하되
   해당 클립의 날씨·조명·인물은 자유롭게 지정한다.
5. 본편 챕터에서 디테일을 가져올 경우, 그 문장이 묘사하는 물리적 사실(외형·소품·배경)만
   가져온다 — 감정적 해석이나 이후 전개를 암시하는 표현은 가져오지 않는다.

### 군중·안전 치환 (flow_prompt_generator.md와 동일하게 적용)

```python
import re
SAFE_SUBS={r'\btied\b':'low-knotted', r'\bbound\b':'wrapped', r'\bgripping\b':'holding',
  r'\bdragged\b':'led', r'\bweeping\b':'with tears on the face', r'\bclutching\b':'holding'}
def postfix(line):
    for p,r in SAFE_SUBS.items(): line=re.sub(p,r,line,flags=re.I)
    return line
```

---

## PHASE 5: 검증

```python
import re
kor=[l[:30] for l in prompt_lines if re.search(r'[가-힣]',l)]
assert not kor, f"❌ 한글 잔존 {kor[:3]}"
assert sum(c["duration"] for c in clips) <= 30, "❌ 30초 상한 초과"
for c in clips: assert c["duration"] <= 10, "❌ 클립 10초 초과"
for c in clips: assert len(c.get("ingredients", [])) <= 3, f"❌ 클립{c['n']} Ingredient 4장+"
for c in clips:
    assert c.get("audio_field") in ("dialogue","mute"), f"❌ 클립{c['n']} 오디오 필드 없음"
# 스포일러 자가 점검 (script_guide.md 기준) — 클립마다 사람이 최종 확인
print(f"✅ 검증 통과 ({len(clips)}클립, 총 {sum(c['duration'] for c in clips)}s)")
```

---

## 출력 (이 구조·순서 고정)

`<title>_intro_flow_prompts.txt` 한 파일로 저장 후 present_files.

```
[훅 원문]
1. <해당 프로젝트 인트로 문장 1, 원문 그대로>
2. <해당 프로젝트 인트로 문장 2, 원문 그대로>
n. ...

STEP 1 (신규 Ingredient만 — 기존 캐릭터는 <제목>_flow_prompts.txt STEP1 참조로 생략)
=== LOCATION (...) ===
=== LOCATION UPLOAD ===
...


===클립===
CLIP 1 ({duration}s, {dialogue|narration})
[원문] <이 클립이 다루는 문장 원문>
[Ingredients] @<name1> [, @<name2>] [, @LOCATION]
[Flow 프롬프트] <조립 공식대로>
[audio] <DIALOGUE "..."  또는  MUTE>

CLIP 2 ({duration}s, ...)
...
```

- 두 블록 완전 분리. STEP 1 → 훅 원문 → 클립 순서 고정.
- 영어 블록(Flow 프롬프트·오디오)에 한글 없음. 헤더(=== 행, [원문] 등)만 한국어 허용.
- 대본 원문은 절대 수정하지 않은 채 [원문]에 그대로 남긴다 — 각색은 [Flow 프롬프트]에서만.
