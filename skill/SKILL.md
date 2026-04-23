---
name: claude-design-to-video
description: "Użyj tego skilla gdy user ma gotową animację (folder/bundle/eksport HTML/React, często w ~/Downloads, np. 'Animacja YouTube') i chce z niej plik wideo/MP4/film. Intencja: animacja → MP4. Typowe sformułowania: 'wyrenderuj animację', 'wyrenderuj mi to', 'zrób film/wideo z animacji', 'zamień animację na MP4', 'potrzebuję MP4 na YT/social', '4K/1080p/60fps render animacji', 'mam folder/bundle → chcę plik wideo'. Triggeruj też gdy user wspomina Claude Design, useTimeline, Stage, animations.jsx, standalone HTML export, lub po prostu 'folder w Downloads z animacją' — niezależnie czy pada nazwa 'Claude Design'. Krótkie frazy typu 'wyrenderuj mi ta animacje' + kontekst gotowego folderu/bundla = trigger. NIE używaj do: transcodowania istniejących plików MP4/wideo, montażu nagrań z kamery/telefonu/GoPro, ekstrakcji audio, projektów Remotion, zwykłego HTML bez animacji JS, generowania grafik/thumbnaili/banerów, podsumowań filmów YouTube. Reguła: animowany artefakt HTML/React + intencja 'chcę plik wideo' = ten skill."
argument-hint: "[folder-bundle | --flags]"
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
---

# /claude-design-to-video

**Platformy:** macOS (main-tested) + Linux (should work, community-tested). **Nie Windows** — skill używa bash/POSIX toolingu (`mktemp`, `trap`, `pkill`, `open`), którego native Windows PowerShell nie ogarnia. Dla Windows użyj WSL2.

Claude Design eksportuje animacje jako standalone HTML (React + Babel + JSX scenes). Żeby zamienić to w MP4 bez drop frames i glitchy — używamy **timecut** (Puppeteer + wirtualny czas + ffmpeg). Nagrywanie real-time (Playwright/OBS) daje szajs — timecut "zamraża czas" w przeglądarce i renderuje klatka-po-klatce deterministycznie.

## ⚠️ Security note

Ten skill **wykonuje arbitralny JavaScript z bundle'a w lokalnym Chrome** i serwuje pliki na `localhost`. Renderuj tylko **zaufane eksporty z Claude Design** (własne lub od znanego source). Bundle od obcej osoby = arbitrary code execution na Twoim systemie.

## Input

**Folder z Claude Design eksportem** (standalone HTML export):
```
<bundle>/
├── *.html              # główny — React + Babel + <Stage> + useTimeline
├── animations.jsx      # silnik czasu (useTime, useTimeline, Stage, Sprite)
├── scenes/*.jsx        # komponenty scen
└── assets/             # obrazki, fonty, pliki
```

Default: `/claude-design-to-video` bez argumentu → autodetect w `~/Downloads/` (szuka folderu z HTML zawierającym **BOTH** `<Stage` AND `useTimeline`, plus walidacja że folder zawiera `animations.jsx` i `scenes/`).

Explicit: `/claude-design-to-video "~/Downloads/Animacja YouTube (1)"` [flagi]

## Flagi

| flag | efekt | default |
|---|---|---|
| `--hq` | CRF 18, preset slow (YouTube source grade) | ✓ on |
| `--fast` | CRF 20, preset medium (iteracja, -30% czasu) | off |
| `--preview N` | tylko N sekund zamiast pełnego duration (do szybkiego sprawdzenia) | off |
| `--4k` | 3840×2160 (default 1920×1080) | off |
| `--fps N` | frame rate (30 default — bo YT tak zapisuje i tak) | 30 |
| `--duration N` | override parsed duration z HTML (wymagane gdy brak `DURATION = N` w HTMLu) | auto |
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
)
CHROME_BIN=""
for p in "${CHROME_PATHS[@]}"; do
  [ -x "$p" ] && CHROME_BIN="$p" && break
