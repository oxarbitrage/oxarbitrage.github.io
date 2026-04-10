# Personal Website Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and deploy a minimal personal website for oxarbitrage (cypherpunk/hacker/Rust dev) with Home, About, Projects, and Blog pages using Zola SSG on GitHub Pages.

**Architecture:** Zola renders Markdown + Tera templates into static HTML. A GitHub Actions workflow builds and pushes the output to `gh-pages` on every push to `main`. All styling lives in a single `style.css` file with a Terminal dark theme (black bg, green accent, monospace).

**Tech Stack:** Zola (Rust SSG), Tera templates, plain CSS, GitHub Actions (`shalzz/zola-deploy-action@v0.13.0`)

---

### Task 1: Install Zola and scaffold project

**Files:**
- Create: `config.toml`
- Create: `content/_index.md`
- Create: `templates/base.html`
- Create: `templates/index.html`

- [ ] **Step 1: Install Zola**

```bash
# Linux (snap)
snap install zola --edge

# Or via cargo
cargo install zola

# Or download binary from https://github.com/getzola/zola/releases
# e.g. v0.19.2
wget https://github.com/getzola/zola/releases/download/v0.19.2/zola-v0.19.2-x86_64-unknown-linux-gnu.tar.gz
tar xzf zola-v0.19.2-x86_64-unknown-linux-gnu.tar.gz
sudo mv zola /usr/local/bin/
```

Verify: `zola --version` → should print `zola 0.19.2`

- [ ] **Step 2: Initialize Zola in the repo**

```bash
cd /home/alfredo/oxarbitrage-web
zola init .
# Answer prompts:
# URL: https://oxarbitrage.github.io
# Sass: n
# Syntax highlighting: y
# Search index: n
```

This creates `config.toml`, `content/`, `templates/`, `static/`, `themes/`.

- [ ] **Step 3: Update `config.toml`**

Replace generated content with:

```toml
base_url = "https://oxarbitrage.github.io"
title = "oxarbitrage"
description = "cypherpunk. hacker. builder."
compile_sass = false
generate_feed = false
taxonomies = []

[markdown]
highlight_code = true
highlight_theme = "base16-ocean-dark"

[extra]
author = "oxarbitrage"
github = "https://github.com/oxarbitrage"
```

- [ ] **Step 4: Download logo**

```bash
curl -L "https://github.com/oxarbitrage.png?size=200" -o static/logo.png
```

- [ ] **Step 5: Create `content/_index.md`**

```markdown
+++
title = "oxarbitrage"
template = "index.html"
+++
```

- [ ] **Step 6: Create minimal `templates/base.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{% block title %}{{ config.title }}{% endblock title %}</title>
  <link rel="stylesheet" href="{{ get_url(path='css/style.css') }}">
</head>
<body>
  <header>
    <a href="{{ config.base_url }}" class="logo-link">
      <img src="{{ get_url(path='logo.png') }}" alt="oxarbitrage logo" class="logo">
    </a>
    <nav>
      <a href="{{ config.base_url }}">home</a>
      <a href="{{ get_url(path='about') }}">about</a>
      <a href="{{ get_url(path='projects') }}">projects</a>
      <a href="{{ get_url(path='blog') }}">blog</a>
    </nav>
  </header>
  <main>
    {% block content %}{% endblock content %}
  </main>
  <footer>
    <a href="{{ config.extra.github }}">github</a>
  </footer>
</body>
</html>
```

- [ ] **Step 7: Create `templates/index.html`**

```html
{% extends "base.html" %}

{% block title %}{{ config.title }}{% endblock title %}

{% block content %}
<section class="home">
  <p class="tagline">cypherpunk. hacker. builder.</p>
  <p>Rust developer. Working on privacy, cryptography, and financial systems.</p>
  <p>
    <a href="{{ get_url(path='about') }}">about</a> ·
    <a href="{{ get_url(path='projects') }}">projects</a> ·
    <a href="{{ get_url(path='blog') }}">blog</a>
  </p>
</section>
{% endblock content %}
```

