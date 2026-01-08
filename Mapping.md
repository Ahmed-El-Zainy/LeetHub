# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

LeetHub is a Chrome/Firefox extension that automatically pushes LeetCode and GeeksforGeeks problem solutions to GitHub when all tests pass. It uses Chrome Extension APIs with OAuth2 authentication to GitHub.

## Development Commands

### Setup
```bash
npm run setup
# Equivalent to: npm i
```

### Code Quality
```bash
npm run format        # Auto-format JavaScript, HTML/CSS with Prettier
npm run format-test   # Verify all code is formatted correctly
npm run lint          # Lint and auto-fix JavaScript with ESLint
npm run lint-test     # Verify all code passes linting (exits with error on failure)
```

### Testing the Extension Locally
1. Navigate to `chrome://extensions` in Chrome
2. Enable Developer mode (toggle in top right)
3. Click "Load unpacked"
4. Select the LeetHub root directory

## Architecture

### Extension Components

**Background Script** (`scripts/background.js`)
- Persistent event listener for messages from content scripts
- Handles OAuth flow completion by storing GitHub token and username
- Opens welcome.html after successful authentication

**Content Scripts** (injected into specific pages)
- `scripts/leetcode.js` - Main LeetCode integration, monitors submissions
- `scripts/gfg.js` - GeeksforGeeks integration, similar pattern to leetcode.js
- `scripts/authorize.js` - Handles OAuth2 redirect from GitHub
- All content scripts run at `document_idle` on their respective domains

**Popup UI**
- `popup.html` + `popup.js` - Extension popup interface
- Shows authentication state, problem statistics, and repo link
- Dynamically displays different modes based on storage state

**Welcome/Setup Page**
- `welcome.html` + `scripts/welcome.js` - Onboarding flow
- Creates or links GitHub repository for storing solutions
- Handles repository creation via GitHub API

### Data Flow

1. **Authentication**: User clicks "Authorize" → OAuth flow → `authorize.js` extracts token → `background.js` stores it → redirects to `welcome.html`

2. **Problem Submission**: 
   - Content script detects successful submission on LeetCode/GFG
   - Extracts problem title, difficulty, code, language
   - Calls `uploadGit()` function to push to GitHub via API
   - Updates local statistics (problems solved by difficulty)

3. **Storage Schema** (chrome.storage.local):
   - `leethub_token` - GitHub OAuth token
   - `leethub_username` - GitHub username
   - `leethub_hook` - Repository full name (e.g., "username/repo")
   - `mode_type` - Either "commit" (repo linked) or "hook" (setup pending)
   - `stats` - Object containing `{solved, easy, medium, hard, sha: {...}}`
   - `pipe_leethub` - Boolean flag for OAuth redirect flow

### GitHub API Integration

All GitHub operations use direct XMLHttpRequest calls with personal access token authentication:
- Repository creation/linking: `POST https://api.github.com/user/repos`
- File upload/update: `PUT https://api.github.com/repos/{owner}/{repo}/contents/{path}`
- User validation: `GET https://api.github.com/user`

Files are uploaded as base64-encoded content with SHA tracking for updates.

### Language Support

The `languages` object maps platform-reported languages to file extensions. Supported languages include: Python, C++, C, Java, C#, JavaScript, Ruby, Swift, Go, Kotlin, Scala, Rust, PHP, TypeScript, and SQL variants.

## Code Style

- **ESLint**: Airbnb config with Prettier integration
- **Prettier**: Single quotes, semicolons, 70-char line width, trailing commas
- Environment: Browser + jQuery + Web Extensions APIs
- All code must pass both `npm run lint-test` and `npm run format-test` before commit

## Key Implementation Patterns

**Content Script Monitoring**: Both `leetcode.js` and `gfg.js` use `setInterval` to poll for submission success indicators in the DOM, then extract problem data and trigger uploads.

**SHA-based Updates**: The extension tracks GitHub file SHAs locally to support updating existing files (for re-submissions) vs. creating new ones.

**Problem Statistics**: Stats are incremented only when README.md is created (sha === null) to avoid double-counting, since each submission creates both README and code file.

**Commit Messages**:
- `"Create README - LeetHub"` - Problem statement
- `"Added solution - LeetHub"` (GFG) or custom messages (LeetCode)
- `"Prepend discussion post - LeetHub"` - For discussion additions

## Important Notes

- OAuth credentials are hardcoded in `authorize.js` (CLIENT_ID, CLIENT_SECRET)
- All repositories are created as private by default
- The extension requires specific DOM selectors that may break with platform UI changes
- No automated tests are included in the project
