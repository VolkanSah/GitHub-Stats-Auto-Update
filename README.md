# 📊 GitHub Stats Auto-Update

Automatically update your GitHub stats in the README.md using GitHub Actions.

## 🎯 What does the script do?

The script displays **only public** GitHub stats:

* Public repositories
* Total stars (from public repos only)
* Total forks (from public repos only)
* Followers

> **Note:** Private repositories and their stars/forks are not counted. The script uses only publicly available API data.

## 🚀 Setup

### 1. Create the files

Create these two files in your profile repository:

**`.github/workflows/update-stats.yml`**

```yaml
name: Update Stats

on:
  schedule:
    - cron: '0 6 * * *'  # Every day at 6 AM
  workflow_dispatch:  # Manual trigger option

jobs:
  update-stats:
    runs-on: ubuntu-latest
    permissions:
      contents: write  # Required for push
    steps:
      - name: Checkout Repo
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.x'
      
      - name: Install Dependencies
        run: pip install requests
      
      - name: Run Stats Script
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: python update_stats.py
      
      - name: Commit & Push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add README.md
          git commit -m "🤖 Automatic GitHub Stats Update" || exit 0
          git push
```

**`update_stats.py`**

```python
import requests
import os
import re

# GitHub Username - ENTER YOUR USERNAME HERE!
username = "YOUR_USERNAME_HERE"

# Get Token
token = os.getenv("GITHUB_TOKEN")
if not token:
    print("❌ GITHUB_TOKEN not found!")
    exit(1)

headers = {"Authorization": f"Bearer {token}"}

# API Calls with Error Handling
try:
    user_url = f"https://api.github.com/users/{username}"
    
    print("🔍 Fetching user data...")
    user_response = requests.get(user_url, headers=headers)
    user_response.raise_for_status()
    user_data = user_response.json()
    
    print("🔍 Fetching repo data...")
    repos_data = []
    page = 1
    while True:
        repos_url_page = f"https://api.github.com/users/{username}/repos?per_page=100&page={page}"
        repos_response = requests.get(repos_url_page, headers=headers)
        repos_response.raise_for_status()
        page_data = repos_response.json()
        if not page_data:  # No more repos
            break
        repos_data.extend(page_data)
        page += 1
    
    # Debug: Check type
    print(f"📊 Repositories found: {len(repos_data)}")
    
    # Calculate stats (public repos only)
    total_stars = sum(repo.get("stargazers_count", 0) for repo in repos_data)
    total_forks = sum(repo.get("forks_count", 0) for repo in repos_data)
    public_repos = user_data.get("public_repos", 0)
    followers = user_data.get("followers", 0)
    
    print(f"⭐ Stars: {total_stars}, 🍴 Forks: {total_forks}, 📁 Repos: {public_repos}, 👥 Followers: {followers}")
    
except requests.exceptions.RequestException as e:
    print(f"❌ API Error: {e}")
    exit(1)
except Exception as e:
    print(f"❌ Error: {e}")
    exit(1)

# Markdown content
stats_md = f"""<!-- STATS-START -->
# 📊 GitHub Stats
- **Public Repositories:** {public_repos}
- **Total Stars:** {total_stars}
- **Total Forks:** {total_forks}
- **Followers:** {followers}

*Last updated automatically via GitHub Actions.*
<!-- STATS-END -->"""

# Load README
try:
    with open("README.md", "r", encoding="utf-8") as f:
        readme_content = f.read()
except FileNotFoundError:
    print("❌ README.md not found!")
    exit(1)

# Replace or insert block
pattern = r"<!-- STATS-START -->.*?<!-- STATS-END -->"
if re.search(pattern, readme_content, re.DOTALL):
    new_readme = re.sub(pattern, stats_md, readme_content, flags=re.DOTALL)
    print("✅ Stats section in README.md updated.")
else:
    new_readme = readme_content.strip() + "\n\n" + stats_md
    print("✅ Stats section added to README.md.")

# Save
with open("README.md", "w", encoding="utf-8") as f:
    f.write(new_readme)

print("🎉 Done!")
```

### 2. Prepare your README.md

Add these markers in your README.md where the stats should appear:

```markdown
<!-- STATS-START -->
<!-- STATS-END -->
```

### 3. Adjust your username

**Important:** Change the following line in `update_stats.py`:

```python
username = "YOUR_USERNAME_HERE"
```

### 4. Test the Action

* Go to your repository
* Click "Actions"
* Select "Update Stats"
* Click "Run workflow"

## ⚙️ Configuration

### Change the schedule

```yaml
- cron: '0 6 * * *'  # Daily at 6 AM
- cron: '0 12 * * 1'  # Mondays at 12 PM  
- cron: '0 0 1 * *'   # First day of each month
```

### Manual trigger

In the repository under "Actions" → "Update Stats" → "Run workflow"

## 🔒 Security

* Uses the default `GITHUB_TOKEN` (no extra secrets needed)
* Displays only public data
* No private repository information

## 📝 Notes

* **Public stats only:** Private repos are not included
* **API limits:** GitHub API has rate limits, but daily updates are fine
* **Pagination:** Fetches all repos (even over 100)
* **Error handling:** Aborts on errors, no broken updates

## 🎨 Customization

You can adjust the Markdown output in the `stats_md` variable:

```python
stats_md = f"""<!-- STATS-START -->
# 🚀 My GitHub Journey
**📁 Repositories:** {public_repos} | **⭐ Stars:** {total_stars} | **🍴 Forks:** {total_forks} | **👥 Followers:** {followers}
<!-- STATS-END -->"""
```

## 🛠️ Troubleshooting

**Action fails:**

* Check if your username is correct
* Look at the Action logs for details

**Stats don't show:**

* Make sure `<!-- STATS-START -->` and `<!-- STATS-END -->` are in your README.md

**Numbers seem wrong:**

* The script only counts public repository stats
* Private repos are not included

---

**Enjoy your automatic GitHub stats! 🎉**


