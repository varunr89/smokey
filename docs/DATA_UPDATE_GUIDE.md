# 📊 Data Update Guide

This guide explains how to update the wildfire dataset (`FW_Veg_Rem_Combined.csv`) that powers the SMOKEY dashboard.

## 🌐 How the Dashboard Loads Data

The dashboard loads data from GitHub using this URL:
```
https://raw.githubusercontent.com/varunr89/smokey/main/FW_Veg_Rem_Combined.csv
```

**Key Points:**
- Updates to the CSV on GitHub are automatically reflected in the dashboard
- Users don't need to re-download the HTML file when data changes
- The dashboard falls back to a local file if GitHub is unavailable
- Browser and CDN caching may delay updates (see Cache Considerations below)

## 🔄 Three Update Strategies

### Strategy 1: Direct Update (Recommended for Minor Changes)

**Use when:**
- Adding new records for recent years
- Fixing data quality issues
- Making minor corrections

**Steps:**
1. Prepare your updated CSV file locally
2. Run data quality checks (see checklist below)
3. Upload to GitHub:
   ```bash
   git pull origin main
   git add FW_Veg_Rem_Combined.csv
   git commit -m "Update wildfire data: [describe change]"
   git push origin main
   ```
4. Test the dashboard (see Testing Procedures below)
5. Monitor for user feedback over 24-48 hours

### Strategy 2: Pull Request Workflow (Recommended for Major Changes)

**Use when:**
- Restructuring the dataset schema
- Adding new attributes
- Merging data from new sources
- Changes that may require dashboard updates

**Steps:**
1. Create a new branch:
   ```bash
   git checkout -b data-update-YYYY-MM-DD
   ```
2. Update the CSV file
3. Test locally with the dashboard
4. Push and create a pull request:
   ```bash
   git add FW_Veg_Rem_Combined.csv
   git commit -m "Update wildfire data: [describe change]"
   git push origin data-update-YYYY-MM-DD
   gh pr create --title "Data Update: [description]" --body "..."
   ```
5. Review changes in the PR
6. Merge to main when ready

### Strategy 3: Versioned Releases (For Production Environments)

**Use when:**
- Distributing to large audiences
- Need rollback capability
- Want to pin to specific data versions

**Steps:**
1. Update the CSV on main branch
2. Create a Git tag:
   ```bash
   git tag -a v1.1 -m "Data update: Added 2016-2020 wildfire records"
   git push origin v1.1
   ```
3. Update the dashboard to reference the tagged version:
   ```javascript
   const githubUrl = 'https://raw.githubusercontent.com/varunr89/smokey/v1.1/FW_Veg_Rem_Combined.csv';
   ```
4. Commit the dashboard change separately

## ✅ Data Quality Checklist

Before updating the CSV on GitHub, verify:

**File Structure:**
- [ ] CSV has exactly 43 columns (matching current schema)
- [ ] Header row matches existing column names
- [ ] No empty rows or malformed records
- [ ] File size is reasonable (current: ~19MB for 55K records)

**Data Integrity:**
- [ ] All required fields are populated (fire_name, state, disc_pre_year, etc.)
- [ ] Missing weather data uses `-1.0` (not null, empty, or other values)
- [ ] fire_size_class contains only A-G values (or missing)
- [ ] Years are in valid range (1991-2025 or expected range)
- [ ] States are valid 2-letter abbreviations
- [ ] Coordinates are within U.S. bounds (lat: 25-50, lon: -125 to -65)

**Test Loading:**
- [ ] Open `index.html` locally with the new CSV in the same folder
- [ ] Verify all charts render correctly
- [ ] Check filter functionality (geography, year range, cause, size)
- [ ] Look for JavaScript console errors (F12 → Console)
- [ ] Verify the count: "Showing: X fires" matches expected total

**Documentation:**
- [ ] Update README.md if record count or year range changed
- [ ] Update `Wildfire_att_description.txt` if new attributes added
- [ ] Document the change in commit message

## 🧪 Testing Procedures

### Test 1: GitHub Loading (Online)

1. Push the updated CSV to GitHub
2. Open the dashboard in an **incognito/private window** (to bypass cache):
   - Chrome: Ctrl+Shift+N (Windows) or Cmd+Shift+N (Mac)
   - Firefox: Ctrl+Shift+P (Windows) or Cmd+Shift+P (Mac)