done
[ -z "$CHROME_BIN" ] && fail "Nie znaleziono Chrome/Chromium. Zainstaluj: https://www.google.com/chrome/"
```

Jak brak jakiejkolwiek zależności → komunikat co zainstalować i przerwij.

## Pipeline

### 1. Locate bundle (portable — bez GNU find)

Jeśli arg podane — użyj. Inaczej autodetect via Python (BSD find na macOS nie ma `-maxdepth`):

```bash
BUNDLE_PATH=$(python3 <<'PY'
import os, sys
root = os.path.expanduser('~/Downloads')
for dirpath, dirs, files in os.walk(root):
    # Max depth 3
    depth = dirpath[len(root):].count(os.sep)
    if depth >= 3:
        dirs[:] = []
        continue
    for f in files:
        if not f.endswith('.html'):
            continue
        fp = os.path.join(dirpath, f)
        try:
            with open(fp, encoding='utf-8', errors='ignore') as fh:
                content = fh.read()
        except OSError:
            continue
        # REQUIRE BOTH markers (not OR) — reduces false positives
        if 'useTimeline' in content and '<Stage' in content:
            # Validate sibling artifacts (animations.jsx + scenes/)
            if os.path.isfile(os.path.join(dirpath, 'animations.jsx')) \
               and os.path.isdir(os.path.join(dirpath, 'scenes')):
                print(dirpath)
                sys.exit(0)
sys.exit(1)
PY
)
[ -z "$BUNDLE_PATH" ] && fail "Nie znaleziono bundle w ~/Downloads. Podaj ścieżkę explicite: /claude-design-to-video <path>"
```

### 2. Parse bundle metadata (tylko z wybranego HTML entry)

```bash
# Entry HTML — pierwszy .html w folderze (Claude Design zwykle generuje tylko jeden)
HTML_FILE=$(ls "$BUNDLE_PATH"/*.html 2>/dev/null | head -1 | xargs basename)
[ -z "$HTML_FILE" ] && fail "Brak pliku .html w $BUNDLE_PATH"

# DURATION — fail fast, nie silent fallback (chyba że user dał --duration N)
if [ -z "$DURATION_OVERRIDE" ]; then
  DURATION=$(grep -oE "DURATION\s*=\s*[0-9]+" "$BUNDLE_PATH/$HTML_FILE" | grep -oE "[0-9]+" | head -1)
  if [ -z "$DURATION" ]; then
    fail "Nie znaleziono DURATION w $HTML_FILE. Podaj explicite: --duration N (sekund)"
  fi
else
  DURATION="$DURATION_OVERRIDE"
fi

# Viewport — whitespace-aware, tylko z entry HTML
WIDTH=$(grep -oE "width=\{\s*[0-9]+" "$BUNDLE_PATH/$HTML_FILE" | grep -oE "[0-9]+" | head -1)
HEIGHT=$(grep -oE "height=\{\s*[0-9]+" "$BUNDLE_PATH/$HTML_FILE" | grep -oE "[0-9]+" | head -1)
WIDTH=${WIDTH:-1920}
HEIGHT=${HEIGHT:-1080}
# --4k override
[ "$FOURK" = "1" ] && WIDTH=3840 && HEIGHT=2160
```

### 3. Setup workspace (dual: copy-to-temp vs in-place)

**Oba tryby** instalują trap — cleanup zawsze działa.

```bash
if [ "$IN_PLACE" = "1" ]; then
  WORK="$BUNDLE_PATH"
  TMPDIR=""
  echo "⚠ --in-place mode: modyfikuję oryginalny folder $BUNDLE_PATH"
else
  TMPDIR=$(mktemp -d -t cdv.XXXXXX)
  WORK="$TMPDIR/bundle"
  cp -R "$BUNDLE_PATH" "$WORK"
fi

# Cleanup trap — kill HTTP server + render process group + temp dir
cleanup() {
  [ -n "$HTTP_PID" ] && kill "$HTTP_PID" 2>/dev/null || true
  [ -n "$RENDER_PID" ] && kill "$RENDER_PID" 2>/dev/null || true
  # Kill wszystkie dzieci tego shella (Chrome helpers itd.)
  pkill -P $$ 2>/dev/null || true
  [ -n "$TMPDIR" ] && [ -d "$TMPDIR" ] && rm -rf "$TMPDIR"
}
trap cleanup EXIT INT TERM
```

### 4. Auto-patch CSS bugs (skip z `--no-patch`)

Claude Design eksport ma **znane** bugi które łamią frame-by-frame capture. timecut przechwytuje tylko JS time APIs (`requestAnimationFrame`, `setTimeout`, `Date.now`) — **nie CSS animations/transitions**. Wszystkie CSS-time rzeczy biegną wall-clock i psują render.

**Policy:** fix silently, na koniec pokaż **raport co zmieniono** (user widzi diff bez interrupcji). Jeśli user nie zgadza się → odpal ponownie z `--no-patch` i patch ręcznie.

#### Patch A — Stage `PlaybackBar` visible w rendercie

W `animations.jsx` Stage renderuje scrubber (play/pause, czas, track). Na screenshot widać jako pasek na dole.

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

#### Patch B — `@keyframes blink` → JS opacity

**Wykryj (conservative — wymaga BOTH patterns w tym samym pliku):**
1. Grep w JSX: `animation:\s*['"](\w+)\s+[\d.]+s\s+infinite['"]` → zbierz nazwy
2. Dla każdej nazwy sprawdź czy w **tym samym pliku** istnieje `@keyframes <name>` (w `<style>` lub CSS)
3. Patchuj **tylko wtedy** gdy oba są present

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

#### Patch D — Webcam idle motion

**Wykryj:** `const sway = .* Math.sin(time` i `const breathe = .* Math.sin(time` w WebcamPip.jsx
**Fix:** Wyłącz (subpixel jitter wygląda brzydko w rendercie):

```diff
-  const sway = drag ? 0 : Math.sin(time * 0.8) * 2;
-  const breathe = drag ? 1 : 1 + Math.sin(time * 1.1) * 0.006;
+  const sway = 0;
+  const breathe = 1;
```

#### Patch report

Po wszystkich patchach pokaż raport:
```
✓ Auto-patch applied (4 changes):
  - animations.jsx:467 — Patch A (PlaybackBar hidden w render mode)
  - scenes/WebcamPip.jsx:193 — Patch B (blink → JS opacity)
  - scenes/WebcamPip.jsx:80 — Patch C (fade transition → JS ease-out)
  - scenes/WebcamPip.jsx:83-84 — Patch D (sway/breathe disabled)
```

#### Patches E+ — nowe wzorce

Jak Claude Design zmieni strukturę eksportu → rozszerz listę patches. Uruchom render bez patch'a, identyfikuj co się psuje, dodaj wzorzec tutaj.

### 5. Install timecut (pinned version, cached)

```bash
CACHE="$HOME/.cache/claude-design-to-video"
mkdir -p "$CACHE/timecut"
TIMECUT_BIN="$CACHE/timecut/node_modules/.bin/timecut"
# Pin version for reproducibility — supply-chain safety
TIMECUT_VERSION="0.3.3"

if [ ! -x "$TIMECUT_BIN" ]; then
  echo "Installing timecut@$TIMECUT_VERSION (one-time, ~150MB)..."
  (cd "$CACHE/timecut" && echo '{}' > package.json && npm install "timecut@$TIMECUT_VERSION") \
    || fail "npm install timecut@$TIMECUT_VERSION failed. Sprawdź internet, albo zainstaluj ręcznie: cd $CACHE/timecut && npm install timecut@$TIMECUT_VERSION"
fi
```

### 6. Start HTTP server (retry loop dla race-free port)

```bash
HTML_ENC=$(python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1]))" "$HTML_FILE")

for attempt in 1 2 3; do
  # Pick free port
  PORT=$(python3 -c "import socket; s=socket.socket(); s.bind(('',0)); print(s.getsockname()[1]); s.close()")
  cd "$WORK" && python3 -m http.server "$PORT" > /dev/null 2>&1 &
  HTTP_PID=$!
  # Wait for server to actually respond (up to 3s)
  for i in 1 2 3 4 5 6; do
    sleep 0.5
    if curl -sI "http://localhost:$PORT/$HTML_ENC" 2>/dev/null | grep -q "200 OK"; then
      break 2  # success, exit both loops
    fi
  done
  # Failed this attempt, kill and retry
  kill "$HTTP_PID" 2>/dev/null
  HTTP_PID=""
  [ "$attempt" = "3" ] && fail "HTTP server nie wystartował po 3 próbach. Sprawdź czy port 49xxx nie jest blokowany."
done
```

### 7. Render timecut

**Monitor postępu** (portable, BSD-safe):
```bash
# tail + awk zamiast grep --line-buffered (GNU-only)
tail -f /tmp/cdv.*/render.log 2>/dev/null | awk '/frame=/ {print}'
```

```bash
# Slug z nazwy folderu (usuń spacje, lower-case)
SLUG=$(basename "$BUNDLE_PATH" | tr ' ' '-' | tr '[:upper:]' '[:lower:]')
OUTPUT="${OUTPUT:-$HOME/Downloads/${SLUG}-${WIDTH}x${HEIGHT}-${FPS}-${MODE}.mp4}"

# Quality mode → ffmpeg flags
case "$MODE" in
  hq)   CRF=18; PRESET=slow ;;
  fast) CRF=20; PRESET=medium ;;
