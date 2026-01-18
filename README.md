# Ballrs Website

Official website for Ballrs - the sports trivia game.

**Live at:** https://ballrs.net

## Pages

- `/` - Landing page
- `/privacy-policy` - Privacy Policy
- `/terms-of-service` - Terms of Service
- `/delete-account` - Account deletion instructions

## Development

This is a static website. To preview locally:

```bash
# Using Python
python -m http.server 8000

# Using Node.js
npx serve .

# Using PHP
php -S localhost:8000
```

Then open http://localhost:8000

## Deployment

### Option 1: Netlify (Recommended)

1. Push this repo to GitHub
2. Go to [Netlify](https://netlify.com)
3. Click "Add new site" > "Import an existing project"
4. Connect your GitHub account and select `ballrs-website`
5. Deploy settings are auto-configured via `netlify.toml`
6. Click "Deploy site"

#### Connect Custom Domain (ballrs.net)

1. In Netlify, go to Site settings > Domain management
2. Click "Add custom domain"
3. Enter `ballrs.net`
4. Add the following DNS records at your domain registrar:

   **For apex domain (ballrs.net):**
   - Type: `A`
   - Name: `@`
   - Value: `75.2.60.5`

   **For www subdomain:**
   - Type: `CNAME`
   - Name: `www`
   - Value: `[your-site-name].netlify.app`

5. Wait for DNS propagation (can take up to 48 hours)
6. Enable HTTPS in Netlify (automatic with Let's Encrypt)

### Option 2: Vercel

1. Push this repo to GitHub
2. Go to [Vercel](https://vercel.com)
3. Click "Add New" > "Project"
4. Import the `ballrs-website` repository
5. Deploy (no configuration needed for static sites)

#### Connect Custom Domain

1. In Vercel, go to Project Settings > Domains
2. Add `ballrs.net`
3. Follow the DNS configuration instructions provided

### Option 3: GitHub Pages

1. Go to repo Settings > Pages
2. Source: Deploy from branch `main`
3. Custom domain: `ballrs.net`
4. Add DNS records as instructed

## Design

- **Colors:**
  - Cream background: `#F5F2EB`
  - Teal primary: `#1ABC9C`
  - Yellow secondary: `#F2C94C`
  - Black: `#1A1A1A`

- **Font:** DM Sans (Google Fonts)

- **Style:** Neubrutalist - bold borders, shadows, strong typography

## Files

```
ballrs-website/
├── index.html           # Landing page
├── privacy-policy.html  # Privacy Policy
├── terms-of-service.html # Terms of Service
├── delete-account.html  # Account deletion
├── styles.css           # Shared styles
├── netlify.toml         # Netlify config
├── favicon.png          # Site favicon (add your own)
└── README.md            # This file
```

## To Do

- [ ] Add favicon.png (copy from app assets)
- [ ] Add App Store link when available
- [ ] Add Google Play link when available
- [ ] Add app screenshots to landing page

## Contact

hello@ballrs.net
