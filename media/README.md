# Media

Drop the recorded walkthrough and screenshots here.

Required by the top-level README:
- **`demo.gif`** — the 60–90s walkthrough (referenced at the top of `README.md`).

Recommended stills (referenced nowhere yet — add them to the README if you want):
- `concept-map.png` — the home concept map with mastery states
- `tutoring.png` — a Prime→Teach→Assess turn in progress
- `dev-trace.png` — the dev-trace panel mid-turn
- `lecturer-insights.png` — the insight suite cards

## Recording the walkthrough (exact script)

Record at ~1280×800, browser only (hide bookmarks bar). Keep it under 90s.

1. **Log in** as the demo student → land on the **concept map**. Pause 1s so the
   map (nodes + prerequisite edges + mastery colors) is readable.
2. **Start a study session** → show the planned session (retrieve / deep / new).
3. Enter a concept → show one **Prime → Teach → Assess** turn. Type a short
   answer, submit, let the **evaluation + feedback** appear.
4. Show a node's **mastery state change** on the map after the turn.
5. **Toggle the dev-trace panel** → scroll the live decision events
   (session_planned, assess, evaluated, mastery_updated, review_scheduled).
   *This is the money shot — linger here ~5s.*
6. Log out → log in as the **lecturer** → show the class board, then the
   **insight suite** cards (misconceptions / next-lecture brief). Linger ~3s.

## Making the GIF

```bash
# macOS screen recording: Cmd+Shift+5 → record selection → save .mov
# then convert (needs ffmpeg + gifsicle):
ffmpeg -i walkthrough.mov -vf "fps=12,scale=800:-1:flags=lanczos" -f gif - \
  | gifsicle --optimize=3 --colors 200 > demo.gif
```

Keep `demo.gif` under ~10 MB so GitHub renders it inline. If it's too big,
trim length or drop fps to 10.
