# HANDOFF — Container load feasibility: config "Dao_20260610"

> Self-contained handoff. Paste this whole file into a new session to continue.
> App: `cointainer_calculator.html` → `index.html` (GitHub Pages: a17255.github.io/container-calculator/).
> Created: 2026-06-10. Deployed app commit at handoff: `8cd8e4b`.

---

## 1. Question
Can all packs in config **Dao_20260610** be loaded into one **40ft HC** container?
Report fill area, available area, and a packing guide.

## 2. Container — 40ft HC
| | value |
|---|---|
| Internal L × W × H | **1203 × 235 × 270 cm** |
| Volume | **76.33 m³** |
| Payload limit | 26,460 kg |

(App container key `40hc`. The saved cfg's cL/cW/cH = 5.90/2.35/2.39 m are ignored because the container is a preset, not "custom".)

## 3. Input packs (11 types — all `rect` except #7 `bag`; gap 0 cm; orientation z = upright)
| # | Name | L×W×H (cm) | Qty | Unit vol (m³) | Total vol (m³) |
|---|------|-----------|-----|---------------|----------------|
| 1 | (Pack 1, unnamed) | 45 × 40 × 26 | 134 | 0.04680 | 6.271 |
| 2 | xơ mướp | 60 × 40 × 35 | 127 | 0.08400 | 10.668 |
| 3 | Lá măng cầu1 | 34 × 20 × 12 | 150 | 0.00816 | 1.224 |
| 4 | Lá măng cầu2 | 27 × 26 × 17 | 240 | 0.01193 | 2.864 |
| 5 | Lá măng cầu3 | 50 × 40 × 27 | 104 | 0.05400 | 5.616 |
| 6 | Rong Sụn 1 | 40 × 30 × 30 | 175 | 0.03600 | 6.300 |
| 7 | Rong Sụn 2 (bag) | 85 × 40 × 15 | 400 | 0.05100 | 20.400 |
| 8 | Bột sake | 32 × 26 × 26 | 60 | 0.02163 | 1.298 |
| 9 | Bột hoa đậu biếc | 30 × 20 × 15 | 100 | 0.00900 | 0.900 |
| 10 | Lá Dứa | 30 × 20 × 20 | 100 | 0.01200 | 1.200 |
| 11 | Lá Khổ qua | 50 × 40 × 27 | 200 | 0.05400 | 10.800 |
| | **TOTAL** | | **1,790** | | **67.54** |

## 4. Result — feasibility
| Metric | Value |
|---|---|
| Total boxes | 1,790 |
| Total box volume (FILL needed) | **67.54 m³** |
| Container volume (AVAILABLE) | 76.33 m³ |
| Raw volumetric fill | **88.5 %** |
| Headroom on paper | 8.79 m³ |
| Realistic packing ceiling for *mixed* boxes | ~85–88 % ≈ **65–67 m³** |

**VERDICT: NO — all 1,790 boxes will not fit in one 40ft HC.**
At 88.5 % volumetric fill you are at/above the practical ceiling for mixed-size boxes
(real loads rarely beat ~85 %). Even a theoretically perfect packer is at its limit;
in practice expect ~3 m³ (≈ one pack-type) of **overflow**. Plan to either remove the
lowest-priority pack(s) or send the remainder in a second (partial) load.

> Note: the live tool currently reports only **71.6 % loaded with the last packs = 0**.
> That is a *packer bug* (greedy single-axis wastes the top/side spare and zeros the last
> packs), NOT the true limit. A fixed packer would reach ~85 %, but you'd still be ~full.

## 5. Packing guide (to get closest to full)
Floor = 1203 × 235 cm; stack height budget = 270 cm.
1. **Base layer** — floor the big rigid boxes full-width first: xơ mướp (35h), Lá Khổ qua (27h), Lá măng cầu3 (27h), Rong Sụn 1 (30h). These set strong vertical stacking columns.
2. **Uniform layers** — stack equal-height boxes together so each layer is flat: the 26–27h group (Pack 1, Bột sake, Lá3, Khổ qua) and the 20h group (Lá Dứa). Maximizes clean stacking to 270.
3. **Filler layers** — the flat bags Rong Sụn 2 (15h, flexible) and thin Lá măng cầu1 (12h) go on top and into height gaps.
4. **Small boxes last** — Bột hoa đậu biếc (30×20×15), Lá Dứa, Lá măng cầu2 plug edge/top gaps.
5. **Overflow** — set aside ~5–10 % (≈ one pack-type, e.g. part of the 400 Rong Sụn 2 bags or the 200 Lá Khổ qua) for a second load.

## 6. Reload the config in the app
Open the app and paste this URL (decodes the full config):

```
https://a17255.github.io/container-calculator/#cfg=eyJjb250YWluZXIiOiI0MGhjIiwiY0wiOiI1LjkwIiwiY1ciOiIyLjM1IiwiY0giOiIyLjM5IiwicGtnVHlwZSI6InJlY3QiLCJvcmllbnRhdGlvbiI6InoiLCJvdXRwdXRNb2RlIjoiY3VzdG9tIiwicF9kIjoiMTAiLCJwX2NoIjoiMTIiLCJwX3MiOiI0MCIsInBfc2giOiI2MCIsInBfbCI6IjQ1IiwicF93IjoiNDAiLCJwX3JoIjoiMjYiLCJwX3NkIjoiMzAiLCJwX3dkbCI6IjYwIiwicF93ZHciOiI0MCIsInBfd2RoIjoiMzAiLCJwX2JnbCI6IjE2MCIsInBfYmd3IjoiMzUiLCJwX2JnaCI6IjM1IiwicF9nYXAiOiIwIiwicF93ZWlnaHQiOiIiLCJwa2dfY291bnQiOiIxMzQiLCJ2aWV3QXoiOjEuNDcwMjEyODU2MjE3ODY2NSwidmlld0VsIjowLjUyMzU5ODc3NTU5ODI5ODgsInNwbGl0QXhpcyI6IkwiLCJ2aWV3TW9kZSI6ImNvbXBhY3QiLCJwYWNrc0NmZyI6W3siaWQiOjEsIm5hbWUiOiIiLCJzaGFwZSI6InJlY3QiLCJkaW1zIjp7ImwiOjQ1LCJ3Ijo0MCwiaCI6MjZ9LCJnYXAiOjAsIndlaWdodCI6MCwiY291bnQiOjEzNH0seyJpZCI6MiwibmFtZSI6InjGoSBtxrDhu5twIiwic2hhcGUiOiJyZWN0IiwiZGltcyI6eyJsIjo2MCwidyI6NDAsImgiOjM1fSwiZ2FwIjowLCJ3ZWlnaHQiOjAsImNvdW50IjoxMjd9LHsiaWQiOjMsIm5hbWUiOiJMw6EgbcOjbmcgY+G6p3UxIiwic2hhcGUiOiJyZWN0IiwiZGltcyI6eyJsIjozNCwidyI6MjAsImgiOjEyfSwiZ2FwIjowLCJ3ZWlnaHQiOjAsImNvdW50IjoxNTB9LHsiaWQiOjQsIm5hbWUiOiJMw6EgbcOjbmcgY+G6p3UyIiwic2hhcGUiOiJyZWN0IiwiZGltcyI6eyJsIjoyNywidyI6MjYsImgiOjE3fSwiZ2FwIjowLCJ3ZWlnaHQiOjAsImNvdW50IjoyNDB9LHsiaWQiOjUsIm5hbWUiOiJMw6EgbcOjbmcgY+G6p3UzIiwic2hhcGUiOiJyZWN0IiwiZGltcyI6eyJsIjo1MCwidyI6NDAsImgiOjI3fSwiZ2FwIjowLCJ3ZWlnaHQiOjAsImNvdW50IjoxMDR9LHsiaWQiOjYsIm5hbWUiOiJSb25nIFPhu6VuIDEiLCJzaGFwZSI6InJlY3QiLCJkaW1zIjp7ImwiOjQwLCJ3IjozMCwiaCI6MzB9LCJnYXAiOjAsIndlaWdodCI6MCwiY291bnQiOjE3NX0seyJpZCI6NywibmFtZSI6IlJvbmcgU+G7pW4gMiIsInNoYXBlIjoiYmFnIiwiZGltcyI6eyJsIjo4NSwidyI6NDAsImgiOjE1fSwiZ2FwIjowLCJ3ZWlnaHQiOjAsImNvdW50Ijo0MDB9LHsiaWQiOjgsIm5hbWUiOiJC4buZdCBzYWtlIiwic2hhcGUiOiJyZWN0IiwiZGltcyI6eyJsIjozMiwidyI6MjYsImgiOjI2fSwiZ2FwIjowLCJ3ZWlnaHQiOjAsImNvdW50Ijo2MH0seyJpZCI6OSwibmFtZSI6IkLhu5l0IGhvYSDEkeG6rXUgYmnhur9jIiwic2hhcGUiOiJyZWN0IiwiZGltcyI6eyJsIjozMCwidyI6MjAsImgiOjE1fSwiZ2FwIjowLCJ3ZWlnaHQiOjAsImNvdW50IjoxMDB9LHsiaWQiOjEwLCJuYW1lIjoiTMOhIEThu6lhIiwic2hhcGUiOiJyZWN0IiwiZGltcyI6eyJsIjozMCwidyI6MjAsImgiOjIwfSwiZ2FwIjowLCJ3ZWlnaHQiOjAsImNvdW50IjoxMDB9LHsiaWQiOjExLCJuYW1lIjoiTMOhIEto4buVIHF1YSIsInNoYXBlIjoicmVjdCIsImRpbXMiOnsibCI6NTAsInciOjQwLCJoIjoyN30sImdhcCI6MCwid2VpZ2h0IjowLCJjb3VudCI6MjAwfV19
```

## 7. Pending tool work (next session)
1. **Config save/load → files only** (remove all config web storage; keep license). Save→`.json` download, Load→file upload.
2. **Fix packer** (guillotine free-space region packer, fills top/side spare → ~85 % not 71.6 %, no false zeros) + **per-pack include/exclude checkbox** (user chooses what to leave out) + **remaining-space report** (free m³ + what could still fit).
3. Full spec in the plan file: `C:\Users\A17255\.claude\plans\load-memory-related-this-graceful-pudding.md`.
