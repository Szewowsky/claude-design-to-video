# CLAUDE.md — instrukcje dla Claude Code

Ten plik kieruje Claude Code gdy user otwiera ten projekt. Zawiera context + policies które mają przyspieszyć onboarding usera skilla `/claude-design-to-video`.

---

## Co to jest

Publiczny skill dla Claude Code: **Claude Design HTML export → MP4 video** przez timecut (Puppeteer + wirtualny czas + ffmpeg). Autor: Robert Szewczyk ([@Szewowsky](https://github.com/Szewowsky)). Repo: https://github.com/Szewowsky/claude-design-to-video

Dwie grupy użytkowników otwierają ten projekt:

1. **End users** — klonują repo żeby zainstalować skill u siebie. Chcą `cp -R skill/ ~/.claude/skills/claude-design-to-video && zrestartować Claude Code`. Skill powinien zadziałać od strzału.
2. **Contributors** — chcą zmodyfikować skill (nowe patche CSS, Linux testing, Windows branch). Chcą tests + package skill.

Twoja rola: ogarnij co user próbuje zrobić, prowadź szybko do celu.

---

## Auto-install brakujących zależności (WAŻNE)

Gdy user uruchamia skill pierwszy raz, najczęstszy problem = brak zależności (ffmpeg, Node, Chrome, Python). **Zamiast wyrzucać error message — zaproponuj instalację.**

### Zasady:

1. **Zawsze wykryj OS** (`uname -s` → Darwin/Linux) przed propozycją komendy
2. **Pokaż dokładną komendę** którą zaraz uruchomisz (full shell command)
3. **CZEKAJ na zgodę usera** — nie odpalaj `sudo apt install` / `brew install` bez potwierdzenia
4. Po zgodzie — odpal, pokaż output, po sukcesie wróć do przerwanego taska

### Matryca instalacji:

| brak narzędzia | macOS (Homebrew) | Linux Debian/Ubuntu | Linux Arch |
|---|---|---|---|
| `ffmpeg` | `brew install ffmpeg` | `sudo apt install -y ffmpeg` | `sudo pacman -S ffmpeg` |
| `node` (≥18) | `brew install node` | `sudo apt install -y nodejs npm` | `sudo pacman -S nodejs npm` |
| `python3` (≥3.8) | wbudowane (od Monterey) | `sudo apt install -y python3` | `sudo pacman -S python` |
| Chrome | `brew install --cask google-chrome` | `wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb && sudo dpkg -i google-chrome-stable_current_amd64.deb` | AUR: `yay -S google-chrome` |
| Homebrew (macOS) | `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"` | — | — |

### Przykładowy dialog:

```
User: /claude-design-to-video ~/Downloads/Animacja\ YouTube

Claude: Sprawdzam zależności... Brak ffmpeg.

        Wykryłem macOS z Homebrew. Mogę zainstalować przez:
          brew install ffmpeg
        
        To zajmie ~30s. Odpalam?

User: tak

Claude: [odpala brew install ffmpeg, pokazuje output]
        ✓ ffmpeg zainstalowany (v8.0.1). Wracam do renderu...
```

**Anti-pattern:** "ffmpeg not found" + exit. Zrób o jedno pytanie więcej, user oszczędza 5 minut szukania instrukcji.

---

## Typowe use cases — jak reagować

### "Skopiowałem skill ale nie działa"

1. Sprawdź czy `skill/SKILL.md` jest w `~/.claude/skills/claude-design-to-video/SKILL.md` (nie w `skill/SKILL.md` tam!)
2. Sprawdź czy user zrestartował Claude Code (skille ładują się przy starcie)
3. Zapytaj o dokładny error — przejdź do [`skill/SKILL.md`](skill/SKILL.md) sekcja "Tryby awarii"

### "Chcę zmodyfikować skill (dodać patch, zmienić flagę)"

1. Edytuj `skill/SKILL.md`
2. Test lokalnie: skopiuj zmodyfikowany plik do `~/.claude/skills/claude-design-to-video/SKILL.md`, zrestartuj Claude Code, uruchom na testowym bundle
3. Zrobić commit → PR na GitHub jeśli chcesz podzielić się zmianą

### "Chcę zrobić `.skill` package do distribucji"

```bash
cd "/path/to/claude-design-to-video"
zip -r claude-design-to-video.skill skill/ -x "skill/.DS_Store" "*/__pycache__/*" "*.pyc"
```

Rezultat: `claude-design-to-video.skill` w root repo, gotowy do commit + GitHub Releases.

### "Mój bundle ma inne bugi niż te 4 patche"

Dodaj Patch E+ w `skill/SKILL.md` sekcja 4. Wzorzec:
1. Renderuj raz bez `--no-patch` — zobacz co się psuje
2. Zidentyfikuj regex wzorca (`grep -rn "problematic_pattern" scenes/`)
3. Napisz sed/JS fix
4. Dodaj do SKILL.md jako nowy Patch

---

## Struktura repo

```
claude-design-to-video/
├── CLAUDE.md                    # ten plik — instrukcje dla Claude Code
├── README.md                    # user-facing docs (polski)
├── LICENSE                      # MIT
├── .gitignore                   # ignoruje assets/ + media files
├── skill/
│   └── SKILL.md                 # główny plik skilla — pipeline + patche
├── claude-design-to-video.skill # zbudowany .skill (zip), install-ready
└── assets/                      # banner + media (gitignored, lokalne)
    └── promo-banner.png
```

---

## Kontekst projektu

- **Timecut** robi frame-by-frame render przez Puppeteer z wirtualnym czasem — override `requestAnimationFrame`, `setTimeout`, `Date.now`
- **Claude Design** eksportuje jako standalone HTML z React + Babel + JSX, ze specyficznym `<Stage>` silnikiem i `useTimeline` hookiem
- **4 znane CSS bugi** w eksporcie Claude Design — wszystkie łamią frame-by-frame capture i są auto-patched
- **Cache timecut** w `~/.cache/claude-design-to-video/timecut/` (~150MB) — reuse między renderami
- **Temp workspace** w `/tmp/cdv.XXXXXX/` — zawsze sprzątany przez `trap` bash

Szczegóły techniczne: [`skill/SKILL.md`](skill/SKILL.md).

---

## Co NIE robić

- **Nie commituj** `assets/` ani `*.mp4` (są w `.gitignore`)
- **Nie zmieniaj** `timecut@0.3.3` na `latest` bez testu (supply-chain risk)
- **Nie usuwaj** `trap cleanup` — cleanup temp/procesów musi działać nawet przy Ctrl+C
- **Nie dodawaj** Windows native branch dopóki nie ma community contribution — WSL2 rozwiązuje 99% przypadków
- **Nie odpalaj** `brew install` / `sudo apt install` bez explicit zgody usera

---

## Contact

Issues / PR: https://github.com/Szewowsky/claude-design-to-video/issues
Autor: Robert Szewczyk — [LinkedIn](https://www.linkedin.com/in/szewowsky) / [YouTube](https://www.youtube.com/@robertszewczyk)