esac

# Preview mode — render tylko N sekund
RENDER_DURATION="${PREVIEW_SEC:-$DURATION}"

# Odpal timecut w background, zapisz PID dla cleanup
"$TIMECUT_BIN" \
  "http://localhost:$PORT/$HTML_ENC?render=1" \
  --viewport="${WIDTH},${HEIGHT}" \
  --fps="$FPS" \
  --duration="$RENDER_DURATION" \
  --output="$OUTPUT" \
  --executable-path="$CHROME_BIN" \
  --output-options="-crf $CRF -preset $PRESET -pix_fmt yuv420p" &
RENDER_PID=$!
wait "$RENDER_PID"
RENDER_EXIT=$?

# Jak render się wywalił — usuń partial mp4
if [ "$RENDER_EXIT" -ne 0 ]; then
  rm -f "$OUTPUT"
  fail "timecut zakończył się błędem (exit $RENDER_EXIT). Sprawdź log: /tmp/cdv.*/render.log"
fi
```

### 8. Cleanup (automatyczny przez trap)

`trap` z kroku 3 wykona się na:
- normalnym zakończeniu (EXIT)
- Ctrl+C (INT)
- zabiciu (TERM)
- błędzie (via `set -e`)

Usuwa:
- HTTP server PID
- render PID (timecut + wszystkie dzieci shella — Chrome helpers)
- cały `$TMPDIR` (jeśli był tworzony, nie in-place)

Cache w `~/.cache/claude-design-to-video/timecut/` **zostaje** — reuse przy następnym renderze (~150MB).

### 9. Raport + validation + open

```bash
echo "✓ Render gotowy:"
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,r_frame_rate,nb_frames,duration,bit_rate \
  -of default=nw=1 "$OUTPUT"
