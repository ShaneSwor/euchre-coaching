# Euchre Coaching

A standalone, single-page Euchre training game with live AI opponents and an automatic coach that explains better plays. No build tools, no frameworks — just `index.html` and a few static assets.

🎴 **Live site:** https://euchre-coaching-d0791.web.app

## About this game

Euchre is a four-player, partner-based trick-taking card game popular across the Midwest, Ontario, and northern parts of the U.S. It's fast, strategic, and full of nuance — which is exactly why it's a great game to learn from a coach.

This site is built for that. You play the **South** seat with North as your partner against East and West (all three are AI). After every bid and every card you play, a coach evaluates whether your move was sound. If it spots a clearly better option — passing a strong hand, leading non-trump when you hold both bowers, over-trumping a partner who was already winning the trick — the coach panel slides up with a short, plain-English explanation of the better play. If you played well, it stays quiet.

The four strategy areas are all covered:
- **Ordering up trump** — knowing when your hand is strong enough to call
- **Going alone** — recognizing the loaded hands worth a 4-point loner
- **Card play and trick strategy** — leading bowers, trumping in, throwing off low
- **Defense** — leading trump against an opponent loner, refusing to over-trump your partner

A hand-strength meter appears during bidding so you can see exactly *why* a hand is or isn't strong enough to order up. Cards earn weighted points (right bower +4, left bower +3, other trump +1, off-suit ace +0.5), and the meter shows where you sit relative to standard call and loner thresholds.

The game runs to 10 points with full Euchre scoring: +1 for making it, +2 for a sweep or a euchre, +4 for a loner sweep.

## Run locally

```bash
git clone https://github.com/ShaneSwor/euchre-coaching.git
cd euchre-coaching
python3 -m http.server 8765
# open http://localhost:8765/
```

Any static server works — `npx serve`, VS Code Live Server, etc.

## Project layout

```
euchre-coaching/
├── index.html                                     # the entire app (HTML + CSS + JS)
├── firebase.json, .firebaserc                     # Firebase Hosting config
└── assets/
    ├── Green_felt_tabletop.png                    # page background
    └── Mackinac_Sunrise_PlayingCardImage_back.png # face-down card art
```

Card faces are rendered in pure CSS (rank/suit corners + center symbol), so no sprite sheet is needed.

## Deployment (Firebase Hosting / GCP)

The live site is hosted on Firebase Hosting (a Google Cloud product). Firebase gives static sites free HTTPS, free CDN, and one-command deploys — perfect for a project like this.

```bash
# One-time setup
curl -sL https://firebase.tools | bash    # install the Firebase CLI
firebase login                            # authenticate

# Deploy after edits
firebase deploy --only hosting
```

Free-tier (Spark plan) limits: 10 GB storage, ~10 GB/month bandwidth — plenty for a personal project.

## Architecture (one-file)

`index.html` contains:
- **CSS** — all styling, including CSS-rendered card faces, scoped via `:root` variables for sizing
- **DOM scaffold** — score bar, four player areas (N/E/S/W), table center, coach panel
- **JS classes**
  - `Card`, `Deck` — 24-card euchre deck, left-bower aware via `Card.effectiveSuit(trump)`
  - `EuchreRules` — pure rules: trick winner, legal plays, hand-strength evaluation
  - `EuchreAI` — bid / discard / play heuristics for the three AI seats
  - `EuchreCoach` — tip generators, called only on user (player 0) decisions
  - `EuchreGame` — state machine: `DEALING → BIDDING_R1 → BIDDING_R2 → DEALER_DISCARD → PLAYING → TRICK_END → HAND_SCORING → GAME_OVER`
  - `EuchreUI` — DOM rendering + AI scheduling via `setTimeout`

The coach evaluates **before** state advances inside `EuchreGame.playCard()` so the hand context used to generate the tip matches what the user saw at decision time.

## Tweaking

- **Card size**: change `--card-w` and `--card-h` in `:root`
- **AI difficulty**: adjust thresholds in `EuchreAI.decideBidR1` / `decideBidR2` and the heuristics in `EuchreAI.choosePlay`
- **Coach strictness**: edit individual checks in `EuchreCoach.evalCardPlay`, `evalBidR1`, `evalBidR2`
- **Strength scoring**: `EuchreRules.evaluateHandForTrump` — both the AI and coach use this single function

## Debugging

The bootstrap exposes the running game on `window.__euchre = { game, ui }` so you can poke at it from the browser console:

```js
__euchre.game.scores                      // current scores
__euchre.game.hands[0]                    // your hand
__euchre.ui._showCoach({                  // force a sample coach tip
  title: 'Coach test',
  message: 'Panel works.',
  severity: 'info'
})
```

## License

Personal / educational project. No formal license — feel free to read, learn, and fork for your own use.
