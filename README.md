# claude-design-to-video (macOS + Linux)

> **Claude Design HTML export → MP4 video** — deterministic frame-by-frame rendering przez timecut.

Claude Design (Anthropic Labs) eksportuje animowane prototypy jako **standalone HTML** (React + Babel + JSX sceny). Ten skill konwertuje eksport do MP4 bez drop frames, glitchy i utraty jakości — gotowe na YouTube, Instagram, LinkedIn, gdziekolwiek.

---

## ⚠️ Uwaga bezpieczeństwa

Ten skill **wykonuje arbitralny JavaScript z bundle'a w Twoim lokalnym Chrome** i serwuje pliki na `localhost`. Renderuj tylko **zaufane eksporty** (własne lub od znanego źródła). Bundle od obcej osoby = arbitrary code execution na Twoim systemie. Traktuj to jak odpalanie nieznanego skryptu.

---

## Po co to istnieje

Nagrywanie animacji z Claude Design przez Playwright/Puppeteer/OBS **nie działa**. Przeglądarka renderuje klatki kiedy może, gubi klatki pod obciążeniem, i binduje animacje do wall-clock time. Jak screenshot trwa 200 ms a animacja oczekuje klatek co 16 ms — dostajesz szarpany bałagan.

**timecut** rozwiązuje to przez **wirtualizację czasu** — override'uje `requestAnimationFrame`, `setTimeout`, `Date.now`, `performance.now` wewnątrz przeglądarki, potem renderuje każdą klatkę w jej dokładnym virtualnym timestamp. Screenshoty mogą trwać ile chcą, wynikowe wideo jest perfectly smooth.

Skill **auto-patchuje znane CSS bugi** w eksportach Claude Design (scrubber, blink keyframes, fade transitions, webcam idle motion) przed renderem — nie musisz ręcznie edytować eksportu.

---

## Co dostajesz