ls -lh "$OUTPUT"

# Validation: czy liczba klatek pasuje do oczekiwanej?
EXPECTED_FRAMES=$((RENDER_DURATION * FPS))
ACTUAL=$(ffprobe -v error -select_streams v:0 -show_entries stream=nb_frames -of csv=p=0 "$OUTPUT")
if [ "$ACTUAL" != "$EXPECTED_FRAMES" ]; then
  echo "⚠ Frame mismatch: expected $EXPECTED_FRAMES, got $ACTUAL — render może być niekompletny"
  echo "  (patrz sekcja Failure modes → 'Frame count mismatch')"
fi

# Open w domyślnym playerze (macOS: QuickTime; Linux: xdg-open)
if command -v open >/dev/null 2>&1; then
  open "$OUTPUT"         # macOS
elif command -v xdg-open >/dev/null 2>&1; then
  xdg-open "$OUTPUT"     # Linux
fi
```

## Przewidywany czas (M-chip Mac, animacja 74s)

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
| `HTTP server nie wystartował po 3 próbach` | Race na wolnym porcie, firewall, brak python3 | Sprawdź czy port 49xxx nie jest blokowany. Try `python3 -m http.server 49999` ręcznie. |
| `TimeoutError: ... connect to the browser` | Chrome nie startuje (brak exec, za dużo instancji) | Sprawdź `$CHROME_BIN` (preflight auto-detect). Gdy user ma 20+ Chrome instancji otwartych — zapytaj o zamknięcie (`killall "Google Chrome"` na macOS / `pkill chromium` na Linux) przed retry. |
| timecut exit ≠ 0, partial MP4 | Crash mid-render (Chrome memory leak, bug ffmpeg) | Trap `cleanup()` usuwa partial. Pokaż tail `/tmp/cdv.*/render.log`. Retry z `--fast` jak Chrome issue. |
| `npm install timecut@0.3.3 failed` | Offline / npm down | Jak cache `~/.cache/claude-design-to-video/timecut/node_modules/.bin/timecut` już istnieje → użyj (skip install). Jak nie i brak netu → fail z komunikatem. |
| Render zwisa (brak log updateów 2+ min) | Chrome headless bug na konkretnej scenie | Kill timecut (trap się zajmie), retry z `--fast`. Jak znów — workaround: renderuj fragmenty przez `--preview N` z offsetami. |
| Frame count mismatch w validation | Cichy drop klatek (timing issue) | Re-render z `--fast`. Jak się powtarza — prawdopodobnie bug w bundle (sprawdź nieznane CSS transitions, rozszerz Patch E+). |
| `Nie znaleziono DURATION w HTML` | Bundle bez `DURATION = N` (nietypowy export) | Podaj `--duration N` explicite. |
| `Nie znaleziono bundle w ~/Downloads` | Autodetect nie znalazł folderu z oboma markerami | Podaj ścieżkę explicite: `/claude-design-to-video "~/Downloads/<folder>"`. |

## Known limitations

- **CSS transitions z dynamic layout** (np. `transition: flex 600ms cubic-bezier`) nie są auto-patched — flex interpolation wymaga znania wymiarów, zbyt ryzykowne. Jeśli trafisz — ręczny port na JS `interpolate()`.
- **Nowe patterns Claude Design** — co N miesięcy autor może zmienić eksport. Jak render wychodzi brudny → zidentyfikuj wzorzec + dodaj patch (sekcja 4).
- **Intel Mac** — Puppeteer/Chrome wolniejszy (1.5-2×). M-chip zalecany.
- **Windows** — nie wspierany natywnie. Użyj WSL2 (Linux distro wewnątrz Windows) — pełna zgodność.

## Cache layout

```
~/.cache/claude-design-to-video/
└── timecut/            # persistent npm install (reuse, ~150MB)
    ├── package.json    # pinned timecut@0.3.3
    └── node_modules/
```

Cache **zostaje między renderami** — reuse. Usuwaj tylko gdy chcesz wymusić re-install (np. bump timecut version).

## Temp (zawsze sprzątany przez trap)

```
/tmp/cdv.XXXXXX/
├── bundle/             # kopia eksportu do patchowania (tylko gdy NIE --in-place)
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

# Override duration (gdy bundle nie ma DURATION = N w HTMLu)
/claude-design-to-video --duration 45
```

## Do rozbudowy w przyszłości

- Automatyczny **thumbnail** z klatki N (np. frame 60 = 2s)
- Export do **webm/VP9** dla web (mniejsze pliki)
- **Remotion port** — bundle który ma za dużo CSS → auto-konwersja do Remotion project (skill `/claude-design-to-remotion`?)
- **Batch mode** — renderuj N bundli równolegle
- **Handoff URL support** — jak Anthropic zrobi publiczny endpoint, fetch bezpośrednio zamiast wymagać lokalnego folderu
- **Windows native** — PowerShell + Chocolatey branch (niska priority — WSL2 rozwiązuje)
