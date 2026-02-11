# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

CyberDiagram is a landing page for an open-source AI-powered cloud penetration testing platform targeting SMEs. The entire site is a single self-contained HTML file (`index.html`) with embedded CSS and JavaScript — no build process, no frameworks, no dependencies beyond Google Fonts (Inter).

**Domain:** cyberdiagram.com (GitHub Pages via CNAME)
**GitHub repo:** https://github.com/flashoop/mvp
**Contact:** flashoop@gmail.com

## Development

```bash
python3 -m http.server 8000
# Visit http://localhost:8000
```

No build step, no linting, no tests. Just static HTML.

## Architecture

`index.html` (~1430 lines) contains everything: CSS (~1010 lines), HTML structure, and JavaScript. The page sections in order:

1. **Nav** — Fixed header with Home/Product/About tabs + GitHub star link
2. **Hero** — Open source badge, value prop, contact info, GitHub CTA
3. **Screenshot** — Product screenshot (`Screeshot.png` — note the typo in filename)
4. **Problem** — Three-card grid: SME security gaps
5. **Solution** — Three-feature showcase (id="product")
6. **How It Works** — Four-step process: Connect → Scan → Visualize → Remediate
7. **Credibility** — Three founder profiles (id="about"): Leo Zhao, Claude Code, Gemini 3
8. **Final CTA** — Open source contribution call, GitHub + email links
9. **Footer** — Contact, resources, community links

### Design System

**Dark theme** with gradient accents:
- Background: `#0f1117` (page), `rgba(15, 17, 23, 0.95)` (nav)
- Text: `#E5E7EB` (primary), `#8b90a5` (secondary)
- Accent gradient: `#6c7ee1` → `#4fc3f7` (logo, highlights, borders)
- Cards: `#161b22` background, `#1e293b` borders
- Font: Inter from Google Fonts

**Responsive breakpoint:** 768px (single mobile breakpoint)

### JavaScript (~50 lines at bottom of file)

Three behaviors, all vanilla JS:
1. Navbar shadow on scroll (past 50px)
2. Smooth scroll for anchor links
3. Active nav link highlighting based on scroll position

## Key Customization Points

- **CTAs** link to `https://github.com/flashoop/mvp` — update if repo moves
- **LinkedIn URL:** `https://www.linkedin.com/in/zhaohongri/`
- **Founder image:** `me.jpeg` (other founders use inline SVG avatars)
- **Screenshot:** `Screeshot.png` (misspelled filename)
- **Footer links:** Several are still `href="#"` placeholders (Blog, Roadmap, API Docs, Privacy, Terms)

## File Notes

- `ARCHITECTURE.md`, `CODE_GUIDE.md`, `FEATURES.md`, `PROJECT_STRUCTURE.md`, `README.md` — These are documentation for a separate project (PentestGPT), not this landing page
- `development_roadmap.md`, `marketing_materials.md`, `quick_pitch_materials.md` — Source content/copy materials
- Image files: `attack.png`, `cloud-architecture.png`, `cloud-attach-path.png`, `cloud-business-architecture.png` — Reference diagrams, not currently used in index.html

## Brand Voice

- Professional but accessible — not overly technical
- Open-source community framing — "contribute", "star", "join"
- Problem-focused — emphasizes pain points before solution
- Founder credibility via certifications (OSEP, OSCP, GCPN, CISSP, CCSP, CIPP/E) and enterprise background (Huawei Cloud, Alibaba Cloud)
