# 📊 GitHub Stats Auto-Update

Automatically update your GitHub stats in the README.md using GitHub Actions. Demo https://github.com/VolkanSah/

## 🎯 What does the script do?

The script displays **only public** GitHub stats with detailed breakdown:

* Public repositories (active + archived)
* Total stars separated by:
  - ⭐ Active repository stars
  - 💎 Archived repository stars
  - 🌟 Total stars per category
* Separate stats for own repos vs. forked repos
* 🎯 Grand total of all stars

> **Note:** Private repositories and their stars are not counted. The script uses only publicly available API data via GraphQL for accurate pagination.

## ✨ Features

- ✅ **Complete pagination** - fetches ALL repos (not just first 100)
- ✅ **Archive-aware** - separates active from archived repository stars
- ✅ **Fork separation** - distinguishes between own and forked repos
- ✅ **Detailed breakdown** - shows active, archived, and total stats
- ✅ **Top 10 list** - displays your most starred repositories
- ✅ **Full repo list** - complete overview of all repos with stars

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
USERNAME = "YOUR_USERNAME_HERE"

TOKEN = os.getenv("GITHUB_TOKEN")
if not TOKEN:
    print("❌ GITHUB_TOKEN not found!")
    exit(1)

HEADERS = {"Authorization": f"Bearer {TOKEN}"}

def fetch_all_repos(is_fork):
    """Fetch ALL repos with pagination + extended info"""
    all_repos = []
    has_next = True
    cursor = None
    
    while has_next:
        query = """
        {
          user(login: "%s") {
            repositories(first: 100, privacy: PUBLIC, isFork: %s, ownerAffiliations: OWNER%s) {
              nodes {
                name
                stargazerCount
                isArchived
                isDisabled
                isLocked
                owner {
                  login
                }
              }
              pageInfo {
                hasNextPage
                endCursor
              }
            }
          }
        }
        """ % (USERNAME, str(is_fork).lower(), f', after: "{cursor}"' if cursor else '')
        
        try:
            response = requests.post(
                "https://api.github.com/graphql",
                json={"query": query},
                headers=HEADERS
            )
            response.raise_for_status()
            data = response.json()
            
            if "errors" in data:
                print(f"❌ API Error: {data['errors']}")
                exit(1)
            
            repos = data["data"]["user"]["repositories"]
            all_repos.extend(repos["nodes"])
            
            page_info = repos["pageInfo"]
            has_next = page_info["hasNextPage"]
            cursor = page_info["endCursor"]
            
            print(f"  📦 Fetched {len(repos['nodes'])} repos (Total: {len(all_repos)})")
            
        except requests.exceptions.RequestException as e:
            print(f"❌ API Error: {e}")
            exit(1)
    
    return all_repos

def calculate_stats(repos, repo_type):
    """Calculate stats with filtering"""
    # Filter: Only active, non-archived repos
    active_repos = [
        r for r in repos 
        if not r.get("isArchived", False) 
        and not r.get("isDisabled", False)
        and not r.get("isLocked", False)
        and r.get("owner", {}).get("login") == USERNAME
    ]
    
    # Archived repos separately
    archived_repos = [
        r for r in repos 
        if (r.get("isArchived", False) or r.get("isDisabled", False) or r.get("isLocked", False))
        and r.get("owner", {}).get("login") == USERNAME
    ]
    
    archived_count = len(archived_repos)
    archived_stars = sum(repo.get("stargazerCount", 0) for repo in archived_repos)
    
    active_stars = sum(repo.get("stargazerCount", 0) for repo in active_repos)
    active_count = len(active_repos)
    
    print(f"\n📊 {repo_type.capitalize()} Repositories:")
    print(f"  ✅ Active: {active_count}")
    if archived_count > 0:
        print(f"  🗄️  Archived/Disabled: {archived_count} (with {archived_stars} ⭐)")
    print(f"⭐ {repo_type.capitalize()} Stars:")
    print(f"  Active: {active_stars}")
    if archived_stars > 0:
        print(f"  Archive: {archived_stars} 💎")
    print(f"  Total: {active_stars + archived_stars}")
    
    # Top 10 repos with most stars
    print(f"\n🏆 Top 10 {repo_type} Repos:")
    top_repos = sorted(active_repos, key=lambda x: x.get("stargazerCount", 0), reverse=True)[:10]
    for i, repo in enumerate(top_repos, 1):
        print(f"  {i:2}. {repo['name']:40} {repo.get('stargazerCount', 0):4} ⭐")
    
    return active_count, active_stars, archived_count, archived_stars

