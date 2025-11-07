# GitHub-Hosted CSV Loading Implementation Plan

**For Claude: REQUIRED SUB-SKILL: Use superpowers:executing-plans**

## Goal
Modify the dashboard to load CSV data from GitHub instead of a local file, enabling single-file distribution where the HTML always fetches the latest data automatically.

## Architecture
Replace local file loading (`Papa.parse('FW_Veg_Rem_Combined.csv')`) with GitHub raw URL fetch. Use GitHub's raw content URL which serves files directly without HTML wrapper. Add error handling for network failures and CORS issues.

## Tech Stack
- **GitHub Raw URL**: `https://raw.githubusercontent.com/varunr89/smokey/main/FW_Veg_Rem_Combined.csv`
- **PapaParse**: Already supports remote URLs via `download: true` parameter
- **Fallback**: Optional local file loading if GitHub fetch fails

---

## Task 1: Update CSV Loading to Use GitHub URL

**Files:**
- **Modify**: `index.html` (lines ~450-470, loadData function)

### Step 1: Identify current loading code

Current code (around line 450):
```javascript
async function loadData() {
    return new Promise((resolve, reject) => {
        Papa.parse('FW_Veg_Rem_Combined.csv', {
            download: true,
            header: true,
            dynamicTyping: true,
            skipEmptyLines: true,
            complete: function(results) {
                rawData = results.data;
                resolve();
            },
            error: function(error) {
                reject(new Error('Could not load data file. ' + error.message));
            }
        });
    });
}
```

### Step 2: Replace with GitHub URL

Replace the loadData function with:
```javascript
async function loadData() {
    // GitHub raw URL for the CSV file
    const githubUrl = 'https://raw.githubusercontent.com/varunr89/smokey/main/FW_Veg_Rem_Combined.csv';

    return new Promise((resolve, reject) => {
        Papa.parse(githubUrl, {
            download: true,  // This tells PapaParse to fetch from URL
            header: true,
            dynamicTyping: true,
            skipEmptyLines: true,
            complete: function(results) {
                rawData = results.data;
                console.log(`Successfully loaded ${rawData.length} records from GitHub`);
                resolve();
            },
            error: function(error) {
                reject(new Error('Could not load data from GitHub. ' + error.message));
            }
        });
    });
}
```

**Verification Step 1:** Open `index.html` in browser
- Expected: Dashboard loads data from GitHub URL
- Expected: Console shows "Successfully loaded X records from GitHub"
- Expected: Dashboard functions normally with all charts
- Expected: Network tab shows request to `raw.githubusercontent.com`

**Commit Step 1:**
```bash
git add index.html
git commit -m "Update dashboard to load CSV from GitHub URL

- Replace local file path with GitHub raw content URL
- Point to main branch for stable data access
- Add console logging for successful GitHub fetch
- Update error message to reflect GitHub loading
- Enables single-file distribution of dashboard"
```

---

## Task 2: Add Fallback to Local File (Optional Safety)

**Files:**
- **Modify**: `index.html` (loadData function)

### Step 2: Add fallback mechanism for offline/local use

Replace loadData function with fallback logic:
```javascript
async function loadData() {
    // Primary: GitHub raw URL
    const githubUrl = 'https://raw.githubusercontent.com/varunr89/smokey/main/FW_Veg_Rem_Combined.csv';
    // Fallback: Local file
    const localPath = 'FW_Veg_Rem_Combined.csv';

    return new Promise((resolve, reject) => {
        // Try GitHub first
        Papa.parse(githubUrl, {
            download: true,
            header: true,
            dynamicTyping: true,
            skipEmptyLines: true,
            complete: function(results) {
                if (results.data && results.data.length > 0) {
                    rawData = results.data;
                    console.log(`✓ Loaded ${rawData.length} records from GitHub`);
                    resolve();
                } else {
                    console.warn('GitHub returned empty data, trying local file...');
                    tryLocalFile(resolve, reject);
                }
            },
            error: function(error) {
                console.warn('GitHub fetch failed:', error.message);
                console.log('Attempting to load from local file...');
                tryLocalFile(resolve, reject);
            }
        });
    });

    function tryLocalFile(resolve, reject) {
        Papa.parse(localPath, {
            download: true,
            header: true,
            dynamicTyping: true,
            skipEmptyLines: true,
            complete: function(results) {
                if (results.data && results.data.length > 0) {
                    rawData = results.data;
                    console.log(`✓ Loaded ${rawData.length} records from local file`);
                    resolve();
                } else {
                    reject(new Error('Both GitHub and local file loading failed.'));
                }
            },
            error: function(error) {
                reject(new Error('Could not load data from GitHub or local file. ' + error.message));
            }
        });
    }
}
```

