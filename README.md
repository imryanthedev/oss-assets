# oss-assets

Public, persistent asset repository for visual proofs (screenshots, screencasts, animations, and before/after comparisons) embedded in pull requests, issues, and discussions across external open-source repositories contributed to by [@imryanthedev](https://github.com/imryanthedev).

---

## Why this repository exists

1. **Permission constraints on external forks:** When contributing to upstream repositories where the contributor lacks write access (`push: false`), GitHub CLI's `--attach` flag and direct API calls to `uploads.github.com/user-attachments/assets` fail (`requires write access to repository`).
2. **Longevity & Cleanliness:** Hosting assets in a dedicated public repository prevents polluting code branches with binary assets and guarantees that embedded images never break (404) when working branches or forks are synchronized or pruned.
3. **Universally accessible URLs:** Assets committed here are served directly and publicly via GitHub's raw CDN:
   ```text
   https://raw.githubusercontent.com/imryanthedev/oss-assets/main/assets/<org>/<repo>/<item>/<filename>
   ```

---

## Directory Structure

All visual media must be structured hierarchically by organization/owner, repository, and issue/PR identifier:

```text
assets/
└── <organization-or-owner>/
    └── <repository-name>/
        └── <pr-or-issue-id>/
            ├── before.png
            ├── after.png
            └── reproduction.gif
```

### Example
For a fix submitted to `zed-industries/zed` on PR `#66960`:
- **Repository path:** `assets/zed-industries/zed/pr-66960/preview-shown.gif`
- **Markdown embed URL:**
  ```markdown
  ![Demonstration](https://raw.githubusercontent.com/imryanthedev/oss-assets/main/assets/zed-industries/zed/pr-66960/preview-shown.gif)
  ```

---

## How Agents & Scripts Upload Assets

### Method 1: GitHub Contents API via `gh api` (Zero Git Checkout Required)

Agents can autonomously upload media without cloning this repository using the GitHub Contents API.

> [!IMPORTANT]
> **Linux `ARG_MAX` buffer safety:** Never pass large base64 strings as CLI arguments (`-f content="$(base64 ...)"`) as Linux limits single arguments to 128KB, causing failures for files > ~96KB (especially GIFs and videos). Always stream the payload via a JSON file or stdin using `jq` and `--input`:

```bash
#!/usr/bin/env bash
set -euo pipefail

FILE_PATH="screenshot.png"
ORG="zed-industries"
REPO="zed"
PR_OR_ISSUE="pr-66960"
FILE_NAME="$(basename "$FILE_PATH")"
TARGET_PATH="assets/${ORG}/${REPO}/${PR_OR_ISSUE}/${FILE_NAME}"

# Prepare JSON payload safely avoiding Linux command argument length limits
TMP_PAYLOAD=$(mktemp)
trap 'rm -f "$TMP_PAYLOAD"' EXIT

jq -n \
  --arg m "assets(${ORG}/${REPO}): add visual proof for ${PR_OR_ISSUE}" \
  --rawfile c <(base64 -w 0 "$FILE_PATH") \
  '{message: $m, content: $c}' > "$TMP_PAYLOAD"

# Upload via GitHub Contents API
RAW_URL=$(gh api \
  --method PUT \
  "/repos/imryanthedev/oss-assets/contents/${TARGET_PATH}" \
  --input "$TMP_PAYLOAD" \
  --jq .content.download_url)

echo "Uploaded successfully: $RAW_URL"
```

The resulting `$RAW_URL` can immediately be embedded into the PR description or comment markdown.

### Method 2: Standard Git Workflow (Local Checkout)

From the local checkout of `oss-assets`:

```bash
mkdir -p "assets/zed-industries/zed/pr-66960"
cp /path/to/screenshot.png "assets/zed-industries/zed/pr-66960/"
git add assets/
git commit -m "assets(zed-industries/zed): add screenshot for pr-66960"
git push origin main
```

---

## Media Guidelines & Policies

- **Supported Formats:** PNG, JPEG, GIF, WebP, SVG, MP4.
- **Optimization:** Optimize and compress screenshots and GIFs before uploading (recommended: keep under 5MB per file).
- **Security & Privacy:** Never commit secrets, tokens, credentials, server IPs, or personal private data.
- **Immutability:** Once an asset is referenced in a merged or active public PR, do not rename or delete it.