- **MP4 output**, 1920×1080 @ 30 fps domyślnie (albo 4K, plus custom fps przez `--fps`)
- **YouTube-grade jakość** — H.264, yuv420p, CRF 18, preset slow
- **Zero drop frames** — każda klatka renderowana deterministycznie
- **Auto-patched CSS bugs** z raportem zmian — zobacz [Auto-patche](#auto-patche) niżej
- **Sprzątanie** — temp files i kopia bundle'a usunięte przez `trap` po każdym renderze
- **Persistent cache timecut** (~150 MB w `~/.cache/claude-design-to-video/`) — reuse między renderami

---

## Wspierane platformy

| OS | Status |
|---|---|
| **macOS** (Intel + Apple Silicon) | ✅ Główna platforma, testowane |
| **Linux** (Debian/Ubuntu/Arch) | ✅ Powinno działać — feedback mile widziany |
| **Windows** | ❌ Nie wspierane natywnie. Użyj [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) z Linux distro wewnątrz. |

Skill używa bash/POSIX toolingu (`mktemp`, `trap`, `pkill`, `open`/`xdg-open`). Windows PowerShell nie może uruchomić tego flow bezpośrednio. Native Windows branch jest w roadmapie ale niski priorytet — WSL2 rozwiązuje to dziś.

---

## Wymagania

| Narzędzie | Minimum | Sprawdź |
|---|---|---|
| **Claude Code** | latest | skill ładuje się przez `.claude/skills/` |
| **ffmpeg** | ≥ 4 | `ffmpeg --version` |
| **Node.js** | ≥ 18 | `node --version` |
| **Python 3** | ≥ 3.8 | `python3 --version` (dla HTTP serwera) |
| **Chrome / Chromium** | dowolny recent | auto-detect na macOS / Linux |

**Auto-install brakujących zależności:** Skill sam wykryje co brakuje i zaproponuje komendę instalacji (`brew install` na macOS, `apt install` na Linux). Zapyta o zgodę przed uruchomieniem — nie instaluje samowolnie.

Pierwszy run instaluje `timecut@0.3.3` (pinned) — cached w `~/.cache/claude-design-to-video/timecut/`, ~150 MB, reuse forever.

---

## Instalacja

### Opcja A — manual copy (najłatwiejsza)

```bash
git clone https://github.com/Szewowsky/claude-design-to-video.git
cp -R claude-design-to-video/skill ~/.claude/skills/claude-design-to-video
```

Zrestartuj Claude Code. Skill pojawi się jako `/claude-design-to-video`.

### Opcja B — `.skill` package

Ściągnij `claude-design-to-video.skill` z [releases](https://github.com/Szewowsky/claude-design-to-video/releases), potem zainstaluj przez Claude Code skill installer.

---

## Quick start

### 1. Wyeksportuj animację z Claude Design
Share → **Download standalone HTML**. Dostaniesz folder:

```
My Animation/
├── index.html          # React + Babel entry
├── animations.jsx      # <Stage>, useTime, useTimeline engine
├── scenes/             # komponenty scen
└── assets/             # obrazki, fonty
```

### 2. Wywołaj skill

```
Ty: wyrenderuj mi animację z Downloads/My Animation
```

albo explicite:

```
Ty: /claude-design-to-video "~/Downloads/My Animation"
```

### 3. Czekaj
- **Preview** (15 s): ~1 min
- **Full 74 s @ 1080p30 fast:** ~10 min
- **Full 74 s @ 1080p30 HQ:** ~15 min
- **Full 74 s @ 4K30 HQ:** ~40–60 min

Output ląduje w `~/Downloads/<slug>-<WxH>-<fps>-<mode>.mp4`, auto-otwiera w domyślnym playerze (QuickTime na macOS, xdg-open na Linux).

---

## Flagi

| flaga | efekt | domyślnie |
|---|---|---|
| `--hq` | CRF 18, preset slow (YouTube source grade) | ✓ on |
| `--fast` | CRF 20, preset medium (iteracja, -30% czasu) | off |
| `--preview N` | renderuj tylko pierwsze N sekund (szybki podgląd) | off |
| `--4k` | 3840×2160 (domyślnie 1920×1080) | off |
| `--fps N` | frame rate | 30 |
| `--duration N` | override parsed duration (wymagane gdy bundle nie ma `DURATION = N`) | auto |
| `--no-patch` | pomiń auto-patch CSS (jeśli bundle już clean) | off |
| `--in-place` | patchuj oryginalny folder zamiast temp kopii | off (bezpieczniej) |
| `--output PATH` | override output location | `~/Downloads/...` |

---

## Auto-patche

Eksporty Claude Design mają CSS timing który łamie frame-by-frame capture (timecut kontroluje JS time, nie CSS `@keyframes` / `transition:`). Skill auto-wykrywa i przepisuje te wzorce przed renderem, potem pokazuje **raport zmian** żebyś widział co dokładnie zostało zmodyfikowane:

| Wzorzec | Fix |
|---|---|
| `<PlaybackBar>` scrubber w `<Stage>` | Ukryty przez URL flag `?render=1` |
| `@keyframes blink` + `animation: 'blink Ns infinite'` | JS `opacity` z `useTime()` i linear fade |
| `transition: opacity Nms cubic-bezier` na webcam | JS ease-out cubic interpolation |
| Webcam idle motion (`sway`, `breathe` przez `Math.sin`) | Wyłączone (subpixel jitter brzydki w pre-rendered video) |

Patche aplikują się do **kopii** bundle'a w temp dir domyślnie — oryginał nietknięty. Użyj `--in-place` żeby modyfikować oryginał. Użyj `--no-patch` jeśli bundle jest już clean albo patchowałeś ręcznie.

Nieznane wzorce → skipped silently (render idzie dalej, możesz ręcznie fixnąć). Nowe wzorce można dodać do listy patches w czasie.

---

## Znane ograniczenia

- **CSS transitions z dynamic layout** (np. `transition: flex 600ms cubic-bezier`) nie są auto-patched — flex interpolation wymaga znania wymiarów, zbyt ryzykowne. Jak trafisz — ręczny port na JS `interpolate()`.
- **Intel Mac** renderuje ~1.5–2× wolniej niż Apple Silicon.
- **Windows** — zobacz [Wspierane platformy](#wspierane-platformy).

---

## Troubleshooting

Zobacz tabelę **Tryby awarii** w [`skill/SKILL.md`](skill/SKILL.md) — pokrywa port conflicts, Chrome timeouts, crash mid-render, offline npm install, frame count mismatches, brak `DURATION`.

---

## Jak to działa pod spodem

```
1. Preflight       → sprawdź ffmpeg, Chrome, Node, Python3 + auto-install jak brak
2. Locate bundle   → arg albo autodetect w ~/Downloads/ (przez Python — portable)
3. Parse metadata  → DURATION, viewport z entry HTML
4. Setup workspace → temp copy (default) albo in-place, trap cleanup w obu modes
5. Auto-patch CSS  → 4 znane bugi fixowane silently, raport zmian na końcu
6. Install timecut → pinned @0.3.3, cached (~150 MB, one-time)
7. Serve bundle    → python3 http.server na random port, retry loop
8. Render          → timecut → PNG frames → ffmpeg → MP4
9. Cleanup (trap)  → kill HTTP server + render + child processes, rm -rf temp
10. Report + open  → ffprobe stats, frame-count validation, open w playerze
```

Pełny flow: [`skill/SKILL.md`](skill/SKILL.md).

---

## Autorzy / credits

- **[timecut](https://github.com/tungs/timecut)** — Steve Tung — silnik virtualized-time browser rendering
- **ffmpeg** — finalny encode
- **Anthropic** — Claude Design
- Skill: Robert Szewczyk ([@Szewowsky](https://github.com/Szewowsky))

---

## Licencja

MIT — zobacz [LICENSE](LICENSE).

---

## Contributing

Issues i PR mile widziane. Szczególnie szukam:
- Nowych CSS pattern auto-patches (jak Claude Design eksport ewoluuje)
- Linux distro compatibility reports (na których distro testowane?)
- Performance improvements (parallel render? canvas capture mode?)
- Native Windows branch (PowerShell equivalent)