**Verification Step 2:** Test both scenarios
- **Scenario A (Online)**: Open HTML with internet
  - Expected: Loads from GitHub, console shows "✓ Loaded X records from GitHub"
- **Scenario B (Offline)**: Open HTML without internet (or disconnect after page load)
  - Expected: Falls back to local file if present
  - Expected: Console shows warning then "✓ Loaded X records from local file"
- **Scenario C (No local file)**: Remove CSV, test online
  - Expected: Still works, loads from GitHub

**Commit Step 2:**
```bash
git add index.html
git commit -m "Add fallback to local CSV file for offline use

- Try GitHub URL first for latest data
- Fall back to local file if GitHub fails (offline use)
- Add console warnings to explain loading source
- Enables both online (single-file) and offline (with CSV) modes
- Improves resilience for network issues"
```

---

## Task 3: Update Loading Messages

**Files:**
- **Modify**: `index.html` (updateLoadingText calls)

### Step 3: Update loading screen messages to reflect GitHub fetch

In the `initDashboard` function, update the loading messages:

```javascript
async function initDashboard() {
    try {
        updateLoadingText('Loading wildfire data...', 'Fetching from GitHub (55K records)');
        await loadData();

        updateLoadingText('Processing data...', 'Cleaning and validating records');
        cleanData();

        updateLoadingText('Preparing visualizations...', 'Almost ready!');
        initializeFilters();

        // Initial chart render
        setTimeout(() => {
            renderAllCharts();
        }, 100);

        // Hide loading, show dashboard
        setTimeout(() => {
            document.getElementById('loading').classList.add('hidden');
            document.getElementById('dashboard').classList.remove('hidden');
        }, 500);

    } catch (error) {
        showError(error);
    }
}
```

Update the error message in `showError` function:

```javascript
function showError(error) {
    document.getElementById('loading').innerHTML = `
        <div class="text-center p-8 max-w-2xl">
            <h2 class="text-2xl font-bold text-red-600 mb-4">⚠️ Error Loading Data</h2>
            <p class="text-gray-700 mb-4">${error.message}</p>
            <div class="text-left bg-gray-50 p-4 rounded mb-4">
                <p class="text-sm text-gray-700 mb-2"><strong>Possible solutions:</strong></p>
                <ul class="text-sm text-gray-600 list-disc list-inside space-y-1">
                    <li>Check your internet connection</li>
                    <li>Ensure GitHub is accessible from your network</li>
                    <li>If offline, download <code class="bg-gray-200 px-1">FW_Veg_Rem_Combined.csv</code> to the same folder</li>
                    <li>Try refreshing the page (Ctrl+R or Cmd+R)</li>
                </ul>
            </div>
            <button onclick="location.reload()" class="bg-blue-500 text-white px-6 py-2 rounded hover:bg-blue-600">
                Retry
            </button>
        </div>
    `;
}
```

**Verification Step 3:** Test error handling
- **Test 1**: Disconnect internet, open HTML without local CSV
  - Expected: Shows error with helpful troubleshooting steps
- **Test 2**: Click "Retry" button
  - Expected: Page reloads and tries again
- **Test 3**: Normal load
  - Expected: Loading messages show "Fetching from GitHub"

**Commit Step 3:**
```bash
git add index.html
git commit -m "Update loading messages for GitHub data fetch

- Change loading text to indicate GitHub fetch
- Update error message with GitHub-specific troubleshooting
- Add helpful solutions list (check internet, download CSV locally)
- Improve error message styling and clarity"
```

---

## Task 4: Update README Documentation

**Files:**
- **Modify**: `README.md` (Quick Start section)

### Step 4: Update README to reflect new distribution model

Replace the "Quick Start" section (around lines 17-30):

**Old version:**
```markdown
## 🚀 Quick Start

**No installation required!** Just follow these 3 simple steps:

1. **Download** both files to the same folder:
   - `index.html` (the dashboard)
   - `FW_Veg_Rem_Combined.csv` (the data)

2. **Double-click** `index.html`

3. **Explore!** Your browser will open the dashboard automatically

That's it! No Python, no command line, no configuration.
```

