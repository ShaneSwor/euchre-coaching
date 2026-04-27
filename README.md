# Euchre Trainer

A standalone, single-page Euchre training game with live AI opponents and an automatic coach that explains better plays. No build tools, no frameworks — just `index.html` + `assets/`.

## Run locally

```bash
cd euchre
python3 -m http.server 8765
# then open http://localhost:8765/
```

(Any static server works — `npx serve`, VS Code Live Server, etc.)

## Project layout

```
euchre/
├── index.html               # the entire app (HTML + CSS + JS)
└── assets/
    ├── Card_Sprites.png                          # 24-card sprite sheet
    ├── Green_felt_tabletop.png                   # background
    └── Mackinac_Sunrise_PlayingCardImage_back.png # face-down card art
```

## How it works

You play the **South** seat. North is your AI partner; East and West are AI opponents. The coach watches your bidding and card play and pops up a tip whenever it spots a better move (over-trumping partner, leading low trump with a right bower, passing a strong hand, etc.).

Game ends at 10 points. Standard scoring: makers +1, sweep +2, loner sweep +4, defenders euchre callers +2.

## Hosting on AWS (S3 + CloudFront)

This is a fully static site — no server needed. Recommended path:

### 1. Push to GitHub
```bash
cd /Users/shane_swor/Documents/Development/euchre
git init
git add .
git commit -m "Initial Euchre trainer"
gh repo create euchre --public --source=. --push
```
(or create the repo in the GitHub UI and `git push`)

### 2. Quick S3 static-website hosting
```bash
BUCKET=your-euchre-site-name
aws s3 mb s3://$BUCKET
aws s3 website s3://$BUCKET --index-document index.html
aws s3 sync . s3://$BUCKET --exclude ".git/*" --exclude "README.md"
aws s3api put-bucket-policy --bucket $BUCKET --policy '{
  "Version":"2012-10-17",
  "Statement":[{"Effect":"Allow","Principal":"*","Action":"s3:GetObject","Resource":"arn:aws:s3:::'$BUCKET'/*"}]
}'
echo "http://$BUCKET.s3-website-$(aws configure get region).amazonaws.com"
```

### 3. Add CloudFront for HTTPS + caching (recommended)
- Create a CloudFront distribution with the S3 bucket as the origin (use the bucket REST endpoint, not the website endpoint, with Origin Access Control).
- Set the default root object to `index.html`.
- Point your custom domain via Route 53.

### 4. CI/CD via GitHub Actions (optional)
A simple workflow to auto-deploy on push to `main`:
```yaml
# .github/workflows/deploy.yml
name: Deploy
on: { push: { branches: [main] } }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/github-deploy
          aws-region: us-east-1
      - run: aws s3 sync . s3://YOUR_BUCKET --exclude ".git/*" --exclude "README.md" --exclude ".github/*"
      - run: aws cloudfront create-invalidation --distribution-id YOUR_DIST_ID --paths "/*"
```

## Architecture (one-file)

`index.html` contains:
- **CSS** — all styling, scoped via `:root` CSS variables for card sizes
- **DOM skeleton** — score bar, north/west/east/south player areas, table center, coach panel
- **JS classes**
  - `Card`, `Deck` — 24-card deck, left-bower aware via `effectiveSuit(trump)`
  - `EuchreRules` — pure rules: trick winner, legal plays, hand evaluation
  - `EuchreAI` — bid/discard/play heuristics
  - `EuchreCoach` — tip generators, called only on user (player 0) decisions
  - `EuchreGame` — state machine: `DEALING → BIDDING_R1 → BIDDING_R2 → DEALER_DISCARD → PLAYING → TRICK_END → HAND_SCORING → GAME_OVER`
  - `EuchreUI` — DOM rendering + AI scheduling via `setTimeout`

The coach evaluates **before** state advances so the hand context used to generate the tip matches what the user saw.

## Tweaking

- **Card size**: change `--card-w` in `:root`.
- **AI difficulty**: tweak thresholds in `EuchreAI.decideBidR1` / `decideBidR2`.
- **Coach strictness**: edit checks in `EuchreCoach.evalCardPlay` etc.
- **Sprite alignment**: `SPRITE.cols_x` / `SPRITE.rows_y` constants near the top of the script.

## Debugging

The bootstrap exposes the running game on `window.__euchre = { game, ui }` so you can poke at it from the browser console.