3. Open browser console (F12 → Console tab)
4. Look for: `✓ Loaded X records from GitHub`
5. Verify the record count matches your updated dataset
6. Test all visualizations and filters

### Test 2: Local Fallback (Offline)

1. Download the updated CSV from GitHub
2. Place it in the same folder as `index.html`
3. Disconnect from the internet (or block GitHub in hosts file)
4. Open `index.html` in a new browser window
5. Look for: `✓ Loaded X records from local file`
6. Verify dashboard works identically to online version

### Test 3: Error Handling

1. Temporarily rename the local CSV file
2. Block GitHub using browser dev tools (Network tab → Offline mode)
3. Open `index.html`
4. Verify the error message displays with troubleshooting steps
5. Restore network and verify the "Retry" button works

### Test 4: Cross-Browser Compatibility

Test in all major browsers:
- [ ] Chrome/Chromium (80+)
- [ ] Firefox (75+)
- [ ] Safari (13+)
- [ ] Edge (80+)

Look for:
- Chart rendering issues
- Filter functionality
- Performance (charts should update in <500ms)

## ⏱️ Cache Considerations

**Browser Cache:**
- GitHub serves raw files with cache headers
- Browsers may cache the CSV for several minutes to hours
- Users may need to hard refresh: Ctrl+Shift+R (Windows) or Cmd+Shift+R (Mac)

**CDN Cache:**
- GitHub's CDN may cache the raw file
- Updates can take 5-15 minutes to propagate globally
- Be patient after pushing updates

**Cache Busting (Advanced):**
If immediate updates are critical, add a version parameter:
```javascript
const version = '2025-11-07'; // Update this when data changes
const githubUrl = `https://raw.githubusercontent.com/varunr89/smokey/main/FW_Veg_Rem_Combined.csv?v=${version}`;
```

**Important:** If using cache busting, you must also update `index.html` and distribute it to users.

## 🔙 Rollback Procedure

If a bad data update is pushed:

**Quick Rollback (Revert Commit):**
```bash
# Find the commit hash of the bad update
git log --oneline

# Revert the commit (creates a new commit that undoes changes)
git revert <commit-hash>
git push origin main
```

**Full Rollback (Reset to Previous State):**
```bash
# Find the last good commit
git log --oneline

# Reset to that commit
git reset --hard <good-commit-hash>

# Force push (WARNING: Use with caution)
git push --force origin main
```

**Using Tagged Versions:**
If you're using Strategy 3 (Versioned Releases), simply update the dashboard to reference the previous tag:
```javascript
const githubUrl = 'https://raw.githubusercontent.com/varunr89/smokey/v1.0/FW_Veg_Rem_Combined.csv';
```

## 📞 Communication

When making significant data updates:

**Notify Users:**
- Update the README.md with the change
- Add a note to the GitHub repository's Releases page
- If users are in a classroom setting, inform the instructor

**Document Changes:**
Create a `CHANGELOG.md` entry:
```markdown
## [1.1.0] - 2025-11-07
### Changed
- Added 5,234 wildfire records for years 2016-2020
- Updated temperature data source to NOAA v2.5
- Fixed 127 records with incorrect state abbreviations
```

## 🚨 Emergency Contacts

If you encounter issues during a data update:

1. **Data Loading Errors:**
   - Check GitHub's status: https://www.githubstatus.com/
   - Verify file permissions: CSV should be public in a public repo
   - Test the raw URL directly in a browser

2. **Dashboard Breaks After Update:**
   - Roll back immediately (see Rollback Procedure)
   - Check browser console for JavaScript errors
   - Verify CSV structure matches schema (43 columns)

3. **Performance Issues:**
   - Monitor file size (should be <50MB for browser loading)
   - Consider filtering to a smaller time range
   - Test with a smaller dataset first

## 📚 Additional Resources

- **Papa Parse Documentation**: https://www.papaparse.com/docs
- **GitHub Raw URLs**: https://docs.github.com/en/repositories/working-with-files/using-files/downloading-source-code-archives
- **Browser Caching**: https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching

---

**Last Updated:** 2025-11-07
**Maintained by:** varunr89