**New version:**
```markdown
## 🚀 Quick Start

**No installation required!** Just follow these 2 simple steps:

1. **Download** the dashboard:
   - `index.html` (the dashboard)

2. **Double-click** `index.html`

3. **Explore!** Your browser will open and automatically fetch the latest data from GitHub

That's it! No Python, no command line, no configuration. The dashboard always loads the most recent data.

### Offline Use (Optional)

If you need to use the dashboard without internet:
1. Download both `index.html` and `FW_Veg_Rem_Combined.csv` to the same folder
2. The dashboard will automatically use the local file when GitHub is unavailable
```

Add a new section after "Quick Start":

```markdown
## 🌐 How It Works

The dashboard loads data directly from GitHub's servers, which means:

✅ **Always up-to-date**: When the data file is updated on GitHub, the dashboard automatically reflects the changes
✅ **Single file distribution**: Share just the HTML file - no need to bundle the 19MB CSV
✅ **Reduced storage**: Teachers only need to distribute one small file (~80KB)
✅ **Offline fallback**: If you have the CSV locally, it works offline too

**Technical details:**
- Data source: `https://raw.githubusercontent.com/varunr89/smokey/main/FW_Veg_Rem_Combined.csv`
- Load time: 3-6 seconds on typical broadband
- Cache: Browser may cache for faster subsequent loads
```

Update the "System Requirements" section:

```markdown
## 💻 System Requirements

- **Browser**: Chrome 80+, Firefox 75+, Safari 13+, or Edge 80+ (any from 2020 onwards)
- **Operating System**: Windows, Mac, Linux, or ChromeOS
- **Internet**: Required for initial data load (19MB download)
- **Screen**: Best on desktop (1920x1080), works on tablets too

**Bandwidth note**: First load downloads ~19MB of data. Subsequent loads may use browser cache.
```

Update the "Hosting Options" section:

```markdown
## 🗺️ Hosting Options

### Share with Students

**Option 1: Direct HTML File** ⭐ Recommended
- Share just the `index.html` file (via email, Google Drive, or USB)
- Students open it in their browser
- Dashboard automatically fetches latest data from GitHub
- No need to redistribute when data updates

**Option 2: Host on School Website**
- Upload `index.html` to any web server
- Share the URL with students
- Works instantly, always shows current data

**Option 3: GitHub Pages**
1. Fork this repository
2. Enable GitHub Pages in Settings
3. Share the public URL: `https://yourusername.github.io/smokey/`

**Option 4: Offline Bundle** (No internet required)
- Zip folder with both `index.html` and `FW_Veg_Rem_Combined.csv`
- Share via USB drive or local network
- Dashboard uses local file automatically
```

**Verification Step 4:** Review README
- Expected: Quick Start shows 2 steps instead of 3
- Expected: New "How It Works" section explains GitHub loading
- Expected: All hosting options updated for new model
- Expected: Offline use documented as optional

**Commit Step 4:**
```bash
git add README.md
git commit -m "Update README for GitHub-hosted data loading

- Simplify Quick Start to 2 steps (just download HTML)
- Add 'How It Works' section explaining GitHub data fetch
- Update hosting options with recommended approach
- Document offline use as optional fallback
- Emphasize always-up-to-date benefit
- Update bandwidth requirements in System Requirements"
```

---

## Task 5: Add Data Source Attribution

**Files:**
- **Modify**: `index.html` (footer section)

### Step 5: Update footer to show data source and freshness

In the footer (around line 420), update to show GitHub source:

```html
<footer class="mt-12 mb-6 text-center text-sm text-gray-600" role="contentinfo">
    <div class="border-t border-gray-300 pt-6">
        <p class="mb-2">
            <strong>SMOKEY Wildfire Dashboard</strong> | Data: 1991-2015 | 55,367 wildfire incidents
        </p>
        <p class="mb-2">
            Data loaded from: <a href="https://github.com/varunr89/smokey" class="text-blue-600 hover:underline" target="_blank" rel="noopener">GitHub Repository</a>
        </p>
        <p class="mb-2">
            Sources: U.S. Forest Service, NOAA, USGS
        </p>
        <p class="text-xs text-gray-500">
            Dashboard built with Plotly.js, PapaParse, and Tailwind CSS.
            <br>
            Remember: <em>"Only YOU can prevent wildfires!"</em> - Smokey Bear
        </p>
    </div>
</footer>
```

**Verification Step 5:** Open dashboard
- Expected: Footer shows GitHub repository link
- Expected: Link opens in new tab when clicked
- Expected: Link points to https://github.com/varunr89/smokey

**Commit Step 5:**
```bash
git add index.html
git commit -m "Add GitHub data source attribution to footer

