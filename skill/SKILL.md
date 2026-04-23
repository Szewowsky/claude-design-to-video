---
name: claude-design-to-video
description: "Użyj tego skilla gdy user ma gotową animację (folder/bundle/eksport HTML/React, często w ~/Downloads, np. 'Animacja YouTube') i chce z niej plik wideo/MP4/film. Intencja: animacja → MP4. Typowe sformułowania: 'wyrenderuj animację', 'wyrenderuj mi to', 'zrób film/wideo z animacji', 'zamień animację na MP4', 'potrzebuję MP4 na YT/social', '4K/1080p/60fps render animacji', 'mam folder/bundle → chcę plik wideo'. Triggeruj też gdy user wspomina Claude Design, useTimeline, Stage, animations.jsx, standalone HTML export, lub po prostu 'folder w Downloads z animacją' — niezależnie czy pada nazwa 'Claude Design'. Krótkie frazy typu 'wyrenderuj mi ta animacje' + kontekst gotowego folderu/bundla = trigger. NIE używaj do: transcodowania istniejących plików MP4/wideo, montażu nagrań z kamery/telefonu/GoPro, ekstrakcji audio, projektów Remotion, zwykłego HTML bez animacji JS, generowania grafik/thumbnaili/banerów, podsumowań filmów YouTube. Reguła: animowany artefakt HTML/React + intencja 'chcę plik wideo' = ten skill."
argument-hint: "[folder-bundle | --flags]"
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
---

# /claude-design-to-video

Claude Design eksportuje animacje jako standalone HTML (React + Babel + JSX scenes). Żeby zamienić to w MP4 bez drop frames i glitchy — używamy **timecut** (Puppeteer + wirtualny czas + ffmpeg). Nagrywanie real-time (Playwright/OBS) daje szajs — timecut "zamraża czas" w przeglądarce i renderuje klatka-po-klatce deterministycznie.

## Input

**Folder z Claude Design eksportem** (standalone HTML export):
```
<bundle>/
├── *.html              # główny — React + Babel + <Stage> + useTimeline
├── animations.jsx      # silnik czasu (useTime, useTimeline, Stage, Sprite)
├── scenes/*.jsx        # komponenty scen
└── assets/             # obrazki, fonty, pliki
```

Default: `/claude-design-to-video` bez argumentu → autodetect w `~/Downloads/` (szuka folderu z HTML zawierającym `<Stage` i `useTimeline`).

Explicit: `/claude-design-to-video "~/Downloads/Animacja YouTube (1)"` [flagi]

## Flagi

| flag | efekt | default |
|---|---|---|
| `--hq` | CRF 18, preset slow (YouTube source grade) | ✓ on |
| `--fast` | CRF 20, preset medium (iteracja, -30% czasu) | off |
| `--preview N` | tylko N sekund zamiast pełnego duration (do szybkiego sprawdzenia) | off |
| `--4k` | 3840×2160 (default 1920×1080) | off |
| `--fps N` | frame rate (30 default — bo YT tak zapisuje i tak) | 30 |
| `--no-patch` | pomiń auto-patch CSS (gdy bundle już clean) | off |
| `--in-place` | patchuj pliki w oryginalnym folderze zamiast kopii | off |
| `--output PATH` | override output location | `~/Downloads/<slug>-<WxH>-<fps>-<mode>.mp4` |

## Preflight (fail fast)

Sprawdź obecność:
- `ffmpeg --version` (≥ 4)
- Chrome/Chromium executable (auto-detect)
- `node --version` (≥ 18)
- `python3 --version` (≥ 3.8)

**Chrome auto-detect** (pierwszy istniejący z listy):
```bash
CHROME_PATHS=(
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"   # macOS
  "/Applications/Chromium.app/Contents/MacOS/Chromium"             # macOS Chromium
  "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser"   # macOS Brave
  "/usr/bin/google-chrome"                                          # Linux Debian/Ubuntu
  "/usr/bin/chromium-browser"                                       # Linux Ubuntu
  "/usr/bin/chromium"                                               # Linux Arch
  "/snap/bin/chromium"                                              # Linux Snap
  "$LOCALAPPDATA/Google/Chrome/Application/chrome.exe"              # Windows
  "/c/Program Files/Google/Chrome/Application/chrome.exe"           # Windows Git Bash
)
CHROME_BIN=""
for p in "${CHROME_PATHS[@]}"; do
  [ -x "$p" ] && CHROME_BIN="$p" && break
done
[ -z "$CHROME_BIN" ] && fail "Nie znaleziono Chrome/Chromium. Zainstaluj: https://www.google.com/chrome/"
```

Jak brak jakiejkolwiek zależności → komunikat co zainstalować i przerwij.

## Pipeline

### 1. Locate bundle