- [ ] **Step 8: Verify build**

```bash
cd /home/alfredo/oxarbitrage-web
zola build
```

Expected: `Building site... -> Site built in Xms` with no errors. `public/` directory created.

- [ ] **Step 9: Commit**

```bash
git add config.toml content/ templates/ static/
git commit -m "feat: scaffold Zola site with home template and logo"
```

---

### Task 2: Add CSS (Terminal dark theme)

**Files:**
- Create: `static/css/style.css`

- [ ] **Step 1: Create `static/css/style.css`**

```css
/* Terminal dark theme */
:root {
  --bg: #0d0d0d;
  --bg-secondary: #111;
  --fg: #c8c8c8;
  --accent: #00ff41;
  --accent-dim: #00b32d;
  --muted: #555;
  --font-mono: 'Courier New', Courier, monospace;
  --font-sans: 'Courier New', Courier, monospace;
  --max-width: 720px;
}

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

html { font-size: 16px; }

body {
  background: var(--bg);
  color: var(--fg);
  font-family: var(--font-mono);
  line-height: 1.7;
  padding: 2rem 1rem;
}

header {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  max-width: var(--max-width);
  margin: 0 auto 3rem;
  border-bottom: 1px solid var(--muted);
  padding-bottom: 1rem;
}

.logo {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  border: 1px solid var(--accent);
}

.logo-link { flex-shrink: 0; }

nav { display: flex; gap: 1.5rem; }

nav a, footer a {
  color: var(--accent);
  text-decoration: none;
  font-size: 0.9rem;
}

nav a:hover, footer a:hover { color: var(--fg); }

main {
  max-width: var(--max-width);
  margin: 0 auto;
}

footer {
  max-width: var(--max-width);
  margin: 4rem auto 0;
  border-top: 1px solid var(--muted);
  padding-top: 1rem;
  font-size: 0.85rem;
  color: var(--muted);
}

/* Typography */
h1, h2, h3 {
  color: var(--accent);
  font-weight: normal;
  margin-bottom: 1rem;
}

h1 { font-size: 1.4rem; }
h2 { font-size: 1.15rem; margin-top: 2rem; }
h3 { font-size: 1rem; margin-top: 1.5rem; }

p { margin-bottom: 1rem; }

a { color: var(--accent); }
a:hover { color: var(--fg); }

code {
  background: var(--bg-secondary);
  color: var(--accent-dim);
  padding: 0.1em 0.3em;
  border-radius: 2px;
  font-size: 0.9em;
}

pre {
  background: var(--bg-secondary);
  border-left: 2px solid var(--accent);
  padding: 1rem;
  overflow-x: auto;
  margin-bottom: 1.5rem;
}

pre code { background: none; padding: 0; }

/* Home */
.tagline {
  font-size: 1.2rem;
  color: var(--accent);
  margin-bottom: 1rem;
}

/* Blog */
.post-list { list-style: none; }

.post-list li {
  display: flex;
  gap: 1.5rem;
  margin-bottom: 0.75rem;
  align-items: baseline;
}

.post-date { color: var(--muted); font-size: 0.85rem; flex-shrink: 0; }

/* Projects */
.project-list { list-style: none; }

.project-list li {
  margin-bottom: 1.5rem;
  border-left: 2px solid var(--muted);
  padding-left: 1rem;
}

.project-list li:hover { border-left-color: var(--accent); }

.project-name { color: var(--accent); font-weight: bold; }

.project-lang {
  font-size: 0.8rem;
  color: var(--muted);
  margin-top: 0.25rem;
}
```

- [ ] **Step 2: Verify build and preview**

```bash
zola serve
```

Open http://127.0.0.1:1111 — should see dark terminal-style home page with green logo border and nav.

- [ ] **Step 3: Commit**

```bash
git add static/css/style.css
git commit -m "feat: add terminal dark CSS theme"
```

---

### Task 3: About and Projects pages

**Files:**
- Create: `content/about.md`
- Create: `content/projects.md`
- Create: `templates/page.html`

