# Wildfire Dashboard Design
**Date:** 2025-11-07
**Target Audience:** General Public / Educators
**Implementation:** Pure HTML/JavaScript (Zero Installation)

---

## Overview

A single-file interactive dashboard for exploring the Smokey wildfire dataset (55,367 fires, 1991-2015). Designed for educational use with minimal technical barriers - users simply download and open an HTML file in their browser.

## Design Goals

1. **Zero Installation**: No Python, Node.js, or package managers required
2. **Educational Focus**: Clear visualizations with contextual information and insights
3. **Accessibility**: Works for non-technical users (teachers, students, general public)
4. **Interactive Exploration**: Powerful filtering without overwhelming complexity
5. **Mobile-Friendly**: Responsive design for tablets and desktops

## Target Audience

**Primary:** Educators and general public seeking to understand wildfire patterns
**Use Cases:**
- Classroom teaching about climate, ecology, fire prevention
- Public awareness and education
- Student research projects
- Policy discussions with visual evidence

## Technical Architecture

### Core Technology Stack

**Single HTML File Approach:**
- File size: ~600-800 lines of HTML/CSS/JavaScript
- All dependencies loaded via CDN (no local installation)
- Data file sits alongside HTML (same directory)

**Libraries (CDN):**
- **Plotly.js** - Interactive visualizations with built-in zoom, pan, hover
- **PapaParse** - Fast CSV parsing (handles 19MB file efficiently)
- **Tailwind CSS** - Modern styling framework
- **Vanilla JavaScript** - No frameworks, keeps it simple

**Browser Compatibility:**
- Minimum: Chrome 80+, Firefox 75+, Safari 13+, Edge 80+ (2020+)
- Optimized for desktop (1920x1080)
- Responsive down to tablets (768px)
- Simplified single-column layout for mobile

### Data Loading Strategy

```
Page Load Flow:
1. Fetch FW_Veg_Rem_Combined.csv from same directory
2. Parse with PapaParse (~2-3 seconds for 55K records)
3. Clean data in memory:
   - Filter corrupted records (fire_size_class with state codes)
   - Convert strings to numbers
   - Handle missing values (-1.0 → null/excluded)
   - Parse dates
   - Create derived fields (region, season, human_vs_natural)
4. Store in JavaScript array for fast filtering
5. Render dashboard with smooth fade-in animation
```

**Performance Optimizations:**
- Debounced filter updates (300ms delay on slider drag)
- Lazy rendering for charts below fold (Intersection Observer)
- Immutable original data, lightweight filtered views
- Optional sampling for scatter plots if >10K points

### File Structure

```
smokey-dashboard/
├── index.html                    # Complete dashboard (self-contained)
├── FW_Veg_Rem_Combined.csv      # Data (18.9 MB)
└── README.md                     # Usage instructions (optional)
```

## Dashboard Layout

### Filter Bar (Sticky Header)

Horizontal control panel that remains visible during scrolling:

```
┌─────────────────────────────────────────────────────────────┐
│  🔥 SMOKEY Wildfire Dashboard (1991-2015)                   │
├─────────────────────────────────────────────────────────────┤
│  📍 Geography: [All States ▼] [Region: All ▼]              │
│  📅 Year Range: [1991 ════●────────●═══ 2015]              │
│  🌡️  Fire Cause: [All Causes ▼]                            │
│  📏 Fire Size: [All Sizes ▼] (A-G classes)                 │
│  [Reset All Filters] 📊 Showing: 55,367 fires              │
└─────────────────────────────────────────────────────────────┘
```

**Filter Components:**

1. **Geography Selector** (Two-level)
   - **State Dropdown**: Alphabetical (AL, AK, AZ...) + "All States" default
   - **Region Dropdown**: West, South, Northeast, Midwest, All Regions
   - Work together: select region OR drill to specific state

2. **Year Range Slider**
   - Dual-handle slider (1991-2015)
   - Select any continuous range
   - Shows distribution histogram underneath

3. **Fire Cause Dropdown**
   - All Causes (default)
   - Human-Caused (aggregates Arson, Debris, Equipment, etc.)
   - Natural (Lightning only)
   - Individual causes (expandable list)

4. **Fire Size Class**
   - Multi-select checkboxes for A, B, C, D, E, F, G
   - Shows count for each class

5. **Active Filter Display**
   - Live count: "Showing: 2,341 fires (Texas, 2005-2010, Arson)"
   - Reset all button clears to full dataset

**Interactive Behavior:**
- All charts update instantly when filters change
- Cross-filtering: Charts can filter each other (click map → filter by state)
- Debounced for performance (300ms)

### Visualization Grid

**Priority Order (per user requirements):**
1. Weather & Fire Risk
2. Temporal Patterns
3. Fire Causes

**Layout: 4 Rows**

