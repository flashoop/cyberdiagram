# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

CyberDiagram is a landing page for an AI-powered cloud security platform that provides automated penetration testing for SMEs. The project consists of a single-page HTML landing page designed to showcase the product offering, explain the problem/solution, and capture early access signups.

**Project Timeline:** 2025-2027
**Target Audience:** Small and medium-sized enterprises (SMEs) lacking dedicated security teams
**Core Value Proposition:** Enterprise-grade cloud security made accessible through AI automation

## Development Commands

### Local Development

```bash
# Serve the landing page locally
python3 -m http.server 8000
# Visit http://localhost:8000
```

### Deployment

The site is configured for static hosting:
- **GitHub Pages**: Deployed at custom domain via CNAME file
- **CNAME**: `cyberdiagram.com` (configured in root CNAME file)
- **Main file**: [index.html](index.html)

No build process required - this is a static HTML/CSS/JavaScript site.

## Architecture

### Single-Page Application Structure

The landing page is a self-contained HTML file with embedded CSS and JavaScript:

1. **Navigation** ([index.html:769-781](index.html#L769-L781))
   - Fixed header with 3 navigation tabs (Home, Product, About)
   - Smooth scroll to sections with active state highlighting
   - Scroll-triggered styling effects

2. **Hero Section** ([index.html:783-795](index.html#L783-L795))
   - Primary value proposition
   - Dual CTAs: "Request Early Access" and "See Demo"
   - Gradient background for visual appeal

3. **Visual Placeholder** ([index.html:797-812](index.html#L797-L812))
   - Ready for attack path diagram visualization
   - Currently displays placeholder content
   - Designed to showcase the product's core visual intelligence feature

4. **Problem Section** ([index.html:814-854](index.html#L814-L854))
   - Three-card grid explaining SME security challenges
   - Icons and hover effects for engagement
   - Pain points: Limited resources, complex clouds, costly pentests

5. **Solution Section** ([index.html:856-896](index.html#L856-L896))
   - Three-feature showcase
   - Continuous Protection, Visual Intelligence, Multi-Cloud Mastery
   - Emphasizes AI/automation advantage over traditional approaches

6. **How It Works** ([index.html:898-939](index.html#L898-L939))
   - Four-step process flow
   - Numbered steps with visual indicators
   - Describes: Connect → Scan → Visualize → Remediate

7. **Founder Credibility** ([index.html:941-998](index.html#L941-L998))
   - Founder profile with image ([me.jpeg](me.jpeg))
   - LinkedIn integration for verification
   - Credentials: 20+ years experience, 6 elite certifications
   - Certifications: OSEP, OSCP, GCPN, CISSP, CCSP, CIPP/E
   - Background: Huawei Cloud (8 years), Alibaba Cloud (3 years)

8. **Final CTA** ([index.html:1000-1007](index.html#L1000-L1007))
   - Conversion-focused section
   - Early access program messaging
   - Preferred pricing incentive

9. **Footer** ([index.html:1009-1042](index.html#L1009-L1042))
   - Placeholder links for future pages
   - Project timeline display
   - Standard legal links (Privacy, Terms)

### Design System

**Color Palette:**
- Primary Blue: `#2563EB` (brand color, CTAs, accents)
- Dark Blue: `#1E40AF` (gradients, hover states)
- LinkedIn Blue: `#0A66C2` (social links)
- Backgrounds: White (`#FFFFFF`), Light Gray (`#F9FAFB`)
- Text: Dark Gray (`#1F2937`, `#4B5563`, `#6B7280`)

**Typography:**
- Font Family: Inter (Google Fonts)
- Heading weights: 600-700
- Body weight: 400-500
- Font size scale follows clear hierarchy

**Responsive Breakpoints:**
- Mobile: < 768px ([index.html:688-739](index.html#L688-L739))
- Tablet/Desktop: ≥ 768px

### JavaScript Functionality

All interactivity is handled by vanilla JavaScript ([index.html:1045-1091](index.html#L1045-L1091)):

1. **Navbar scroll effects**: Adds shadow when scrolled past 50px
2. **Smooth scrolling**: Native CSS smooth scroll with JavaScript fallback
3. **Active link highlighting**: Updates navigation state based on scroll position
4. **Section visibility detection**: Triggers active state 200px before section

## Key Technical Details

### File Structure

```
mockup/
├── CNAME                        # Domain configuration
├── index.html                   # Main landing page (self-contained)
├── me.jpeg                      # Founder profile image
├── attack.png                   # Attack path visualization reference
├── cloud-architecture.png       # Architecture diagram
├── cloud-attach-path.png        # Attack path diagram
├── cloud-business-architecture.png  # Business architecture
├── README.md                    # Customization guide
├── development_roadmap.md       # Personal learning/project roadmap
├── marketing_materials.md       # Source marketing copy
└── quick_pitch_materials.md     # Pitch deck materials
```

### Content Strategy

The landing page implements a classic conversion-focused structure:
1. **Value prop** → 2. **Visual proof** → 3. **Problem** → 4. **Solution** → 5. **Process** → 6. **Credibility** → 7. **CTA**

### Customization Points

When updating the landing page, focus on these areas:

1. **Call-to-Action Links**: Replace `href="#"` with actual signup form or email capture (currently placeholders at [index.html:791-792](index.html#L791-L792))

2. **Visual Placeholder**: Replace with actual attack path diagram at [index.html:800-810](index.html#L800-L810) - image reference already exists as [attack.png](attack.png)

3. **Founder Profile**: Image path at [index.html:956](index.html#L956) - currently uses [me.jpeg](me.jpeg)

4. **LinkedIn URL**: Update at [index.html:963](index.html#L963) - currently set to `https://www.linkedin.com/in/zhaohongri/`

5. **Email Capture**: No backend integration yet - needs form handling for lead collection

## Design Principles

1. **Minimalist White Aesthetic**: Clean, professional, enterprise-appropriate
2. **Blue Color Scheme**: Trust, security, technology
3. **Responsive-First**: Mobile, tablet, desktop optimized
4. **Performance**: Single file, minimal dependencies, fast load
5. **Accessibility**: Semantic HTML, proper heading hierarchy
6. **Smooth Interactions**: CSS transitions, hover effects, scroll animations

## Future Development Notes

Per the README and development roadmap, future work includes:

1. **Email Capture Integration**: Connect to backend/CRM (Mailchimp, SendGrid, etc.)
2. **Analytics**: Add Google Analytics or similar tracking
3. **Additional Pages**: Product details, About page, Documentation
4. **Testimonials Section**: Once beta customers provide feedback
5. **A/B Testing**: Test different CTA copy and positioning
6. **Backend API**: For lead processing and early access management

## Security Considerations

- No sensitive data currently in codebase
- Static HTML only - no server-side processing yet
- Future: Implement proper form validation and CSRF protection when adding backend
- All external links use `target="_blank"` with `rel="noopener noreferrer"` for security

## Brand Voice

Copy tone is:
- **Professional but accessible** - not overly technical
- **Problem-focused first** - emphasizes pain points before solution
- **Founder-led credibility** - leverages personal expertise
- **Urgency without hype** - "early access" framing, not overselling

## Context for AI Development

This landing page serves as the public face of CyberDiagram, a project aiming to:
- Democratize enterprise-grade cloud security testing
- Use AI (particularly large language models) for automated pentesting
- Provide visual attack path diagrams that non-technical stakeholders can understand
- Target the underserved SME market (vs. traditional enterprise focus)

The founder has deep cloud security expertise (Huawei, Alibaba) and elite security certifications, which serves as the primary credibility driver for this early-stage product.