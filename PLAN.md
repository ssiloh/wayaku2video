# wayaku2video 계획서

wayaku JSON(단어장)과 유튜브 링크로 **FHD(1920×1080) 가사 영상**을 만드는 도구.
가사·발음·단어 설명이 나오는 방식은 [hanon-lyrics](https://github.com/ssiloh/hanon-lyrics)의 미리보기 화면과 **똑같이** 한다.

> 상태: 계획 단계 (코드 없음). 아래 "열린 질문"을 정한 뒤 M0부터 시작한다.

---

## 1. 목표와 범위

**목표**
- 곡 하나에 대해 `mp4` 하나를 자동 생성한다. 원곡 오디오 + hanon-lyrics 스타일의 가사/발음/단어 설명 화면.
- 한 줄 명령으로 재현 가능해야 한다 (같은 입력 → 같은 영상).

**범위 밖 (처음에는 하지 않음)**
- 가사 타이밍 자동 정렬(음성 인식/강제 정렬) — 4장의 "타이밍" 참고, 후속 단계.
- 영상 업로드 자동화, 웹 UI.
- 원곡 영상 화면을 배경으로 쓰는 기능 (오디오만 사용).

## 2. 가장 중요한 전제: wayaku JSON에는 시간이 없다

wayaku JSON은 가사·발음·단위·단어 설명만 갖고 있고 **"몇 초에 이 줄이 나오는지"는 없다.** 시간은 hanon-lyrics 앱의 타이밍 데이터(`SongTiming`)에 있다.

| 데이터 | 가진 것 | 시간 정보 |
|---|---|---|
| wayaku JSON (`제목/유튜브링크/원어/번역어/가사목록/재생순서`) | 가사, 가사발음, 단위, 단위발음, 단어설명 | 없음 |
| 타이밍 JSON (`SongTiming`: `lines[].startTime`, `units[].startTime`) | 위 내용 + 줄/단위별 시작 시각 | **있음** |

- 타이밍 JSON의 `lines`는 이미 `재생순서`(반복 포함)대로 펼쳐져 있고, 줄마다 `가사발음`과 `단어설명`을 들고 있다.
- 따라서 **영상 생성의 필수 입력은 타이밍 JSON**이고, wayaku JSON은 (a) 제목·링크·`원어`/`번역어` 읽기, (b) 타이밍 JSON에 설명이 빠진 줄 보충, (c) 검증에 쓴다.
- 레포 안의 일부 타이밍 파일(예: `yours-timing.json`)은 `startTime`이 전부 `null`이다. 즉 앱에서 타이밍을 아직 찍지 않은 곡은 영상을 만들 수 없다. 렌더 전에 검증하고 명확한 에러로 중단한다.

**타이밍 입력 방법(택1 또는 둘 다 지원)**
1. 파일 경로: `--timing path/to/song-timing.json`
2. hanon-lyrics 서버에서 가져오기: `--video-id <id>` → `GET /api/timing?videoId=<id>` (Vercel Blob에 저장된 `timing/{videoId}.json`)

## 3. 입력 / 출력

```
wayaku2video render \
  --wayaku  ./wayaku/yours_wayaku.json \
  --timing  ./wayaku/yours-timing.json \      # 또는 --video-id DGXPro9x0yo
  [--url    https://youtu.be/DGXPro9x0yo] \   # 생략 시 wayaku의 "유튜브링크"
  --out     ./out/yours.mp4
```

- 출력: `1920×1080`, 30fps, H.264(`yuv420p`) + AAC 192k, `.mp4`. 기본 CRF 18.
- 중간 산출물(오디오, 프레임 캐시)은 `.cache/{videoId}/`에 두고 재실행 시 재사용한다.

## 4. 파이프라인

```
 wayaku JSON ─┐
 타이밍 JSON ─┼─▶ 1.로드·검증 ─▶ 2.장면 계획 ─▶ 3.프레임 렌더 ─┐
 유튜브 링크 ─┴─▶ 1b.yt-dlp 오디오 ──────────────────────────────┴─▶ 4.ffmpeg 합성 ─▶ mp4
```

### 4.1 로드·검증
- 스키마 검증: 필수 키, `재생순서` 인덱스 범위, 줄 수(타이밍 `lines.length` = `재생순서.length`) 일치.
- 타이밍 검증: 모든 줄 `startTime`이 숫자이고 단조 증가해야 함. `null`이나 역전이 있으면 줄 번호와 함께 에러.
- `원어`/`번역어`가 없으면 `ko`/`ja`로 간주 (hanon-lyrics와 동일한 기본값).

### 4.2 yt-dlp 오디오
- `yt-dlp -f bestaudio -x --audio-format wav -o ".cache/%(id)s/audio.%(ext)s" <URL>`
- 영상 화면은 쓰지 않으므로 오디오만 받는다 (용량·시간 절약).
- 오디오 길이를 `ffprobe`로 읽어 마지막 줄의 종료 시각으로 사용.

### 4.3 장면 계획
- 줄 `i`의 표시 구간 = `[startTime_i, startTime_{i+1})`. 마지막 줄은 `[startTime_last, 오디오 끝)`, 단 최대 N초(기본 8초)로 제한.
- 첫 줄 이전 구간(인트로)은 제목 카드를 보여준다 (곡 제목만, 4.4의 "제목 카드").
- 장면은 "시간 → 활성 줄" 순수 함수로 만든다. 같은 시간이면 항상 같은 프레임이 나오도록 (렌더 재현성).

### 4.4 화면 구성 (hanon-lyrics 미리보기와 동일)
`src/components/PreviewScreen.tsx`의 배치를 1920×1080 캔버스로 옮긴다. 컨트롤 바·재생 버튼은 제외.

- 배경: 어두운 단색 (`zinc-950`), 전체 가운데 정렬.
- **가사 줄**: 가운데 큰 글씨(굵게, 줄 간격 넉넉히). 바로 아래에 **가사발음**을 작은 모노스페이스, 회색(`zinc-400`)으로.
- **단어 설명 패널**: 둥근 테두리 박스(`border zinc-800`, 배경 `zinc-900/60`), 위에 작은 대문자 라벨. 단어마다 한 줄, 3열 그리드:
  | 열 | 내용 | 스타일 |
  |---|---|---|
  | 1 | **단어**(앰버 `amber-300`) + 그 아래 **단위발음**(작은 모노, `zinc-500`) | 왼쪽 |
  | 2 | **설명** | `zinc-400` |
  | 3 | **번역어 표현**(`"일본어"` 필드 값) | `zinc-200` |
- 설명이 빈 문자열인 단어도 행은 유지한다 (앱과 동일).
- 줄이 바뀌면 즉시 교체 (앱과 동일). 페이드는 옵션으로만 두고 기본은 끈다.

앱 대비 달라지는 점은 두 가지뿐이다: 컨트롤 바 제거, 인트로 제목 카드 추가. 나머지 색·배치·열 구성은 앱을 기준으로 한다.

**언어 대응**
- 패널 라벨과 3열의 이름은 `번역어`에 따라 바뀐다 (`ja`면 「日本語」, `en`이면 "English" 등). 문구 표는 hanon-lyrics의 `src/i18n/translations.ts`에서 가져온다.
- 글꼴은 `원어`/`번역어`에 맞춰 Noto Sans KR / JP / 기본 sans를 번들로 포함하고, 발음은 모노스페이스.

**넘침 처리**
- 단어가 많은 줄(예: 10개 이상)은 패널이 화면을 넘을 수 있다. 우선순위: ① 글자 크기 단계적 축소(최소 크기 보장) ② 그래도 넘으면 패널을 2열로 분할. 넘침 줄은 렌더 로그에 경고로 남긴다.

### 4.5 프레임 렌더
렌더링 방식은 3개를 비교했고 **Remotion(React)을 권장**한다.

| 방식 | 장점 | 단점 |
|---|---|---|
| **A. Remotion (권장)** | hanon-lyrics가 React/Tailwind라 마크업·타입(`src/types/timing.ts`)을 거의 그대로 재사용 → "똑같은 화면" 보장이 쉬움. 프레임=시간의 순수 함수라 재현성 좋음. ffmpeg 합성까지 내장. | Node + Chromium 필요, 무거움 |
| B. Playwright로 HTML 프레임 캡처 | 같은 HTML/CSS 재사용 가능, 의존성 적음 | 프레임 시킹·합성 코드를 직접 짬 |
| C. Pillow/Skia 직접 그리기 | 가볍고 서버 의존 없음 | 레이아웃을 전부 다시 구현해야 해서 앱과 어긋나기 쉬움 |

- yt-dlp는 Python 도구지만 **CLI로만 호출**하므로 언어 선택에 영향 없음. 전체를 Node/TypeScript로 통일한다.

### 4.6 합성
- Remotion이 만든 무음 영상 + `.cache/{videoId}/audio.wav`를 ffmpeg로 mux (Remotion 내장 오디오 사용도 가능).
- 오디오와 영상의 시작 오프셋(`--offset <초>`)을 옵션으로 둔다. 앱에서 찍은 타이밍과 yt-dlp로 받은 오디오 사이에 앞부분 무음 차이가 있을 수 있기 때문.

## 5. 저장소 구조 (안)

```
wayaku2video/
├─ PLAN.md
├─ package.json
├─ src/
│  ├─ cli.ts              # render 명령 진입점
│  ├─ load.ts             # wayaku/타이밍 로드·검증
│  ├─ download.ts         # yt-dlp 호출, 캐시
│  ├─ plan.ts             # 시간 → 활성 줄 계산
│  ├─ types.ts            # hanon-lyrics의 types/timing.ts 복사(동기화 주의)
│  └─ video/
│     ├─ Root.tsx         # Remotion 컴포지션 (1920×1080, 30fps)
│     └─ LyricScreen.tsx  # PreviewScreen 배치를 옮긴 컴포넌트
├─ fonts/                 # Noto Sans KR/JP
├─ samples/               # 테스트용 wayaku + 타이밍 JSON
└─ out/                   # 결과물 (gitignore)
```

## 6. 단계별 계획

| 단계 | 내용 | 완료 기준 |
|---|---|---|
| **M0** | 환경 준비: Node, ffmpeg, yt-dlp 설치 확인, 샘플 곡 1개 선정 | 3개 도구 버전 출력, 타이밍이 다 찍힌 샘플 확보 |
| **M1** | 로드·검증 + yt-dlp 오디오 다운로드 + 캐시 | 샘플에서 `audio.wav`와 검증 리포트 생성 |
| **M2** | 정지 화면 1장: `LyricScreen`을 1920×1080 PNG로 렌더 | hanon-lyrics 미리보기와 나란히 비교해 배치 일치 |
| **M3** | 전체 곡 렌더: 타이밍대로 줄 교체 + 오디오 합성 | 30fps mp4, 줄 전환 시각이 타이밍과 ±1프레임 이내 |
| **M4** | 다국어 대응(`원어`/`번역어`), 넘침 처리, `--offset` | `ja`/`en` 번역어 샘플로 각각 렌더 성공 |
| **M5** | 여러 곡 일괄 렌더, 에러 메시지·README 정리 | `wayaku/` 폴더 전체를 한 명령으로 처리 |

## 7. 테스트 / 검증

- 단위 테스트: 장면 계획(`plan.ts`) — 경계 시각, 마지막 줄, 인트로, 반복 줄.
- 검증기 테스트: `null` 타이밍, 역전, 줄 수 불일치, `재생순서` 범위 초과 입력이 각각 올바른 에러를 내는지.
- 시각 확인: 샘플 곡의 대표 프레임 5장(첫 줄, 단어가 가장 많은 줄, 마지막 줄 등)을 PNG로 저장해 hanon-lyrics 화면과 비교.
- 싱크 확인: 줄 전환 시각 3곳을 사람이 직접 재생해 확인.

## 8. 위험 요소

| 위험 | 대응 |
|---|---|
| **타이밍 부재** — 타이밍을 안 찍은 곡은 영상 불가 | 렌더 전 검증, 에러 메시지에 "hanon-lyrics 관리자 화면에서 타이밍을 찍어주세요" 안내. 장기적으로 자동 정렬 도입 검토 |
| yt-dlp 깨짐(유튜브 변경) | yt-dlp 버전을 로그에 남기고, 실패 시 `yt-dlp -U` 안내. 오디오 파일 직접 지정 옵션(`--audio`) 제공 |
| **저작권** — 원곡 오디오를 쓴 영상 | 개인 소장·학습용으로 한정하는 것을 README에 명시. 공개 업로드는 별도 판단 (자동화 범위 밖) |
| 싱크 어긋남 | `--offset` 옵션, 오디오 길이 검증 |
| 무거운 렌더(Chromium+인코딩) | 이 개발 기기(Android Termux proot)에서는 느리거나 실패할 수 있음 → 렌더는 PC 또는 GitHub Actions에서 실행하는 것을 기본으로 가정 |
| hanon-lyrics 타입 변경 | `types.ts`를 복사본으로 두되 `원어/번역어` 같은 변경 시 같이 갱신. 필요하면 후속으로 공용 패키지화 |

## 9. 열린 질문 (진행 전에 정할 것)

1. **타이밍을 어디서 가져올까?** 파일 경로만 / 서버(`--video-id`)만 / 둘 다. (제안: 둘 다)
2. **렌더 환경**: PC에서 돌릴지, GitHub Actions에서 돌릴지. (제안: 로컬 PC 우선, Actions는 M5 이후)
3. **인트로 제목 카드**: 넣을지, 곡 시작부터 바로 첫 줄을 보여줄지.
4. **단위(어절) 단위 강조(노래방식 하이라이트)**: hanon-lyrics와 똑같이 하려면 불필요하지만, `units[].startTime`이 있으므로 후속 옵션으로 추가 가능. 지금은 범위 밖으로 둠.
5. 레포 공개 여부: 우선 비공개.

## 10. 참고
- hanon-lyrics: 데이터 형식 `src/types/timing.ts`, 화면 `src/components/PreviewScreen.tsx`, 문구 `src/i18n/translations.ts`, wayaku 형식 설명 `wayaku/readme.txt`.
- 타이밍 저장 위치: Vercel Blob `timing/{videoId}.json` (`GET /api/timing?videoId=`).
