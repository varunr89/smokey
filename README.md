# 🔥 SMOKEY Wildfire Dashboard

An interactive, browser-based dashboard for exploring U.S. wildfire patterns from 1991-2015. Designed for educators and the general public with zero technical setup required.

## 📊 Dataset

- **55,367 wildfire incidents** across the United States
- **Time period**: 1991-2015 (25 years)
- **Data sources**:
  - Fire incidents: U.S. Forest Service Fire Program Analysis
  - Weather data: NOAA Integrated Surface Database
  - Vegetation: USGS Land Cover Classification

## 🚀 Quick Start

**No installation required!** Just follow these 3 simple steps:

1. **Download** both files to the same folder:
   - `index.html` (the dashboard)
   - `FW_Veg_Rem_Combined.csv` (the data)

2. **Double-click** `index.html`

3. **Explore!** Your browser will open the dashboard automatically

That's it! No Python, no command line, no configuration.

## 💻 System Requirements

- **Browser**: Chrome 80+, Firefox 75+, Safari 13+, or Edge 80+ (any from 2020 onwards)
- **Operating System**: Windows, Mac, Linux, or ChromeOS
- **Internet**: Required only for initial load (CDN libraries)
- **Screen**: Best on desktop (1920x1080), works on tablets too

## 🎯 Features

### Interactive Filters
- **Geography**: Filter by state or region (West, South, Northeast, Midwest)
- **Time**: Dual-slider for year range selection (1991-2015)
- **Cause**: View all fires, human-caused, natural, or specific causes
- **Size**: Filter by fire size class (A-G)

### Visualizations

**Weather & Fire Risk**
- Temperature trends before fires (4 time windows)
- Humidity and precipitation patterns
- Identify drought conditions

**Temporal Patterns**
- Seasonal heatmap showing month-year fire frequency
- Year-over-year trends with cause breakdown
- Peak fire season identification

**Fire Causes**
- Donut chart with human vs. natural classification
- Dynamic prevention tips based on top cause
- Educational messaging

**Geographic Overview**
- Interactive U.S. map colored by fire density
- Click states to filter entire dashboard
- Hover for detailed state statistics

### Educational Features
- Collapsible info panels (About Data, How to Use, Fire Classes, Prevention)
- Auto-generated insights (top state, peak month, human-caused %)
- Fire size class definitions (A-G)
- Smokey Bear prevention tips

## 📚 For Educators

### Classroom Use

This dashboard is perfect for:
- **Science classes**: Climate, ecology, weather patterns
- **Geography**: Regional patterns, state comparisons
- **Statistics**: Data analysis, visualization interpretation
- **Environmental studies**: Human impact, conservation

### Learning Objectives

Students can:
- Identify seasonal fire patterns (peak in March-April)
- Understand human vs. natural fire causes (70% human-caused)
- Explore weather correlations (temperature, humidity, precipitation)
- Analyze geographic hotspots (Southern states)
- Practice data filtering and hypothesis testing

### Activities

1. **Pattern Discovery**: Have students find the month with most fires
2. **Regional Comparison**: Compare fire patterns between regions
3. **Cause Analysis**: Identify top causes and brainstorm prevention
4. **Weather Investigation**: Explore how temperature affects fire size
5. **Presentation**: Export charts for reports (📷 Save button)

## 🛠️ Usage Tips

### Exploring the Data
1. Start with no filters to see the full dataset
2. Use the filter bar to focus on specific states, time periods, or causes
3. Click on chart elements to filter (e.g., click a state on the map)
4. Reset filters anytime with the "Reset All Filters" button

### Downloading Charts
- Click the "📷 Save" button on any chart
- Charts download as PNG images (1200x600px)
- Perfect for reports, presentations, or posters

### Understanding Missing Data
- Weather charts show a warning: "Based on fires with weather data"
- 25.7% of records have missing weather measurements
- This is normal for historical data from older weather stations

### Performance
- Initial load takes 2-4 seconds (loading 55K records)
- Filter changes update charts in <500ms
- If slow, try reducing the year range or selecting a specific state

## 🗺️ Hosting Options

### Share with Students

**Option 1: Local Files**
- Zip the folder (index.html + CSV)
- Share via email, Google Drive, or USB
- Students unzip and open index.html

**Option 2: GitHub Pages**
1. Create a GitHub repository
2. Upload both files
3. Enable GitHub Pages in Settings
4. Share the public URL

**Option 3: School Server**
- Upload both files to any web server
- No server-side processing needed
- Works instantly

## 📖 Data Dictionary

### Fire Attributes
- `fire_name`: Name/identifier of fire incident
- `fire_size`: Area burned in acres
- `fire_size_class`: Classification A-G (A: <0.25 acres, G: >5000 acres)
- `stat_cause_descr`: Cause (Arson, Lightning, Debris Burning, etc.)
- `discovery_month`: Month fire was discovered
- `disc_pre_year`: Year of discovery

### Weather Attributes (4 time windows)
- `Temp_pre_X`: Temperature (°C) X days before fire
- `Wind_pre_X`: Wind speed (m/s) X days before fire
- `Hum_pre_X`: Humidity (%) X days before fire
- `Prec_pre_X`: Precipitation (mm) X days before fire
- `*_cont`: Same measurements on containment day

### Geographic Attributes
- `state`: U.S. state abbreviation
- `latitude`, `longitude`: Fire coordinates
- `remoteness`: Distance to nearest city (0-1, normalized)

### Ecological
- `Vegetation`: Land cover type (1-28 classification)

## ⚠️ Data Limitations

- **Missing weather data**: 25.7% of records have incomplete weather measurements (-1.0 values)
- **Missing containment dates**: Only 46.3% include when fire was contained
- **Data quality**: Some records (~0.36%) have corrupted fire size classifications
- **Geographic bias**: Southern/Eastern states may be overrepresented

## 🔥 Fire Prevention

**Remember**: 70% of wildfires are human-caused and preventable!

- Never leave campfires unattended
- Check for burn bans before burning debris
- Don't park vehicles on dry grass
- Properly dispose of cigarettes
- Teach children about fire safety

**"Only YOU can prevent wildfires!"** - Smokey Bear

## 📄 License & Attribution

### Data Sources
- Fire data: U.S. Forest Service (public domain)
- Weather data: NOAA (public domain)
- Vegetation data: USGS (public domain)

### Dashboard
- Built with [Plotly.js](https://plotly.com/javascript/) (MIT License)
- CSV parsing: [PapaParse](https://www.papaparse.com/) (MIT License)
- Styling: [Tailwind CSS](https://tailwindcss.com/) (MIT License)

Created for educational purposes. Free to use and modify.

## 🐛 Troubleshooting

**Dashboard won't load**
- Ensure `FW_Veg_Rem_Combined.csv` is in the same folder as `index.html`
- Check browser console (F12) for error messages
- Try a different browser (Chrome recommended)

**Charts look broken**
- Make sure you have internet for initial load (CDN libraries)
- Update to a modern browser (2020 or newer)
- Try refreshing the page (Ctrl+R or Cmd+R)

**Slow performance**
- Use a more powerful computer (recommended: 4GB RAM)
- Filter to a smaller dataset (single state or shorter year range)
- Close other browser tabs

**Data seems wrong**
- Remember: filters apply to ALL charts simultaneously
- Click "Reset All Filters" to return to full dataset
- Check the "Showing: X fires" count in the filter bar

## 📧 Questions or Feedback?

This dashboard was created to make wildfire data accessible for education. If you have questions, find bugs, or have suggestions for improvement, please open an issue on GitHub!

---

**Happy exploring! 🔥📊**