Jeśli arg podane — użyj. Inaczej autodetect:

```bash
find ~/Downloads -maxdepth 3 -name "*.html" -type f 2>/dev/null | while read f; do
  if grep -q "useTimeline\|<Stage" "$f" 2>/dev/null; then
    dirname "$f"
    break
  fi
done
```

Walidacja: folder MUSI zawierać `animations.jsx` + `scenes/` + co najmniej jeden `.html`. Jeśli nie — fail z sugestią eksportu z Claude Design jako "standalone HTML".

### 2. Parse bundle metadata

Wyciągnij:
- **DURATION** — `grep "DURATION = " *.html` → liczba sekund (fallback: 60, z warning)
- **viewport** — portable (BSD/GNU compatible):
  ```bash
  WIDTH=$(grep -oE "width=\{[0-9]+" *.html | grep -oE "[0-9]+" | head -1)
  HEIGHT=$(grep -oE "height=\{[0-9]+" *.html | grep -oE "[0-9]+" | head -1)
  WIDTH=${WIDTH:-1920}; HEIGHT=${HEIGHT:-1080}
  ```
  (nie używaj `grep -oP` — BSD grep na macOS nie wspiera Perl regex)
- **HTML entry** — pierwszy `.html` w folderze (zwykle jeden)

### 3. Copy bundle do temp (default, bez `--in-place`)

```bash
TMPDIR=$(mktemp -d -t cdv.XXXXXX)
WORK="$TMPDIR/bundle"
cp -R "$BUNDLE_PATH" "$WORK"
trap 'kill "${HTTP_PID:-0}" 2>/dev/null; rm -rf "$TMPDIR"' EXIT INT TERM
```