- [ ] **Step 1: Create `templates/page.html`**

```html
{% extends "base.html" %}

{% block title %}{{ page.title }} | {{ config.title }}{% endblock title %}

{% block content %}
<article>
  <h1>{{ page.title }}</h1>
  {{ page.content | safe }}
</article>
{% endblock content %}
```

- [ ] **Step 2: Create `content/about.md`**

```markdown
+++
title = "about"
template = "page.html"
+++

cypherpunk. hacker. anarchist.

I'm a Rust developer focused on privacy, cryptography, and financial systems. I believe in open protocols, individual sovereignty, and building tools that resist surveillance and coercion.

- **GitHub:** [oxarbitrage](https://github.com/oxarbitrage)
- **Interests:** zero-knowledge proofs, formal verification, decentralized finance, privacy tech

> *"Privacy is necessary for an open society in the electronic age."* — Eric Hughes, A Cypherpunk's Manifesto
```

- [ ] **Step 3: Create `content/projects.md`**

```markdown
+++
title = "projects"
template = "page.html"
+++

<ul class="project-list">
  <li>
    <div><a class="project-name" href="https://github.com/oxarbitrage/formal-market-mechanisms">formal-market-mechanisms</a></div>
    <div>Formal verification of batch auction market mechanisms. Proving fairness properties of trade execution using mathematical proof techniques.</div>
    <div class="project-lang">Rust · Formal Methods</div>
  </li>
  <li>
    <div><a class="project-name" href="https://github.com/oxarbitrage">→ more on GitHub</a></div>
    <div>Full project list on GitHub.</div>
  </li>
</ul>
```

- [ ] **Step 4: Build and verify both pages render**

```bash
zola build 2>&1 | grep -E "error|warning|Built"
```

Expected: `Site built in Xms` with no errors. Check `public/about/index.html` and `public/projects/index.html` exist.

- [ ] **Step 5: Commit**

```bash
git add content/about.md content/projects.md templates/page.html
git commit -m "feat: add about and projects pages"
```

---

### Task 4: Blog section and first post

**Files:**
- Create: `content/blog/_index.md`
- Create: `content/blog/why-the-way-you-trade-matters.md`
- Create: `templates/section.html`

- [ ] **Step 1: Create `templates/section.html`** (blog index listing)

```html
{% extends "base.html" %}

{% block title %}{{ section.title }} | {{ config.title }}{% endblock title %}

{% block content %}
<article>
  <h1>{{ section.title }}</h1>
  <ul class="post-list">
    {% for page in section.pages %}
    <li>
      <span class="post-date">{{ page.date | date(format="%Y-%m-%d") }}</span>
      <a href="{{ page.permalink }}">{{ page.title }}</a>
    </li>
    {% endfor %}
  </ul>
</article>
{% endblock content %}
```

- [ ] **Step 2: Create `content/blog/_index.md`**

```markdown
+++
title = "blog"
template = "section.html"
sort_by = "date"
+++
```

- [ ] **Step 3: Create `content/blog/why-the-way-you-trade-matters.md`**

Paste the full blog post content. Front matter:

```markdown
+++
title = "Why the way you trade matters — and how privacy makes it fair"
date = 2024-01-01
template = "page.html"
description = "On batch auctions, front-running, MEV, and why privacy in trading is a structural fairness requirement."
+++

When you place a trade on most exchanges, you're walking into a game that's rigged by design. Not because anyone is cheating — because the rules themselves create winners and losers.

On a traditional order book (the kind NYSE, Binance, and Coinbase use), the first order in gets the best price. That sounds fair until you realize that "first" means whoever has the fastest connection, the closest server, the most expensive infrastructure. Speed becomes a tax on everyone else. Billions are spent each year on shaving microseconds off trade execution — not to build better products, but to see your order before you do and trade ahead of it. That's front-running, and it's structural, not a bug.

On an AMM like Uniswap, there's no order book — a mathematical formula sets the price. Anyone can trade, anytime. That sounds fair too, until someone sees your pending transaction, buys before you to push the price up, lets your trade execute at the worse price, then sells. You lost money. They gained it. That's a sandwich attack, and it happens thousands of times a day on Ethereum. It's estimated that MEV extraction — profits taken from ordinary users through transaction ordering — totals hundreds of millions of dollars per year.

## What if order didn't matter?

There's a different design: batch auctions. Instead of matching orders one by one as they arrive, orders are collected over a window and then cleared all at once at a single price. Everyone in the batch gets the same price. It doesn't matter if your order arrived first or last, from a fast server or a slow one.

We formally verified this — using the same mathematical proof techniques used in aerospace and chip design. Not simulated, not argued, *proved*: in a batch auction, the clearing price is the same regardless of the order in which trades are submitted. There is no spread to capture, no timing advantage to exploit, no sandwich to construct. The game is fair because the rules make exploitation structurally impossible.

## Privacy is not about having something to hide

Now consider: even in a fair batch auction, everyone can see what you're trading. They can see which asset pair you're interested in, how active that market is, and adjust their strategy accordingly. If a fund is rebalancing its portfolio, the market sees it. If a DAO votes to diversify its treasury, front-runners position themselves before the diversification even begins. If an individual in a difficult jurisdiction is moving assets, the pair itself reveals their intent.

Privacy in this context is not about concealing wrongdoing. It's about removing the information asymmetry that allows the powerful to exploit the ordinary. When your order contents are sealed and even the asset pair is hidden, no one can target you — because no one knows what you're doing. Your trade is indistinguishable from everyone else's. You are just another opaque commitment in a batch.

We prove this too — that a private batch auction with sealed bids satisfies the same fairness properties as a public one, while additionally guaranteeing that no participant can be selectively targeted based on what they're trading.
```

> **Note:** The full blog post source is at https://github.com/oxarbitrage/formal-market-mechanisms/blob/main/blog.md — fetch the complete content and paste it in after the front matter block above. Use `curl https://raw.githubusercontent.com/oxarbitrage/formal-market-mechanisms/main/blog.md` to get it.

- [ ] **Step 4: Build and verify**

```bash
zola build 2>&1 | grep -E "error|warning|Built"
```

Check `public/blog/index.html` and `public/blog/why-the-way-you-trade-matters/index.html` exist.

- [ ] **Step 5: Serve and check blog visually**

```bash
zola serve
```

Open http://127.0.0.1:1111/blog — should list the post. Click through to the post page.

- [ ] **Step 6: Commit**

```bash
git add content/blog/ templates/section.html
git commit -m "feat: add blog section and first post"
```

---

### Task 5: GitHub Actions deploy workflow

**Files:**
- Create: `.github/workflows/deploy.yml`

- [ ] **Step 1: Create `.github/workflows/deploy.yml`**

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build and Deploy
        uses: shalzz/zola-deploy-action@v0.19.2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 2: Verify the workflow file is valid YAML**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/deploy.yml'))" && echo "Valid YAML"
```

Expected: `Valid YAML`

- [ ] **Step 3: Commit and push**

```bash
git add .github/workflows/deploy.yml
git commit -m "feat: add GitHub Actions deploy workflow for GitHub Pages"
git push -u origin main
```

- [ ] **Step 4: Enable GitHub Pages**

In the GitHub repo settings:
1. Go to **Settings → Pages**
2. Set **Source** to `Deploy from a branch`
3. Set **Branch** to `gh-pages` / `/ (root)`
4. Save

- [ ] **Step 5: Verify deployment**

Check GitHub Actions tab — the workflow should run and succeed. After ~1 minute, visit https://oxarbitrage.github.io — the site should be live.

---

### Task 6: Final polish and `.gitignore`

**Files:**
- Create: `.gitignore`

- [ ] **Step 1: Create `.gitignore`**

```
public/
.sass-cache/
```

- [ ] **Step 2: Final build check**

```bash
zola build
```

Expected: clean build, no warnings or errors.

- [ ] **Step 3: Commit**

```bash
git add .gitignore
git commit -m "chore: add .gitignore, exclude Zola build output"
```