---

#### **Row 1: Weather & Fire Risk** (Priority #1)

```
┌─────────────────────────────────┬─────────────────────────────┐
│ Temperature Before Fires        │ Humidity & Precipitation    │
│ (Multi-line chart)              │ (Dual-axis chart)           │
└─────────────────────────────────┴─────────────────────────────┘
```

**Chart 1A: Temperature Before Fires**
- **Type**: Multi-line chart
- **X-axis**: Time windows (-30 days, -15 days, -7 days, containment)
- **Y-axis**: Average temperature (°C)
- **Lines**: Grouped by fire size class (A-G) with color coding
- **Insight**: Shows if higher pre-fire temps correlate with larger fires
- **Missing Data**: Exclude records with -1.0 values, show badge "Based on 74.3% of fires with weather data"

**Chart 1B: Humidity & Precipitation**
- **Type**: Dual-axis combination chart
- **Left Y-axis**: Humidity (%) - Bar chart
- **Right Y-axis**: Precipitation (mm) - Line chart
- **X-axis**: Same time windows as temperature
- **Insight**: Visualize drought conditions (low humidity + low precip = high risk)
- **Color**: Blues for humidity, greens for precipitation

---

#### **Row 2: When Fires Strike** (Priority #2)

```
┌─────────────────────────────────┬─────────────────────────────┐
│ Seasonal Heatmap                │ Year-over-Year Trends       │
│ (Calendar heatmap)              │ (Line chart)                │
└─────────────────────────────────┴─────────────────────────────┘
```

**Chart 2A: Seasonal Heatmap**
- **Type**: Calendar-style heatmap
- **X-axis**: Months (Jan-Dec)
- **Y-axis**: Years (1991-2015)
- **Color**: Fire count intensity (gradient from light to dark red)
- **Interaction**: Click cell → filter dashboard to that month/year
- **Insight**: Immediately see March-April peaks, year-by-year patterns

**Chart 2B: Year-over-Year Trends**
- **Type**: Multi-line time series
- **X-axis**: Year (1991-2015)
- **Y-axis**: Fire count
- **Lines**: Total fires + separate lines for top 3 causes (stacked or grouped)
- **Insight**: Shows activity trends, policy impacts, climate cycles
- **Interaction**: Click legend to toggle lines on/off

---

#### **Row 3: Fire Causes & Characteristics** (Priority #3)

```
┌─────────────────────────────────┬─────────────────────────────┐
│ What Starts Fires?              │ Fire Size Distribution      │
│ (Donut chart)                   │ (Histogram)                 │
└─────────────────────────────────┴─────────────────────────────┘
```

**Chart 3A: What Starts Fires?**
- **Type**: Donut chart with center text
- **Segments**: Cause categories with percentages
- **Color**: Human causes (reds/oranges), Natural (blues), Unknown (grays)
- **Center Text**: "70% Human-Caused"
- **Interaction**:
  - Hover → Show prevention tips tooltip
  - Click segment → Filter entire dashboard by cause
- **Educational**: Each segment links to prevention messaging

**Chart 3B: Fire Size Distribution**
- **Type**: Histogram (log scale)
- **X-axis**: Fire size (acres) - logarithmic bins
- **Y-axis**: Count of fires
- **Annotation**: Mean line, median line
- **Insight**: Most fires are small (0.5-10 acres), few are catastrophic (>100K)
- **Color**: Gradient by size class (A=green → G=red)

---

#### **Row 4: Geographic Overview** (Full Width)

```
┌───────────────────────────────────────────────────────────────┐
│ U.S. Map - Fire Density by State                             │
│ (Choropleth map)                                              │
└───────────────────────────────────────────────────────────────┘
```