Z `--in-place` pracuj bezpośrednio na `$BUNDLE_PATH` (user's responsibility).

### 4. Auto-patch CSS bugs (skip z `--no-patch`)

Claude Design eksport ma **znane** bugi które łamią frame-by-frame capture. timecut przechwytuje tylko JS time APIs (`requestAnimationFrame`, `setTimeout`, `Date.now`) — **nie CSS animations/transitions**. Wszystkie CSS-time rzeczy biegną wall-clock i psują render.

#### Patch A — Stage `PlaybackBar` visible w rendercie

W `animations.jsx` Stage renderuje scrubber (play/pause, czas, track). Na screenshot to widać jako pasek na dole.

**Wykryj:** `<PlaybackBar` w `animations.jsx`
**Fix:** Owiń w condition na URL param `?render=1`:

```diff
-      <PlaybackBar
-        time={displayTime}
-        ...
-      />
+      {!(typeof window !== 'undefined' && window.location.search.includes('render=1')) && (
+        <PlaybackBar
+          time={displayTime}
+          ...
+        />
+      )}
```

Timecut dostaje URL z `?render=1`, scrubber znika, canvas wypełnia okno.

#### Patch B — `@keyframes blink` (WebcamPip)

**Wykryj (conservative):**
1. Grep w JSX: `animation:\s*['"](\w+)\s+[\d.]+s\s+infinite['"]` → zbierz nazwy (`\w+`)
2. Dla każdej nazwy sprawdź czy w **tym samym pliku** istnieje `@keyframes <name>` (w `<style>` lub CSS)
3. Patchuj **tylko wtedy** gdy oba są present. Sam `animation:` bez `@keyframes` może być JS-driven (Web Animations API) — wtedy zostaw.

Po zidentyfikowaniu: zapytaj usera przed zmianą (nie ślepo patch). Pokaż diff.

**Fix dla blink 1.2s z opacity 1→0.3:**

```diff
-          animation: 'blink 1.2s infinite',
+          opacity: (p => p < 0.6 ? 1 : p < 0.7 ? 1 - (p - 0.6) * 7 : 0.3)((time % 1.2) / 1.2),
```

I usuń `<style>{`@keyframes blink ...`}</style>` blok.

Uwaga: `time` musi być w scope (z `useTimeline()` lub `useTime()`). Sprawdź czy komponent już destrukturyzuje `time` — jeśli nie, dodaj.

#### Patch C — `transition: opacity Nms` (webcam hidden windows fade)

**Wykryj:** `transition: ... opacity \d+ms cubic-bezier` w WebcamPip.jsx
**Fix:** Zamień instant opacity + CSS transition na JS ease-out cubic:

```diff
-  const hidden = state.autoHide && WEBCAM_HIDDEN_WINDOWS.some(([s, e]) => time >= s && time <= e);
-  const opacity = hidden ? 0 : 1;
+  let opacity = 1;
+  if (state.autoHide) {
+    const fadeTime = 0.5;
+    let mul = 1;
+    for (const [s, e] of WEBCAM_HIDDEN_WINDOWS) {
+      if (time >= s - fadeTime && time <= e + fadeTime) {
+        if (time < s) mul = Math.min(mul, (s - time) / fadeTime);
+        else if (time > e) mul = Math.min(mul, (time - e) / fadeTime);
+        else mul = 0;
+      }
+    }
+    opacity = 1 - Math.pow(1 - mul, 3);
+  }
```

I usuń `transition: '...opacity 500ms...'` (lub zamień na `transition: 'none'`).

#### Patch D — Webcam idle motion (opcjonalny)

**Wykryj:** `const sway = .* Math.sin(time` i `const breathe = .* Math.sin(time` w WebcamPip.jsx
**Fix:** Wyłącz (subpixel jitter wygląda brzydko w rendercie):

```diff
-  const sway = drag ? 0 : Math.sin(time * 0.8) * 2;
-  const breathe = drag ? 1 : 1 + Math.sin(time * 1.1) * 0.006;
+  const sway = 0;
+  const breathe = 1;
```

#### Patches E+ — nowe wzorce

Jak Claude Design zmieni strukturę eksportu → rozszerz listę patches. Uruchom render bez patch'a, identyfikuj co się psuje, dodaj wzorzec tutaj.

### 5. Setup persistent cache + temp workspace

```bash
CACHE="$HOME/.cache/claude-design-to-video"
mkdir -p "$CACHE/timecut"
# Reuse npm install (timecut to ~150MB, zero sensu reinstall za każdym razem)
if [ ! -x "$CACHE/timecut/node_modules/.bin/timecut" ]; then
  echo "Installing timecut (one-time, ~30s)..."
  (cd "$CACHE/timecut" && echo '{}' > package.json && npm install timecut)
fi
TIMECUT_BIN="$CACHE/timecut/node_modules/.bin/timecut"
```

### 6. Start HTTP server (wolny port)

```bash
# Pick free port dynamically (49000-49999 safe range)
PORT=$(python3 -c "import socket; s=socket.socket(); s.bind(('',0)); print(s.getsockname()[1]); s.close()")
cd "$WORK" && python3 -m http.server "$PORT" > /dev/null 2>&1 &
HTTP_PID=$!
sleep 1
# Verify server responds
curl -sI "http://localhost:$PORT/$HTML_FILE" | grep -q "200 OK" || fail "HTTP server failed"
```

### 7. Render timecut

**Monitor postępu w drugim terminalu** (1080p30 74s trwa ~15 min):
```bash
tail -f /tmp/cdv.*/render.log 2>/dev/null | grep --line-buffered -oE "frame=\s*[0-9]+"
```
Albo po prostu czekaj — Claude Code powiadomi gdy background task się skończy.

```bash
# Slug z nazwy folderu (usuń spacje, lower-case)
SLUG=$(basename "$BUNDLE_PATH" | tr ' ' '-' | tr '[:upper:]' '[:lower:]')
OUTPUT="${OUTPUT:-$HOME/Downloads/${SLUG}-${WIDTH}x${HEIGHT}-${FPS}-${MODE}.mp4}"

# Quality mode → ffmpeg flags
case "$MODE" in
  hq)   CRF=18; PRESET=slow ;;
  fast) CRF=20; PRESET=medium ;;
esac

# URL-encode HTML file name (may contain spaces)
HTML_ENC=$(python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1]))" "$HTML_FILE")

"$TIMECUT_BIN" \
  "http://localhost:$PORT/$HTML_ENC?render=1" \
  --viewport="${WIDTH},${HEIGHT}" \
  --fps="$FPS" \
  --duration="$DURATION" \
  --output="$OUTPUT" \
  --executable-path="$CHROME_BIN" \
  --output-options="-crf $CRF -preset $PRESET -pix_fmt yuv420p"
```

### 8. Cleanup (automatyczny przez trap)

`trap` z kroku 3 wykona się na:
- normalnym zakończeniu (EXIT)
- Ctrl+C (INT)
- zabiciu (TERM)
- błędzie (via `set -e`)

Usuwa: HTTP server PID, cały `$TMPDIR`.

Cache w `~/.cache/claude-design-to-video/timecut/` **zostaje** — reuse przy następnym renderze.

### 9. Raport + validation + open

```bash
echo "✓ Render gotowy:"
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,r_frame_rate,nb_frames,duration,bit_rate \
  -of default=nw=1 "$OUTPUT"
ls -lh "$OUTPUT"

# Validation: czy liczba klatek pasuje do oczekiwanej?
EXPECTED_FRAMES=$((DURATION * FPS))
ACTUAL=$(ffprobe -v error -select_streams v:0 -show_entries stream=nb_frames -of csv=p=0 "$OUTPUT")
if [ "$ACTUAL" != "$EXPECTED_FRAMES" ]; then
  echo "⚠ Frame mismatch: expected $EXPECTED_FRAMES, got $ACTUAL — render może być niekompletny"
  echo "  (patrz sekcja Failure modes → 'Frame count mismatch')"
fi

open "$OUTPUT"  # QuickTime
```

## Przewidywany czas (M-chip Mac, tej animacji 74s)

| rozdzielczość | mode | czas renderu |
|---|---|---|
| 1080p30 | `--fast` | ~10 min |
| 1080p30 | `--hq` (default) | ~15 min |
| 1080p30 | `--preview 15` | ~1 min |
| 4K30 | `--hq` | ~40-60 min |

Powolność wynika z: Chrome renderuje pojedyncze klatki (szczególnie cięższe sceny typu Scene7_Features z flex expansion) + CRF 18 preset slow = wolniejszy encoding.

## Failure modes (co może pójść źle)

| Symptom | Przyczyna | Reakcja |
|---|---|---|
| `curl` w kroku 6 nie daje 200 OK | HTTP server port zajęty (rare, bo dynamic) | Retry z nowym portem (do 3 prób). Potem fail z diagnostyką. |
| `TimeoutError: ... connect to the browser` | Chrome nie startuje (brak exec, za dużo instancji) | Sprawdź `$CHROME_BIN` (preflight auto-detect). Gdy user ma 20+ Chrome instancji otwartych — zapytaj o zamknięcie (`killall "Google Chrome"` na macOS / `pkill chromium` na Linux) przed retry. |
| timecut exit ≠ 0, partial MP4 w outputcie | Crash mid-render (Chrome memory leak, bug ffmpeg) | Usuń partial (`rm -f "$OUTPUT"`) + pokaż tail `/tmp/cdv.*/render.log`. Retry z `--fast` jak Chrome issue. |
| `npm install timecut` fail | Offline / pakiet zniknął z npm | Jak cache `~/.cache/claude-design-to-video/timecut/node_modules/.bin/timecut` istnieje → użyj (pomiń install). Jak nie i brak netu → fail z "Pierwsza instalacja wymaga internetu (~150MB)". |
| Render zwisa (brak log updateów 2+ min) | Chrome headless bug na konkretnej scenie | Kill timecut, retry z `--fast`. Jak znów — workaround: renderuj fragmenty przez `--preview N` z różnymi offsetami. |
| Frame count mismatch w validation | Cichy drop klatek (ffmpeg/timecut timing issue) | Re-render z `--fast`. Jak się powtarza — prawdopodobnie bug w bundle (sprawdź nieznane CSS transitions, rozszerz Patch E+). |

## Known limitations

- **Scene7_Features** w tej konkretnej animacji ma `transition: flex 600ms cubic-bezier` i `transition: 'all 400ms'` — auto-patch ich **nie obejmuje** (flex interpolation wymaga znajomości wymiarów, ryzyko zepsucia layoutu). Przy rozwijających się panelach funkcji może być widoczny jitter. Workaround: ręczny port na JS interpolate lub pominąć (zwykle efekt niezauważalny).
- **Nowe patterns Claude Design** — co N miesięcy autor może zmienić eksport. Jak render wychodzi brudny → zidentyfikuj wzorzec + dodaj patch (sekcja 4).
- **Intel Maca** — puppeteer/Chrome będzie wolniejszy (1.5-2×). M-chip zalecany.

## Cache layout

```
~/.cache/claude-design-to-video/
└── timecut/            # persistent npm install (reuse, ~150MB)
    └── node_modules/
```

## Temp (zawsze sprzątany)

```
/tmp/cdv.XXXXXX/
├── bundle/             # kopia eksportu do patchowania
└── (timecut-temp-*)    # timecut sam sprząta swoje frames folder
```

## Przykłady

```bash
# Default: autodetect, 1080p30, HQ, output do Downloads
/claude-design-to-video

# Konkretny folder
/claude-design-to-video "~/Downloads/Animacja YouTube (1)"

# Szybka iteracja: 15s preview, mniej jakości
/claude-design-to-video --preview 15 --fast

# 4K finalny render (długo)
/claude-design-to-video --4k

# Custom output
/claude-design-to-video --output "~/videos/intro-yt.mp4"

# Bundle już clean (nie patchować), patchuj in-place
/claude-design-to-video "~/my-animation" --no-patch --in-place
```

## Do rozbudowy w przyszłości

- Automatyczny **thumbnail** z klatki N (np. frame 60 = 2s)
- Export do **webm/VP9** dla web (mniejsze pliki)
- **Remotion port** — bundle który ma za dużo CSS → auto-konwersja do Remotion project (skill `/claude-design-to-remotion`?)
- **Batch mode** — renderuj N bundli równolegle
- **Handoff URL support** — jak Anthropic zrobi publiczny endpoint, fetch bezpośrednio zamiast wymagać lokalnego folderu
