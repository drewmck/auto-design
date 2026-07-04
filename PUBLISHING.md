# Publishing `auto-design` as a public plugin

Self-serve steps to release this repo as a public Claude Code marketplace + plugin.

## Layout

This repo **is** the plugin — no nested subfolder:

```
auto-design-public/
├── .claude-plugin/
│   ├── plugin.json        # plugin manifest
│   └── marketplace.json   # marketplace catalog (source: ".", points at this same repo)
├── skills/auto-design/SKILL.md
├── README.md
├── LICENSE
└── PUBLISHING.md          # this file
```

`marketplace.json`'s single plugin entry uses `"source": "."`, the documented pattern for a
repo that is simultaneously the marketplace and the plugin it lists.

`auto-design-post.md` is an unrelated blog draft that lives in this working directory for
convenience only. It is listed in `.gitignore` and must **not** be committed to this repo.
`Output.svg` is committed and used as the README header image.

## What was already done

- **Bring-your-own design system:** no hardcoded design-system references; the skill asks for
  the user's Figma library key on first run and stores it in `~/.claude/auto-design/config.json`.
- **Converge-only:** the skill builds and refines one design direction per feature (no
  alternative-versions divergence path).
- **Sanitized:** `plugin.json` author email, `README.md` install instructions, and app-specific
  examples in `SKILL.md`.
- `LICENSE` (MIT) and `.claude-plugin/marketplace.json` added.

## Steps

### 1. Verify there are no leftover internal references
```bash
cd ~/workspace/auto-design-public
grep -rniE "continuum|pagerduty|lk-c8b1|ux-product-design" --exclude-dir=.git .
```
Expect **zero hits** against tracked files (the grep will still match `auto-design-post.md` if
present — that's fine, it's gitignored and won't be committed).

### 2. Validate the plugin/marketplace manifests
```bash
claude plugin validate .
```
Fix any schema errors it reports before continuing.

### 3. Commit locally
```bash
git init
git add .
git status   # confirm auto-design-post.md and Output.svg are NOT staged
git commit -m "auto-design: public release (bring-your-own design system, converge-only)"
```

### 4. Create the public repo and push
```bash
gh repo create drewmck/auto-design --public --source=. --remote=origin --push
```
Or create it in the GitHub UI, then:
```bash
git remote add origin git@github.com:drewmck/auto-design.git
git push -u origin main
```

### 5. Test the public install
In a Claude Code session:
```
/plugin marketplace add drewmck/auto-design
/plugin install auto-design@auto-design
```
Then run the skill twice to exercise the config round-trip:
```
/auto-design:auto-design <a-figma-section-url>
```
- **First run:** lists libraries via `get_libraries`, prompts for design system name + library key,
  writes `~/.claude/auto-design/config.json`.
- **Second run:** reads the config silently and builds one converged design per feature.
- **Reconfigure test:** delete `~/.claude/auto-design/config.json` (or say "reconfigure") and
  confirm it prompts again.

---

## Optional: also update the internal version

The internal copy at PagerDuty (`internal-claude-code-marketplace`, team `ux-product-design`) has
not received these changes. If you want to bring it up to date too:

1. Apply the same three file changes (bring-your-own design system, converge-only, sanitized
   copy) under `plugins/teams/ux-product-design/auto-design/` in that repo.
2. Fix that plugin's entry in the root `.claude-plugin/marketplace.json` — it still says
   `"Creates Continuum-based…"` and has `continuum` / `UX Product Design` keywords.
3. Commit and open a PR (that repo has CODEOWNERS + pre-commit/CI, so it goes through review).