**Chart 4: Interactive U.S. Map**
- **Type**: Choropleth map (Plotly's built-in US map)
- **Color**: States shaded by fire count (white → dark red gradient)
- **Hover**: Tooltip shows:
  - State name
  - Total fires
  - Average fire size
  - Top cause
- **Interaction**: Click state → Apply geographic filter to all charts
- **Legend**: Color scale with numerical ranges

---

## Chart Interactivity Features

**All Charts Support:**
- **Linked Filtering**: Click any element → updates entire dashboard
- **Hover Tooltips**: Detailed data with context
- **Zoom & Pan**: Enabled on time series and map
- **Download**: Each chart has "📷 Save as PNG" button
- **Reset**: Double-click to reset zoom

**Cross-Chart Linking Examples:**
- Click Texas on map → All charts show Texas data only
- Click "Arson" in donut → Weather charts show arson fire patterns
- Click March cell in heatmap → All charts filter to March fires

## Educational Features

### Information Panels (Collapsible)

1. **"About This Data"**
   - Dataset description (55K fires, 1991-2015)
   - Data sources (US Forest Service, NOAA, USGS)
   - Limitations and caveats

2. **"How to Use This Dashboard"**
   - Quick tutorial with screenshots
   - Filter examples
   - Tips for exploration

3. **"Understanding Fire Size Classes"**
   - Class A-G definitions
   - Visual examples of what each size means
   - Containment challenges by class

4. **"Fire Prevention Tips"**
   - Actionable advice based on top causes
   - Links to Smokey Bear resources
   - Seasonal safety reminders

### Auto-Generated Insights

**Dynamic Text Summaries:**
- "In Texas during 2006, 342 fires occurred, mostly from debris burning."
- "Peak fire danger: March-April when temperatures rise but humidity remains low"
- "70% of fires are human-caused and preventable"

**"Did You Know?" Callouts:**
- Random interesting facts from the data
- Updates when filters change
- Examples:
  - "The largest fire (606,945 acres) occurred in [year/state]"
  - "Lightning causes only 15% of fires but they tend to be larger"

### Accessibility Features

- **Keyboard Navigation**: Tab through all filters and controls
- **Screen Reader Labels**: ARIA labels on all interactive elements
- **High Contrast Mode**: Toggle button for better visibility
- **Text Size Controls**: +/- buttons to adjust font size
- **Color-Blind Safe Palette**: Using Plotly's default accessible colors
- **Alt Text**: Descriptive text for all visualizations

## Data Processing & Quality

### Data Cleaning Pipeline

Runs once on page load:

```javascript
1. Filter corrupted records:
   - Remove rows where fire_size_class contains state codes (" IA", " IOWA")
   - ~200 records affected (0.36% of dataset)

2. Type conversions:
   - fire_size, temperatures, wind, humidity → floats
   - disc_pre_year → integer
   - dates → Date objects

3. Handle missing weather data:
   - -1.0 values → null (excluded from averages)
   - Track count for data quality badges
   - 25.7% of records affected

4. Parse temporal data:
   - disc_pre_year + discovery_month → Date object
   - Extract: year, month, season, decade

5. Create derived fields:
   - region: Map state → West/South/Northeast/Midwest
   - season: Map month → Winter/Spring/Summer/Fall
   - human_vs_natural: Boolean flag based on cause
   - fire_size_bucket: Small/Medium/Large/Catastrophic
```

### Missing Data Strategy

**Weather Data (25.7% missing):**
- **Display**: Show data quality badge on weather charts
  - "⚠️ Based on 74.3% of fires with complete weather data"
- **Calculation**: Exclude -1.0 values from all averages/aggregations
- **Optional**: Toggle button "Show only fires with complete weather data"
- **Transparency**: Explain in "About This Data" panel

**Containment Dates (53.7% missing):**
- **Impact**: Can't analyze containment time for half the dataset
- **Approach**: Show count of "fires with containment data" when relevant
- **Charts**: Avoid putout_time visualizations due to sparsity

### Performance Considerations

**Large Dataset Handling:**
- 55K records is manageable for modern browsers
- If filtered set >10K points on scatter plots, consider sampling
- Always show full counts/aggregations (never sample those)

**Memory Management:**
- Keep original parsed data immutable
- Filtered views are lightweight references
- Clear old chart objects before re-rendering

**Loading States:**
- Show spinner during CSV load (2-3 seconds)
- Progress bar during parsing
- Smooth fade-in when ready

## User Experience & Error Handling

### Loading Experience

```
Timeline:
0-2s:  Animated fire icon + "Loading wildfire data..."
2-3s:  Progress bar: "Parsing 55,367 records..."
3-4s:  "Preparing visualizations..."
4s:    Fade in dashboard with smooth animation
```

**Error Handling:**

**CSV Not Found:**
```
Error Message:
"Could not load data file. Please ensure FW_Veg_Rem_Combined.csv
is in the same folder as this HTML file."

[Retry Button] [Download Data Button]
```

**Browser Too Old:**
```
"Your browser may not support this dashboard.
Please update to Chrome 80+, Firefox 75+, Safari 13+, or Edge 80+."
```

**Parse Error:**
```
"Data file appears corrupted. Please re-download from GitHub."
```

### Edge Cases

1. **No Results After Filtering**
   - Message: "No fires match your filters. Try expanding your selection."
   - Show suggestion: "Try: [Reset All Filters]"

2. **Single Data Point**
   - Adapt charts: Show as text statistic instead of empty graph
   - Example: "Only 1 fire matches your filters: [details]"

3. **Slow Performance**
   - Show "Processing..." overlay during heavy filtering
   - Debounce slider inputs (300ms)
   - Consider asking user to reduce filter range

4. **Offline Use**
   - Works if CSV is local
   - CDN libraries require internet (acceptable trade-off)
   - Future option: Bundle CDN into HTML for full offline

## Deployment & Distribution

### For Educators

**Download Package:**
1. Visit GitHub repository
2. Download 2 files:
   - `index.html` (dashboard)
   - `FW_Veg_Rem_Combined.csv` (data)
3. Save to same folder
4. Double-click `index.html`
5. Browser opens → Dashboard loads automatically

**No Installation Required:**
- No command line
- No Python/Node.js
- No package managers
- No configuration files

### Hosting Options

**GitHub Pages (Public URL):**
1. Push files to repository
2. Settings → Pages → Enable
3. Get shareable link: `https://username.github.io/smokey`
4. Share with students/colleagues

**Local Sharing:**
- Zip folder → Share via email/USB drive
- Works on any computer with modern browser
- Great for classrooms without internet

**School/University Servers:**
- Upload to any web server
- Works instantly (static files)
- No server-side processing needed

### System Requirements

**Minimum:**
- Modern browser (Chrome 80+, Firefox 75+, Safari 13+, Edge 80+)
- 2GB RAM (for 19MB CSV in memory)
- 1024x768 screen resolution

**Recommended:**
- Latest browser version
- 4GB RAM
- 1920x1080 display
- Broadband internet (for initial CDN load)

**Supported Platforms:**
- Windows 10/11
- macOS 10.15+
- Linux (any distribution)
- ChromeOS
- iPad/Android tablets (simplified layout)

## Visual Design Principles

### Color Palette

**Fire-Themed Gradients:**
- **Fire Intensity**: Yellow → Orange → Red → Dark Red
- **Temperature**: Blue (cool) → Red (hot)
- **Vegetation**: Greens for forests, browns for shrubland
- **Causes**: Reds for human, blues for natural, grays for unknown

**Accessibility:**
- Color-blind safe (using Plotly defaults)
- High contrast mode available
- Never rely on color alone (use shapes/patterns too)

### Typography

- **Headers**: Sans-serif, bold, 24-32px
- **Body**: Sans-serif, 14-16px
- **Chart Labels**: 12-14px
- **Tooltips**: 13px with clear hierarchy

### Spacing & Layout

- **Grid Gap**: 20px between charts
- **Padding**: 16px inside panels
- **Responsive Breakpoints**:
  - Desktop: 1920px (2x2 grid)
  - Tablet: 768px (1x4 stacked)
  - Mobile: 480px (simplified)

## Future Enhancements (Out of Scope for v1)

**Potential Additions:**
- Compare multiple states side-by-side
- Animated time-lapse map (fires spreading over years)
- Export filtered data as CSV for classroom exercises
- Embed YouTube videos about fire prevention
- Quiz mode for students
- Advanced weather correlations (wind direction, humidity×temp)
- Integration with real-time fire data APIs
- Vegetation type explorer (photos of each type)

**Community Features:**
- Share custom filter configurations via URL parameters
- Bookmark interesting findings
- Teacher lesson plan templates
- Student worksheet generator

## Success Metrics

**User Experience:**
- Page loads in <5 seconds on average internet
- All interactions respond in <500ms
- Works on 95%+ of modern browsers
- Mobile-friendly (tablet minimum)

**Educational Impact:**
- Users can answer: "When do fires happen most?"
- Users can identify: "What causes most fires?"
- Users understand: "How weather affects fire risk"
- Teachers can build lessons around the data

## Implementation Checklist

- [ ] Set up HTML structure with Tailwind CSS
- [ ] Integrate Plotly.js and PapaParse from CDN
- [ ] Build data loading and cleaning pipeline
- [ ] Create filter bar with all 5 controls
- [ ] Implement cross-filter logic
- [ ] Build Row 1: Weather charts (temp, humidity/precip)
- [ ] Build Row 2: Temporal charts (heatmap, trends)
- [ ] Build Row 3: Cause + size distribution charts
- [ ] Build Row 4: Interactive U.S. map
- [ ] Add educational info panels
- [ ] Implement auto-generated insights
- [ ] Add accessibility features (keyboard nav, ARIA labels)
- [ ] Error handling and edge cases
- [ ] Loading states and animations
- [ ] Performance optimizations
- [ ] Mobile responsive adjustments
- [ ] Cross-browser testing
- [ ] Documentation (README.md)
- [ ] Final polish and UX refinements

## Conclusion

This dashboard design balances **simplicity** (zero installation) with **power** (5 filters, 7 interactive charts) to create an educational tool that makes wildfire data accessible to non-technical audiences. By focusing on the story (Weather → Time → Causes) and using familiar interaction patterns (dropdowns, sliders, click-to-filter), educators can guide students through complex data without overwhelming them.

The single-file HTML approach eliminates all technical barriers while modern JavaScript libraries provide professional-grade visualizations. The result is a tool that "just works" for classrooms, presentations, and self-guided exploration.