- Add clickable link to GitHub repository in footer
- Indicate data is loaded from GitHub
- Open link in new tab with security attributes
- Maintain existing source attributions"
```

---

## Task 6: Test and Document GitHub Branch Strategy

**Files:**
- **Create**: `docs/DATA_UPDATE_GUIDE.md`

### Step 6: Document how to update data on GitHub

Create documentation for maintaining the data:

```markdown
# Data Update Guide

This guide explains how to update the wildfire dataset on GitHub, which will automatically be reflected in all deployed dashboards.

## Current Setup

The dashboard loads data from:
```
https://raw.githubusercontent.com/varunr89/smokey/main/FW_Veg_Rem_Combined.csv
```

**Important**: The URL points to the `main` branch, ensuring stable production data.

## Updating the Dataset

### Option 1: Direct Update (Simple)

1. Navigate to the GitHub repository
2. Click on `FW_Veg_Rem_Combined.csv`
3. Click the "Edit" button (pencil icon)
4. Upload new CSV file
5. Commit changes with descriptive message
6. **Result**: All dashboards automatically use new data on next load

### Option 2: Pull Request (Recommended for significant changes)

1. Create a new branch: `data-update-YYYY-MM-DD`
2. Upload new CSV to the branch
3. Test with dashboard pointing to branch URL temporarily
4. Create pull request to `main`
5. Review and merge
6. **Result**: Controlled update with review process

### Option 3: Versioned Data (Advanced)

For maintaining multiple versions:

```
data/
├── FW_Veg_Rem_Combined.csv          # Latest (symlink or copy)
├── FW_Veg_Rem_Combined_v1.csv       # Historical version 1
├── FW_Veg_Rem_Combined_v2.csv       # Historical version 2
└── FW_Veg_Rem_Combined_latest.csv   # Always current
```

Update dashboard to use `data/FW_Veg_Rem_Combined_latest.csv` for clear versioning.

## Testing Data Updates

Before merging to main:

1. **Upload new CSV to test branch**
2. **Modify dashboard locally** to point to test branch:
   ```javascript
   const githubUrl = 'https://raw.githubusercontent.com/varunr89/smokey/test-branch/FW_Veg_Rem_Combined.csv';
   ```
3. **Open dashboard** and verify:
   - Data loads successfully
   - Record count is correct
   - All charts render properly
   - No console errors
4. **Merge to main** once verified

## Cache Considerations

**Browser Caching:**
- Browsers may cache the CSV for performance
- Users may need to hard refresh (Ctrl+Shift+R) to see updates immediately
- Typical cache duration: 5-60 minutes depending on browser

**CDN Caching (if using GitHub Pages with CDN):**
- May take 5-15 minutes for changes to propagate
- Can bust cache by adding query parameter: `?v=TIMESTAMP`

**Force Refresh in Dashboard:**
Add cache-busting to loadData function:
```javascript
const githubUrl = `https://raw.githubusercontent.com/varunr89/smokey/main/FW_Veg_Rem_Combined.csv?t=${Date.now()}`;
```

## Data Quality Checks

Before updating production data, verify:

- [ ] CSV is properly formatted (commas, quotes)
- [ ] Header row matches expected columns
- [ ] No corrupted fire_size_class values
- [ ] Numeric fields are numbers (not strings)
- [ ] Date fields are consistent
- [ ] File size is reasonable (<50MB)

## Rollback Procedure

If bad data is deployed:

1. **Immediate**: Revert the commit on `main` branch
2. **Quick**: Replace CSV with last known good version
3. **Communicate**: Notify users to hard refresh browsers

## Monitoring

After data update, monitor:
- GitHub repository traffic (Insights → Traffic)
- User reports of loading issues
- Error rates in browser console logs (if analytics enabled)

## Version History

| Date | Version | Records | Changes | Author |
|------|---------|---------|---------|--------|
| 2020-10-06 | 1.0 | 55,367 | Initial dataset | varunr89 |
| TBD | 2.0 | TBD | TBD | TBD |

```

**Verification Step 6:** Review documentation
- Expected: Clear instructions for updating data
- Expected: Multiple strategies documented (simple to advanced)
- Expected: Testing procedures outlined
- Expected: Cache considerations explained

