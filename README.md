# Authalla Agent Skills

Official [Claude Code skills](https://skills.sh) for [Authalla](https://authalla.com) — the OAuth2/OIDC authentication platform.

## Install

```bash
npx skills add authalla/authalla-skills
```

## Skills

### authalla

Set up Authalla authentication for your app through a guided interactive flow. Configures branding, custom domain, email, social login, and creates your OAuth2 client.

**Prerequisites:**
- An Authalla account ([sign up](https://authalla.com))
- The Authalla CLI: `brew tap authalla/tap && brew install authalla`

**Usage:** Once installed, Claude Code will automatically use this skill when you ask it to set up Authalla authentication.
