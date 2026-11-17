# Together World · Children's Grand Park Stamp Adventure Web App

[English](#english) · [한국어](#한국어)

---

<a id="english"></a>

## English

A mobile web app where visitors find 4 game stations hidden around Seoul Children's Grand Park, clear each game, and collect digital stamps by tapping an NFC tag.
It is the app part of the STDT × ACDT group project (G12, Together World). Nothing to install: it opens in a phone browser (Safari, Chrome).

- File: a single `index.html` (HTML, CSS and JavaScript all in one file)
- Languages: Korean / English (toggle at the top)
- Server: Firebase Authentication + Cloud Firestore (for member records)

---

### 1. Features

| Screen | What it does |
|---|---|
| Start screen | Welcome message and a choice: "Log in and enter" or "Look around as a guest". The log-in side offers log in and sign up |
| Map | Illustrated park map (zoom, pan), 58 facility pins, category chips (Animal Land, Nature Land, Fun Land, Amenities, Gates), facility search. Tapping a pin opens an info sheet (description, hours, closures, fees) |
| Hidden `?` pins | The 4 game stations. Tapping one shows the same location hint as the stamp screen. The pin changes color once cleared |
| Facility list | All facilities grouped by category. Tapping one jumps to its pin on the map |
| Stamps | A 4-cell leaflet (Dino Game, Rock-Paper-Scissors, Spot the Difference, Pose Game), progress bar, hint for the next place. Tapping a cell shows how to play, the clear condition and how to use NFC |
| Adventure complete | Collecting all 4 shows a celebration screen (confetti) and any new badge |
| Badge roadmap | One badge at the 1st, 3rd, 5th, 7th and 10th completed adventure (Sprout Explorer → Legendary Guide). Members only |
| Season stamps | Stamps from season-only NFC stickers placed in the park. Autumn (Oct–Nov), Winter (Dec–Feb), Christmas Hidden (Dec). Members only |
| Settings | Change nickname, change password (saved to the server). The ID cannot be changed |

#### Members and guests

| | Member | Guest |
|---|---|---|
| Collect today's stamps | Yes | Yes |
| Record kept (adventures completed, plays per game) | Yes | No |
| Badge roadmap, season stamps | Yes | No |
| Continue on another device | Yes | No |
| Name shown | Nickname chosen at sign-up | Name entered at the start (blank = "Explorer") |

Stamps reset **once a day** in Korea time (KST). On the same device and day they stay after a refresh.

---

### 2. Game stations and placement

The route is **enter at the Back Gate → zoo → quiet south-east path → sports area → leave at the Back Gate**. Locations follow the team document "Together World — 데이터 근거 정리" (data evidence summary).

| Order | id | Game | Location | Hardware | To clear |
|---|---|---|---|---|---|
| 1 | `dino` | Dino Game | Back Gate plaza, beside the Rainbow Fountain | Arduino Uno + 2.4" TFT (SPI, 240×320) + buttons | Jump over 10 obstacles |
| 2 | `rps` | Rock-Paper-Scissors | Zoo entrance, between Palgakdang and the Waterfowl Aviary | Arduino Uno + TFT + buttons | Win 2 times |
| 3 | `diff` | Spot the Difference | Path from the Marine Animal House to the Soccer Field, before the Guui Gate Cafeteria | Arduino Uno + TFT (touch) | Find 3 differences |
| 4 | `pose` | Pose Game | Between the Tennis Courts and the Seniors' Playground | Raspberry Pi 4 + iPad (camera input and display) | Succeed in 2 of 3 rounds |

Every station signals a clear with an LED and buzzer, then sends the clear to the web app through an NFC tag.

---

### 3. NFC tags

Put a URL in this format on the tag (or in the NFC signal the station sends):

```
https://<deployed site>/?g=<id>
```

Example: `https://example.github.io/together-world/?g=dino`

These forms are accepted too:

```
https://<site>/?stamp=dino
https://<site>/#stamp-dino
https://<site>/stamp?g=dino
```

| id | Meaning |
|---|---|
| `dino`, `rps`, `diff`, `pose` | Game stamps |
| `autumn`, `winter`, `xmas` | Season stamps (valid only in their months, members only) |

After stamping, the app rewrites the address to `#stamps`, so a refresh does not stamp twice.
If the visitor has not entered yet (logged in or chosen guest), the start screen comes first and the stamp is given right after.

**On the phone (also explained inside the app)**
1. When you clear a game, the station LED turns on.
2. Within 30 seconds, hold the top back of the phone to the NFC mark.
3. Tap the notification (link) that appears to get the stamp.

iPhone needs the screen on; Android needs NFC turned on.

---

### 4. Deployment

It is a single static file, so any static host works (GitHub Pages, Netlify, Firebase Hosting, etc.).

1. Put `index.html` at the root of the repository.
2. Turn on hosting and use the resulting address in the NFC tag URLs.
3. The address must be HTTPS (both the Firebase calls and the NFC links assume HTTPS).

> The Claude artifact preview blocks outside server calls, so log in and sign up fail there with "Couldn't reach the server". Test member features on the deployed address.

---

### 5. Firebase setup

Settings at the top of `index.html`:

```js
var FIREBASE = { apiKey: '...', projectId: 'grand-project-eb000' };
```

If both values are left empty, the app runs in **practice mode** and accounts are saved only in that device's browser.

#### In the Firebase console

1. **Authentication → Sign-in method → Email/Password**: enable
2. Create a **Firestore Database**
3. Publish these **Firestore rules** (each user can read and write only their own document)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

If a step is missing, the app shows the likely cause next to the error ("turn on Email/Password in Firebase Authentication", "check the apiKey", "publish the Firestore rules").

#### How IDs are handled

Firebase only signs up by email, so each ID is registered as a placeholder email, `<id>@together-world.example`.
That is also how users appear in the Firebase console. IDs are 3–20 letters, digits or underscores, case-insensitive; passwords need 6 or more characters.

---

### 6. Data

#### Server (Firestore)

One document, `users/{uid}`, holds the whole member record as a JSON string.

| Field | Content |
|---|---|
| `data` | The record object below, `JSON.stringify`-ed |
| `nick` | Nickname (stored separately so it is easy to read in the console) |

```json
{
  "nick": "Juwon",
  "clears": 3,
  "clearDates": ["2026-11-15", "2026-11-16", "2026-11-17"],
  "games": { "dino": { "n": 3, "last": "2026-11-17" } },
  "seasons": { "autumn": "2026-11-17" },
  "today": { "date": "2026-11-17", "done": { "dino": 1763362800000 }, "cleared": false }
}
```

If the record diverges across devices, logging in **merges** the server copy and the device copy and keeps whichever holds more.

#### Device (localStorage)

| Key | Content |
|---|---|
| `tw.lang` | Chosen language (`ko` / `en`) |
| `tw.session` | Current entry (member or guest, name, log-in tokens) |
| `tw.rec.<id>` | Cached member record |
| `tw.stamps` | A guest's stamps for today |
| `tw.accounts` | Practice-mode accounts (only when no server is set) |

Where localStorage is unavailable, the app still works by keeping data in memory.

---

### 7. Settings the team can change

All are in the "settings the team may change" block at the top of the `<script>` in `index.html`.

| Name | What it changes |
|---|---|
| `GAMES` | Station ids and `?` pin coordinates (x, y) on the map. `null` hides the pin |
| `FIREBASE` | Server connection |
| `SEASONS` | Season stamp ids, valid months, icons |
| `BADGES` | How many completed adventures earn each badge, and its icon |
| `STR` | Every on-screen text (Korean `ko`, English `en`). Hints use keys like `h_dino` |
| `RAW` | The 58 facilities (number, category, coordinates, names, descriptions, flags) |

The map uses the same 1078 × 660 coordinate space as the official park map. Facility flags: `p` park hours, `z` zoo hours, `c` closed, `$` paid.
Facility locations: Seoul Facilities Corporation, Seoul Children's Grand Park.

---

### 8. Testing

1. Add `?g=dino` to the deployed address to get the Dino Game stamp. This checks the flow with just a browser, no NFC.
2. Stamp `rps`, `diff` and `pose` too to see the adventure-complete screen and badge.
3. The menu item **"Clear today's stamps (testing)"** empties today's stamps.
4. In the browser developer console, `__tw.state()` shows the current state (view, language, stamps, session, record, server connection).

---

### 9. Known limitations

- **The URL is the only proof of a clear.** Anyone who knows `?g=dino` can get the stamp without playing. The 30-second limit must be enforced by the station hardware; the web app does not check time. This is fine for a presentation or demo, but real operation would need the station to add a changing code (such as a one-time token) to the URL.
- **No password recovery.** Accounts use placeholder emails, so no reset email can be received.
- **A testing menu is still in place.** Remove "Clear today's stamps" before real deployment.
- **The map is a simplified illustration** of the official park map, so real paths and distances may differ.
- Facility information (hours, Botanical Garden closure, etc.) is as of November 2026.

---

<a id="한국어"></a>

## 한국어


서울어린이대공원 곳곳에 숨겨 둔 게임 스테이션 4곳을 찾아 게임을 깨고, NFC 태그로 디지털 도장을 모으는 모바일 웹앱입니다.
STDT × ACDT 조별 프로젝트(G12, Together World)의 앱 파트이며, 설치 없이 휴대폰 브라우저(Safari, Chrome)에서 열어 씁니다.

- 파일: `index.html` 하나 (HTML, CSS, JavaScript가 모두 한 파일에 들어 있음)
- 언어: 한국어 / English (상단 버튼으로 전환)
- 서버: Firebase Authentication + Cloud Firestore (회원 기록 저장용)

---

### 1. 주요 기능

| 화면 | 내용 |
|---|---|
| 시작 화면 | 환영 문구, "로그인하고 입장하기 / 비회원으로 둘러보기" 선택. 로그인 쪽에서 로그인과 회원가입 제공 |
| 지도 | 공원 일러스트 지도(확대·축소·드래그), 시설 핀 58곳, 카테고리 칩(동물나라·자연나라·재미나라·편의시설·출입문), 시설 검색. 핀을 누르면 정보 시트(설명·운영시간·휴관·유료 여부) |
| 숨은 장소 `?` 핀 | 게임 스테이션 4곳. 누르면 스탬프 화면과 같은 장소 힌트가 뜸. 클리어하면 핀 색이 바뀜 |
| 시설 목록 | 카테고리별로 묶은 전체 시설 목록. 누르면 지도에서 해당 핀으로 이동 |
| 스탬프 | 리플렛 모양 4칸(공룡게임·가위바위보·틀린그림 찾기·포즈게임), 진행 바, 다음 장소 힌트, 칸을 누르면 게임 방법·클리어 조건·NFC 사용법 |
| 탐험 완료 | 4개를 모두 모으면 축하 화면(꽃가루 애니메이션)과 새 배지 표시 |
| 배지 로드맵 | 탐험 완료 1·3·5·7·10회째에 배지 1개씩 (새싹 탐험가 → 전설의 길잡이). 회원만 |
| 시즌 스탬프 | 공원에 붙인 시즌 전용 NFC 스티커로 받는 도장. 가을(10~11월), 겨울(12~2월), 크리스마스 히든(12월). 회원만 |
| 설정 | 닉네임 변경, 비밀번호 변경(서버 반영). 아이디는 변경 불가 |

#### 회원과 비회원

| | 회원 | 비회원 |
|---|---|---|
| 오늘 도장 찍기 | O | O |
| 기록 누적(탐험 완료 횟수, 게임별 횟수) | O | X |
| 배지 로드맵, 시즌 스탬프 | O | X |
| 다른 기기에서 이어 하기 | O | X |
| 부르는 이름 | 가입 때 정한 닉네임 | 입장할 때 입력한 이름 (비우면 "탐험가님") |

도장은 한국 시간(KST) 기준 **하루 단위**로 초기화됩니다. 같은 날 같은 기기에서는 새로고침해도 유지됩니다.

---

### 2. 게임 스테이션과 배치

동선은 **후문 입장 → 동물원 → 남동쪽 조용한 길 → 체육시설 쪽 → 후문 퇴장** 순서입니다. 위치 근거는 팀 문서 "Together World — 데이터 근거 정리"를 따릅니다.

| 순서 | id | 게임 | 위치 | 하드웨어 | 클리어 조건 |
|---|---|---|---|---|---|
| 1 | `dino` | 공룡게임 | 후문 광장, 무지개분수 옆 | Arduino Uno + 2.4" TFT(SPI, 240×320) + 버튼 | 장애물 10개 넘기 |
| 2 | `rps` | 가위바위보 | 동물원 입구, 팔각당과 물새장 사이 | Arduino Uno + TFT + 버튼 | 2번 이기기 |
| 3 | `diff` | 틀린그림 찾기 | 바다동물원 → 축구장 가는 길, 구의문 카페테리아 앞 | Arduino Uno + TFT(터치) | 틀린 곳 3군데 찾기 |
| 4 | `pose` | 포즈게임 | 테니스장과 어르신놀이터 사이 | Raspberry Pi 4 + 아이패드(카메라 입력·화면 출력) | 3라운드 중 2번 성공 |

모든 스테이션은 클리어하면 LED·부저로 알려 주고, 이어서 NFC 태그로 웹앱에 클리어 신호를 보냅니다.

---

### 3. NFC 태그 연동

태그(또는 스테이션이 내보내는 NFC 신호)에는 아래 형식의 URL을 넣습니다.

```
https://<배포한 사이트 주소>/?g=<id>
```

예: `https://example.github.io/together-world/?g=dino`

다음 형식도 같은 의미로 받아들입니다.

```
https://<사이트>/?stamp=dino
https://<사이트>/#stamp-dino
https://<사이트>/stamp?g=dino
```

| id | 의미 |
|---|---|
| `dino`, `rps`, `diff`, `pose` | 게임 도장 |
| `autumn`, `winter`, `xmas` | 시즌 도장 (해당 월에만 유효, 회원만) |

URL로 들어오면 앱이 도장을 찍은 뒤 주소창을 `#stamps`로 바꿔, 새로고침해도 도장이 두 번 찍히지 않게 합니다.
아직 입장(로그인 또는 비회원 선택) 전이라면 시작 화면을 먼저 보여 주고, 입장이 끝나면 그 도장을 이어서 찍습니다.

**휴대폰 쪽 사용법(앱 안에도 안내 있음)**
1. 게임을 클리어하면 스테이션 LED가 켜집니다.
2. 30초 안에 휴대폰 뒷면 윗부분을 NFC 표시에 댑니다.
3. 화면에 뜨는 알림(링크)을 누르면 도장이 찍힙니다.

iPhone은 화면을 켠 상태, Android는 NFC를 켠 상태여야 합니다.

---

### 4. 배포

정적 파일 하나라서 어떤 정적 호스팅에도 올릴 수 있습니다. (GitHub Pages, Netlify, Firebase Hosting 등)

1. `index.html`을 저장소 루트에 둡니다.
2. 호스팅을 켜고 나온 주소를 NFC 태그 URL에 씁니다.
3. HTTPS 주소여야 합니다. (Firebase 요청과 NFC 링크 모두 HTTPS 기준)

> Claude 아티팩트 미리보기 화면에서는 외부 서버 요청이 막혀 있어 로그인·회원가입이 "서버에 닿지 못했어요"로 실패합니다. 회원 기능은 반드시 배포한 주소에서 확인하세요.

---

### 5. Firebase 설정

`index.html` 상단의 설정값:

```js
var FIREBASE = { apiKey: '...', projectId: 'grand-project-eb000' };
```

두 값을 모두 비우면 **연습 모드**로 돌아갑니다. 이때 계정은 그 기기의 브라우저에만 저장됩니다.

#### 콘솔에서 해야 할 일

1. **Authentication → 로그인 방법 → 이메일/비밀번호** 사용 설정
2. **Firestore Database** 생성
3. **Firestore 규칙**을 아래처럼 게시 (본인 문서만 읽고 쓰기)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

설정이 빠지면 앱이 오류 문구 옆에 원인을 같이 보여 줍니다. ("이메일/비밀번호를 켜 주세요", "apiKey를 다시 확인해 주세요", "Firestore 규칙을 게시해 주세요")

#### 아이디 처리 방식

Firebase는 이메일로만 가입을 받기 때문에, 아이디를 `<아이디>@together-world.example` 형태의 가짜 이메일로 바꿔서 등록합니다.
그래서 Firebase 콘솔의 사용자 목록에도 이 형태로 보입니다. 아이디는 영문·숫자·밑줄 3~20자, 대소문자 구분 없음, 비밀번호는 6자 이상입니다.

---

### 6. 데이터 구조

#### 서버 (Firestore)

`users/{uid}` 문서 하나에 회원 기록 전체를 JSON 문자열로 저장합니다.

| 필드 | 내용 |
|---|---|
| `data` | 아래 기록 객체를 `JSON.stringify` 한 문자열 |
| `nick` | 닉네임 (콘솔에서 보기 쉽게 따로 저장) |

```json
{
  "nick": "주원",
  "clears": 3,
  "clearDates": ["2026-11-15", "2026-11-16", "2026-11-17"],
  "games": { "dino": { "n": 3, "last": "2026-11-17" } },
  "seasons": { "autumn": "2026-11-17" },
  "today": { "date": "2026-11-17", "done": { "dino": 1763362800000 }, "cleared": false }
}
```

여러 기기에서 쓰다가 기록이 갈라지면, 로그인할 때 서버 기록과 기기 기록을 **합쳐서** 더 많이 모은 쪽을 남깁니다.

#### 기기 (localStorage)

| 키 | 내용 |
|---|---|
| `tw.lang` | 선택한 언어 (`ko` / `en`) |
| `tw.session` | 현재 입장 정보 (회원/비회원, 이름, 로그인 토큰) |
| `tw.rec.<아이디>` | 회원 기록 캐시 |
| `tw.stamps` | 비회원의 오늘 도장 |
| `tw.accounts` | 연습 모드 계정 (서버 미연결 때만) |

localStorage를 쓸 수 없는 환경에서도 메모리에 임시 저장하며 동작합니다.

---

### 7. 팀이 바꿀 수 있는 설정값

모두 `index.html`의 `<script>` 맨 위 "settings the team may change" 구역에 있습니다.

| 이름 | 바꾸는 것 |
|---|---|
| `GAMES` | 스테이션 id와 지도 위 `?` 핀 좌표(x, y). 좌표를 `null`로 두면 핀을 숨김 |
| `FIREBASE` | 서버 연결 정보 |
| `SEASONS` | 시즌 도장 id, 유효 월, 아이콘 |
| `BADGES` | 배지가 나오는 탐험 완료 횟수와 아이콘 |
| `STR` | 화면에 나오는 모든 문구(한국어 `ko`, 영어 `en`). 힌트는 `h_dino` 같은 키 |
| `RAW` | 시설 목록 58곳(번호, 분류, 좌표, 이름, 설명, 운영 표시) |

지도 좌표계는 공식 공원 안내도와 같은 1078 × 660입니다. 시설 운영 표시는 `p` 공원 운영시간, `z` 동물원 운영시간, `c` 휴관, `$` 유료입니다.
시설 위치 출처: 서울시설공단 서울어린이대공원.

---

### 8. 테스트 방법

1. 배포 주소 끝에 `?g=dino`를 붙여 열면 공룡게임 도장이 찍힙니다. NFC 없이 브라우저만으로 흐름을 확인할 수 있습니다.
2. `rps`, `diff`, `pose`까지 찍으면 탐험 완료 화면과 배지가 나옵니다.
3. 메뉴의 **"오늘 도장 지우기 (테스트용)"** 으로 오늘 도장을 비울 수 있습니다.
4. 브라우저 개발자 도구 콘솔에서 `__tw.state()`를 입력하면 현재 상태(보기, 언어, 도장, 세션, 기록, 서버 연결 여부)를 볼 수 있습니다.

---

### 9. 알려진 한계

- **클리어 증명이 URL뿐입니다.** `?g=dino` 주소만 알면 게임을 하지 않고도 도장을 받을 수 있습니다. "30초 안에 태그" 제한도 스테이션 하드웨어 쪽에서 지켜야 하며, 웹앱은 시간을 확인하지 않습니다. 발표나 시연에서는 문제가 없지만 실제 운영이라면 스테이션이 매번 바뀌는 코드(예: 일회용 토큰)를 URL에 붙이는 방식이 필요합니다.
- **비밀번호 찾기가 없습니다.** 가짜 이메일로 가입하기 때문에 재설정 메일을 받을 수 없습니다.
- **테스트용 메뉴가 남아 있습니다.** 실제 배포 전에 "오늘 도장 지우기" 메뉴를 빼야 합니다.
- **지도는 공식 안내도를 단순화한 일러스트**라 실제 길과 거리는 다를 수 있습니다.
- 시설 정보(운영시간, 식물원 휴관 등)는 2026년 11월 기준입니다.