**Commit Step 6:**
```bash
git add docs/DATA_UPDATE_GUIDE.md
git commit -m "Add data update guide for GitHub-hosted CSV

- Document three update strategies (direct, PR, versioned)
- Explain browser and CDN caching behavior
- Provide testing checklist before production updates
- Include rollback procedure for bad data
- Add data quality checks
- Create version history template"
```

---

## Task 7: Final Testing and Commit

**Files:**
- **Verify**: All changes working together

### Step 7: Comprehensive testing

**Test Checklist:**

1. **Online with GitHub data** ✓
   - Open HTML with internet connection
   - Verify data loads from GitHub
   - Check console: "✓ Loaded X records from GitHub"
   - Verify all 7 charts render correctly
   - Test all 5 filters work properly

2. **Offline with local CSV** ✓
   - Place CSV in same folder as HTML
   - Disconnect internet or block GitHub domain
   - Open HTML
   - Verify fallback to local file works
   - Check console: warning + "✓ Loaded X records from local file"

3. **Error handling** ✓
   - Remove local CSV, block GitHub
   - Verify friendly error message appears
   - Verify troubleshooting steps are clear
   - Test "Retry" button reloads page

4. **Documentation** ✓
   - README Quick Start shows 2 steps
   - Footer shows GitHub link
   - All instructions updated

**Manual Testing Steps:**

```bash
# Test 1: GitHub loading (online)
# 1. Open index.html in Chrome
# 2. Open DevTools Network tab
# 3. Filter for "FW_Veg"
# 4. Verify request to raw.githubusercontent.com succeeds
# 5. Verify dashboard loads fully

# Test 2: Local fallback (offline simulation)
# 1. Open index.html
# 2. In DevTools, open Network tab
# 3. Enable "Offline" mode
# 4. Refresh page
# 5. Verify error appears OR local file loads (if CSV present)

# Test 3: Performance check
# 1. Open index.html in Chrome Incognito (no cache)
# 2. Note load time in console
# 3. Verify < 10 seconds on reasonable connection
# 4. Close and reopen - should be faster (cached)
```

**Commit Step 7:**
```bash
git add -A
git commit -m "Complete GitHub-hosted CSV implementation

All tasks completed:
- Task 1: Replace local CSV path with GitHub URL
- Task 2: Add fallback to local file for offline use
- Task 3: Update loading messages and error handling
- Task 4: Update README for new distribution model
- Task 5: Add GitHub attribution to footer
- Task 6: Create data update guide
- Task 7: Comprehensive testing verified

Dashboard now loads data from:
https://raw.githubusercontent.com/varunr89/smokey/main/FW_Veg_Rem_Combined.csv

Benefits:
- Single-file distribution (just HTML needed)
- Always loads latest data automatically
- Offline fallback to local CSV if available
- Simpler for educators (no CSV to manage)
- Easier updates (change GitHub file only)

Tested: Online loading, offline fallback, error handling"
```

**Final Push:**
```bash
git push -u origin claude/deep-thinking-exploration-011CUsRCws1k2YdqV72W4U4n
```

---

## Implementation Summary

### Changes Made

1. **Data Loading**: GitHub URL instead of local file path
2. **Fallback Logic**: Local CSV as backup for offline use
3. **Loading Messages**: Updated to reflect GitHub fetch
4. **Error Handling**: Improved with GitHub-specific troubleshooting
5. **README**: Simplified to 2-step Quick Start, added GitHub benefits
6. **Footer**: Added GitHub repository link
7. **Documentation**: Created comprehensive data update guide

### Benefits

✅ **Single-file distribution**: Teachers only share HTML (80KB vs 19MB)
✅ **Always current**: Dashboard automatically uses latest data
✅ **Easier updates**: Update CSV on GitHub, not redistribute HTML
✅ **Offline capable**: Falls back to local file when needed
✅ **Better for students**: Just open HTML, no file management

### Trade-offs

**Pros:**
- Dramatically simpler distribution
- Always fresh data
- Reduced teacher workload
- Smaller file sizes to share

**Cons:**
- Requires internet on first load (19MB download)
- Slight delay vs local file (~3-6 seconds)
- Depends on GitHub availability (highly reliable)
- Browser cache may delay seeing updates

### Recommendation

This approach is **strongly recommended** for educational use because:
1. Teachers share one tiny HTML file instead of bundling CSV
2. Students always see current data without redistribution
3. Data corrections/updates propagate automatically
4. Offline fallback handles no-internet scenarios

The 3-6 second initial load is acceptable trade-off for the massive distribution benefits.
