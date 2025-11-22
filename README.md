#!/usr/bin/env bash
set -e
echo "=== CHAT-BOOSTER ZERO-TOUCH APPLY ==="

# check zip exists
if [ ! -f chat_booster_production.zip ]; then
  echo "ERROR: chat_booster_production.zip not found. Upload it to Codespaces first."
  exit 1
fi

# unzip
rm -rf chat-booster
unzip -o chat_booster_production.zip -d chat-booster
cd chat-booster

# install deps & playwright
echo "Installing npm deps..."
npm ci
echo "Installing Playwright browsers..."
npx playwright install --with-deps

# git init & commit
git init
git checkout -b main
git add .
git commit -m "chore: initial zero-touch apply"

# ask repo url
echo ""
read -p "Enter your GitHub repo URL (git@github.com:USER/REPO.git or https://...): " REPO_URL
git remote add origin "$REPO_URL"
git push -u origin main --force

# ask whether Vercel app installed
echo ""
echo "Important: for true zero-touch, install 'Vercel for GitHub' on your repo (one-time via web UI)."
read -p "Have you installed Vercel for GitHub on that repo? (y/N): " VERCEL_APP_INSTALLED
VERCEL_APP_INSTALLED=${VERCEL_APP_INSTALLED:-N}

# optional: write secrets via GitHub CLI (user choice)
echo ""
read -p "Do you want to optionally add API keys as GitHub Secrets now via gh CLI? (y/N): " ADD_SECRETS
ADD_SECRETS=${ADD_SECRETS:-N}
if [ "$ADD_SECRETS" = "y" ] || [ "$ADD_SECRETS" = "Y" ]; then
  echo "This will use 'gh' (GitHub CLI) to write secrets into the repo. You will be prompted for a GitHub PAT if needed."
  # check gh auth
  if ! gh auth status >/dev/null 2>&1; then
    echo "gh CLI not authenticated. Please login (one-time):"
    gh auth login
  fi
  echo "Setting placeholders for secrets (you can override in GitHub UI later)."
  gh secret set VERCEL_TOKEN -b"placeholder_vercel_token"
  gh secret set GOOGLE_TRENDS_API_KEY -b"placeholder_google"
  gh secret set REDDIT_TOKEN -b"placeholder_reddit"
  gh secret set X_BEARER_TOKEN -b"placeholder_x"
  echo "Secrets set (placeholders). Update them securely in GitHub Settings → Secrets if you want real API integration."
fi

# If Vercel app installed — we can trigger a deploy via push (Vercel auto-deploys)
if [[ "$VERCEL_APP_INSTALLED" =~ ^([yY])$ ]]; then
  echo "Vercel GitHub App detected by you. Vercel will auto-deploy on push."
  echo "Triggering a deployment by creating an empty commit..."
  git commit --allow-empty -m "chore: trigger vercel deploy [zero-touch]" || true
  git push origin main
  echo "Deployment triggered via Vercel GitHub App."
else
  echo ""
  echo "If you did not install Vercel GitHub App, you can still deploy once by generating a Vercel token locally and running the vercel CLI."
  echo "To install Vercel App: https://github.com/marketplace/vercel"
  echo "Or run: vercel --prod --token YOUR_TOKEN"
fi

echo ""
echo "=== ZERO-TOUCH APPLY COMPLETE ==="
echo "What happens next (automated):"
echo "- GitHub Actions will run CI (lint, jest, playwright e2e, visual tests)."
echo "- Daily & monthly analysis jobs will run on schedule and may open PRs to optimize weights."
echo "- If Vercel GitHub App is installed, deployments are automatic on push."
echo ""
echo "Notes:"
echo "- If you want the system to fetch real Google Trends/Reddit/X data, update the corresponding GitHub Secrets with real tokens."
echo "- You can later enable fully automatic secrets updates via an admin PAT; but that is optional and not required for zero-touch baseline."