def update_readme(own_repos, own_stars, own_archived_stars, forked_repos, forked_stars, forked_archived_stars):
    """Update the README"""
    stats_md = f"""<!-- STATS-START -->
## 📊 GitHub Stats
- **Own Public Repositories:** {own_repos}
  - ⭐ Active Stars: {own_stars}
  - 💎 Archived Stars: {own_archived_stars}
  - 🌟 Total Own Stars: {own_stars + own_archived_stars}
- **Forked Public Repositories:** {forked_repos}
  - ⭐ Active Stars: {forked_stars}
  - 💎 Archived Stars: {forked_archived_stars}
  - 🌟 Total Fork Stars: {forked_stars + forked_archived_stars}
- **🎯 Grand Total Stars:** {own_stars + own_archived_stars + forked_stars + forked_archived_stars}

*Last updated automatically via GitHub Actions.*
<!-- STATS-END -->"""
    
    try:
        with open("README.md", "r", encoding="utf-8") as f:
            readme_content = f.read()
    except FileNotFoundError:
        print("❌ README.md not found!")
        exit(1)
    
    pattern = r"<!-- STATS-START -->.*?<!-- STATS-END -->"
    if re.search(pattern, readme_content, re.DOTALL):
        new_readme = re.sub(pattern, stats_md, readme_content, flags=re.DOTALL)
        print("\n✅ Stats section updated.")
    else:
        new_readme = readme_content.strip() + "\n\n" + stats_md
        print("\n✅ Stats section added.")
    
    with open("README.md", "w", encoding="utf-8") as f:
        f.write(new_readme)
    
    print("🎉 Done!")

if __name__ == "__main__":
    print("🔍 Fetching own repositories...")
    own_repos_data = fetch_all_repos(False)
    own_repos, own_stars, own_archived, own_archived_stars = calculate_stats(own_repos_data, "own")
    
    print("\n" + "="*80)
    print("🔍 Fetching forked repositories...")
    forked_repos_data = fetch_all_repos(True)
    forked_repos, forked_stars, forked_archived, forked_archived_stars = calculate_stats(forked_repos_data, "forked")
    
    print("\n" + "="*80)
    print(f"📈 TOTAL:")
    print(f"  Active Repos: {own_repos + forked_repos}")
    print(f"  Archived Repos: {own_archived + forked_archived}")
    print(f"  ⭐ Active Stars: {own_stars + forked_stars}")
    print(f"  💎 Archive Stars: {own_archived_stars + forked_archived_stars}")
    print(f"  🌟 GRAND TOTAL: {own_stars + own_archived_stars + forked_stars + forked_archived_stars} ⭐")
    
    update_readme(own_repos, own_stars, own_archived_stars, forked_repos, forked_stars, forked_archived_stars)
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
USERNAME = "YOUR_USERNAME_HERE"
```

### 4. Test the Action

* Go to your repository
* Click "Actions"
* Select "Update Stats"
* Click "Run workflow"

## 📊 Stats Breakdown

The script shows:

### Own Repositories
- **Active Stars**: Stars from currently maintained projects
- **Archived Stars**: Stars from archived/legacy projects
- **Total Own Stars**: Complete star history

### Forked Repositories  
- Same breakdown for your forked repositories

### Grand Total
- Your complete GitHub star collection across all public repositories

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
* Uses GraphQL API for efficient data fetching

## 📝 Notes

* **Public stats only:** Private repos are not included
* **Complete pagination:** Fetches ALL repositories (no 100-repo limit)
* **Archive-aware:** Distinguishes between active and archived projects
* **Error handling:** Aborts on errors, no broken updates
* **Top 10 list:** Shows your most popular repositories in console output

## 🎨 Customization

You can adjust the Markdown output in the `stats_md` variable in the `update_readme()` function.

Example for a compact single-line format:

```python
stats_md = f"""<!-- STATS-START -->
**📁 Repos:** {own_repos} | **⭐ Active Stars:** {own_stars} | **💎 Archive Stars:** {own_archived_stars} | **🎯 Total:** {own_stars + own_archived_stars + forked_stars + forked_archived_stars}
<!-- STATS-END -->"""
```

## 🛠️ Troubleshooting

**Action fails:**
* Check if your username is correct
* Look at the Action logs for details
* Verify `GITHUB_TOKEN` permissions

**Stats don't show:**
* Make sure `<!-- STATS-START -->` and `<!-- STATS-END -->` are in your README.md
* Check that markers are on separate lines

**Numbers seem wrong:**
* The script only counts public repository stats
* Private repos are not included
* Check console output for detailed breakdown

**GraphQL API errors:**
* GitHub API has rate limits
* Daily updates should work fine
* Manual runs might hit limits if run too frequently

## 🌟 Why GraphQL?

This script uses GitHub's GraphQL API instead of REST because:
- ✅ **Complete pagination** - no 100-repo limit issues
- ✅ **Single request** per page - more efficient
- ✅ **Precise filtering** - `ownerAffiliations: OWNER` ensures only your repos
- ✅ **Archive detection** - knows which repos are archived
- ✅ **Better rate limits** - GraphQL is more efficient

---

**Enjoy your automatic GitHub stats with full transparency! 🎉**
