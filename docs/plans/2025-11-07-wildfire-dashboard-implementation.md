# Wildfire Dashboard Implementation Plan

**For Claude: REQUIRED SUB-SKILL: Use superpowers:executing-plans**

## Goal
Build a zero-installation, browser-based interactive dashboard for exploring the Smokey wildfire dataset (55,367 fires, 1991-2015), designed for educators and the general public.

## Architecture
Single self-contained HTML file with embedded JavaScript and CSS. Uses Plotly.js for visualizations, PapaParse for CSV parsing, and Tailwind CSS for styling (all via CDN). Data loads from CSV file in same directory. All interactions happen client-side in the browser with no server required.

## Tech Stack
- **Plotly.js 2.27.0** - Interactive charts
- **PapaParse 5.4.1** - CSV parsing
- **Tailwind CSS 3.3.0** - Styling framework
- **Vanilla JavaScript ES6** - No frameworks
- **HTML5** - Single file structure

---

## Task 1: Create Basic HTML Structure and Data Loading

**Files:**
- **Create**: `index.html` (complete dashboard file)

### Step 1: Create HTML skeleton with CDN dependencies

Create the foundational HTML structure with all required libraries:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SMOKEY Wildfire Dashboard (1991-2015)</title>

    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>

    <!-- Plotly.js -->
    <script src="https://cdn.plot.ly/plotly-2.27.0.min.js"></script>

    <!-- PapaParse -->
    <script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>

    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
            background: #f5f5f5;
        }

        .loading-spinner {
            border: 4px solid #f3f3f3;
            border-top: 4px solid #ff6b35;
            border-radius: 50%;
            width: 50px;
            height: 50px;
            animation: spin 1s linear infinite;
        }

        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }

        .chart-container {
            background: white;
            border-radius: 8px;
            padding: 20px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            margin-bottom: 20px;
        }
    </style>
</head>
<body class="p-4">
    <!-- Loading Screen -->
    <div id="loading" class="fixed inset-0 bg-white flex flex-col items-center justify-center z-50">
        <div class="loading-spinner mb-4"></div>
        <h2 class="text-2xl font-bold text-gray-700" id="loading-text">Loading wildfire data...</h2>
        <p class="text-gray-500 mt-2" id="loading-progress">Initializing...</p>
    </div>

    <!-- Main Dashboard (hidden until loaded) -->
    <div id="dashboard" class="hidden">
        <header class="text-center mb-8">
            <h1 class="text-4xl font-bold text-gray-800 mb-2">🔥 SMOKEY Wildfire Dashboard</h1>
            <p class="text-gray-600">Exploring U.S. Wildfire Patterns (1991-2015)</p>
        </header>

        <!-- Filter bar will go here -->
        <div id="filters" class="mb-6"></div>

        <!-- Visualizations will go here -->
        <div id="visualizations"></div>
    </div>

    <script>
        // Global data storage
        let rawData = [];
        let filteredData = [];

        // Initialize dashboard
        async function initDashboard() {
            try {
                updateLoadingText('Loading CSV file...', 'Fetching FW_Veg_Rem_Combined.csv');
                await loadData();

                updateLoadingText('Processing data...', 'Cleaning 55,367 records');
                cleanData();

                updateLoadingText('Preparing visualizations...', 'Almost ready!');

                // Hide loading, show dashboard
                setTimeout(() => {
                    document.getElementById('loading').classList.add('hidden');
                    document.getElementById('dashboard').classList.remove('hidden');
                }, 500);

            } catch (error) {
                showError(error);
            }
        }

        function updateLoadingText(main, progress) {
            document.getElementById('loading-text').textContent = main;
            document.getElementById('loading-progress').textContent = progress;
        }

        function showError(error) {
            document.getElementById('loading').innerHTML = `
                <div class="text-center p-8">
                    <h2 class="text-2xl font-bold text-red-600 mb-4">⚠️ Error Loading Data</h2>
                    <p class="text-gray-700 mb-4">${error.message}</p>
                    <p class="text-gray-600">Please ensure <code class="bg-gray-100 px-2 py-1 rounded">FW_Veg_Rem_Combined.csv</code> is in the same folder as this HTML file.</p>
                    <button onclick="location.reload()" class="mt-4 bg-blue-500 text-white px-6 py-2 rounded hover:bg-blue-600">
                        Retry
                    </button>
                </div>
            `;
        }

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

        function cleanData() {
            // Filter out corrupted records and clean data
            filteredData = rawData.filter(row => {
                // Remove corrupted fire_size_class entries
                const validClasses = ['A', 'B', 'C', 'D', 'E', 'F', 'G'];
                if (!validClasses.includes(row.fire_size_class)) {
                    return false;
                }

                // Ensure basic required fields exist
                if (!row.state || !row.disc_pre_year) {
                    return false;
                }

                return true;
            });

            // Add derived fields
            filteredData.forEach(row => {
                // Map state to region
                row.region = getRegion(row.state);

                // Map month to season
                row.season = getSeason(row.discovery_month);

                // Classify as human vs natural
                row.is_human_caused = isHumanCaused(row.stat_cause_descr);

                // Handle missing weather data
                ['Temp_pre_30', 'Temp_pre_15', 'Temp_pre_7', 'Temp_cont',
                 'Wind_pre_30', 'Wind_pre_15', 'Wind_pre_7', 'Wind_cont',
                 'Hum_pre_30', 'Hum_pre_15', 'Hum_pre_7', 'Hum_cont',
                 'Prec_pre_30', 'Prec_pre_15', 'Prec_pre_7', 'Prec_cont'].forEach(field => {
                    if (row[field] === -1.0) {
                        row[field] = null;
                    }
                });
            });

            console.log(`Loaded ${filteredData.length} valid records`);
        }

        function getRegion(state) {
            const regions = {
                'West': ['WA', 'OR', 'CA', 'NV', 'ID', 'MT', 'WY', 'UT', 'CO', 'AZ', 'NM', 'AK', 'HI'],
                'Midwest': ['ND', 'SD', 'NE', 'KS', 'MN', 'IA', 'MO', 'WI', 'IL', 'MI', 'IN', 'OH'],
                'South': ['TX', 'OK', 'AR', 'LA', 'MS', 'AL', 'TN', 'KY', 'WV', 'VA', 'NC', 'SC', 'GA', 'FL', 'MD', 'DE', 'DC'],
                'Northeast': ['PA', 'NY', 'NJ', 'CT', 'RI', 'MA', 'VT', 'NH', 'ME']
            };

            for (let [region, states] of Object.entries(regions)) {
                if (states.includes(state)) return region;
            }
            return 'Other';
        }

        function getSeason(month) {
            const seasons = {
                'Winter': ['Dec', 'Jan', 'Feb'],
                'Spring': ['Mar', 'Apr', 'May'],
                'Summer': ['Jun', 'Jul', 'Aug'],
                'Fall': ['Sep', 'Oct', 'Nov']
            };

            for (let [season, months] of Object.entries(seasons)) {
                if (months.includes(month)) return season;
            }
            return 'Unknown';
        }

        function isHumanCaused(cause) {
            const humanCauses = ['Arson', 'Debris Burning', 'Equipment Use', 'Campfire',
                                'Children', 'Smoking', 'Fireworks', 'Railroad', 'Powerline', 'Structure'];
            return humanCauses.includes(cause);
        }

        // Start the dashboard
        initDashboard();
    </script>
</body>
</html>
```

**Verification Step 1:** Open `index.html` in browser
- Expected: See loading spinner with "Loading wildfire data..." text
- Expected: After 2-3 seconds, loading should complete or show error if CSV not found
- Expected: Console should log "Loaded [number] valid records"

**Commit Step 1:**
```bash
git add index.html
git commit -m "Add basic HTML structure with data loading

- Set up single-file HTML with CDN dependencies (Plotly, PapaParse, Tailwind)
- Implement CSV loading with PapaParse
- Add data cleaning pipeline (filter corrupted records, add derived fields)
- Create loading screen with error handling
- Map states to regions, months to seasons, classify human vs natural causes"
```

---

## Task 2: Build Filter Bar - Geography Selectors

**Files:**
- **Modify**: `index.html` (add filter bar HTML and logic)

### Step 2: Add geography filter controls

Insert after the `<div id="filters" class="mb-6"></div>` line, replace that div with:

```html
<!-- Filter Bar -->
<div id="filters" class="bg-white rounded-lg shadow-md p-6 mb-6 sticky top-4 z-40">
    <div class="grid grid-cols-1 md:grid-cols-5 gap-4">
        <!-- Geography Filters -->
        <div>
            <label class="block text-sm font-semibold text-gray-700 mb-2">📍 State</label>
            <select id="filter-state" class="w-full border border-gray-300 rounded-md px-3 py-2 focus:ring-2 focus:ring-blue-500">
                <option value="all">All States</option>
            </select>
        </div>

        <div>
            <label class="block text-sm font-semibold text-gray-700 mb-2">🗺️ Region</label>
            <select id="filter-region" class="w-full border border-gray-300 rounded-md px-3 py-2 focus:ring-2 focus:ring-blue-500">
                <option value="all">All Regions</option>
                <option value="West">West</option>
                <option value="Midwest">Midwest</option>
                <option value="South">South</option>
                <option value="Northeast">Northeast</option>
            </select>
        </div>

        <!-- Year Range (placeholder for now) -->
        <div>
            <label class="block text-sm font-semibold text-gray-700 mb-2">📅 Year Range</label>
            <div class="text-sm text-gray-500 py-2">1991 - 2015</div>
        </div>

        <!-- Fire Cause (placeholder) -->
        <div>
            <label class="block text-sm font-semibold text-gray-700 mb-2">🔥 Cause</label>
            <select id="filter-cause" class="w-full border border-gray-300 rounded-md px-3 py-2 focus:ring-2 focus:ring-blue-500">
                <option value="all">All Causes</option>
            </select>
        </div>

        <!-- Fire Size Class (placeholder) -->
        <div>
            <label class="block text-sm font-semibold text-gray-700 mb-2">📏 Size</label>
            <select id="filter-size" class="w-full border border-gray-300 rounded-md px-3 py-2 focus:ring-2 focus:ring-blue-500">
                <option value="all">All Sizes</option>
            </select>
        </div>
    </div>

    <!-- Filter Summary -->
    <div class="mt-4 pt-4 border-t border-gray-200 flex justify-between items-center">
        <div id="filter-summary" class="text-sm text-gray-600">
            Showing: <span id="record-count" class="font-bold text-blue-600">0</span> fires
        </div>
        <button id="reset-filters" class="bg-gray-200 hover:bg-gray-300 text-gray-700 px-4 py-2 rounded-md text-sm font-semibold">
            Reset All Filters
        </button>
    </div>
</div>
```

Now add the JavaScript logic before `initDashboard()`:

```javascript
// Filter state
let currentFilters = {
    state: 'all',
    region: 'all',
    yearMin: 1991,
    yearMax: 2015,
    cause: 'all',
    sizeClass: 'all'
};

function initializeFilters() {
    // Populate state dropdown
    const states = [...new Set(filteredData.map(r => r.state))].sort();
    const stateSelect = document.getElementById('filter-state');
    states.forEach(state => {
        const option = document.createElement('option');
        option.value = state;
        option.textContent = state;
        stateSelect.appendChild(option);
    });

    // Populate cause dropdown
    const causes = [...new Set(filteredData.map(r => r.stat_cause_descr))].filter(c => c).sort();
    const causeSelect = document.getElementById('filter-cause');
    causeSelect.innerHTML = '<option value="all">All Causes</option>';
    causeSelect.innerHTML += '<option value="human">Human-Caused</option>';
    causeSelect.innerHTML += '<option value="natural">Natural (Lightning)</option>';
    causeSelect.innerHTML += '<option disabled>──────────</option>';
    causes.forEach(cause => {
        const option = document.createElement('option');
        option.value = cause;
        option.textContent = cause;
        causeSelect.appendChild(option);
    });

    // Populate size class dropdown
    const sizeSelect = document.getElementById('filter-size');
    ['A', 'B', 'C', 'D', 'E', 'F', 'G'].forEach(size => {
        const option = document.createElement('option');
        option.value = size;
        option.textContent = `Class ${size}`;
        sizeSelect.appendChild(option);
    });

    // Add event listeners
    document.getElementById('filter-state').addEventListener('change', handleFilterChange);
    document.getElementById('filter-region').addEventListener('change', handleFilterChange);
    document.getElementById('filter-cause').addEventListener('change', handleFilterChange);
    document.getElementById('filter-size').addEventListener('change', handleFilterChange);
    document.getElementById('reset-filters').addEventListener('click', resetFilters);

    // Initial update
    updateFilteredData();
}

function handleFilterChange(event) {
    const filterId = event.target.id.replace('filter-', '');

    if (filterId === 'state') {
        currentFilters.state = event.target.value;
        // Reset region if state selected
        if (currentFilters.state !== 'all') {
            currentFilters.region = 'all';
            document.getElementById('filter-region').value = 'all';
        }
    } else if (filterId === 'region') {
        currentFilters.region = event.target.value;
        // Reset state if region selected
        if (currentFilters.region !== 'all') {
            currentFilters.state = 'all';
            document.getElementById('filter-state').value = 'all';
        }
    } else if (filterId === 'cause') {
        currentFilters.cause = event.target.value;
    } else if (filterId === 'size') {
        currentFilters.sizeClass = event.target.value;
    }

    updateFilteredData();
}

function updateFilteredData() {
    let data = [...filteredData];

    // Apply state filter
    if (currentFilters.state !== 'all') {
        data = data.filter(r => r.state === currentFilters.state);
    }

    // Apply region filter
    if (currentFilters.region !== 'all') {
        data = data.filter(r => r.region === currentFilters.region);
    }

    // Apply cause filter
    if (currentFilters.cause === 'human') {
        data = data.filter(r => r.is_human_caused);
    } else if (currentFilters.cause === 'natural') {
        data = data.filter(r => r.stat_cause_descr === 'Lightning');
    } else if (currentFilters.cause !== 'all') {
        data = data.filter(r => r.stat_cause_descr === currentFilters.cause);
    }

    // Apply size class filter
    if (currentFilters.sizeClass !== 'all') {
        data = data.filter(r => r.fire_size_class === currentFilters.sizeClass);
    }

    // Apply year range filter
    data = data.filter(r => r.disc_pre_year >= currentFilters.yearMin && r.disc_pre_year <= currentFilters.yearMax);

    // Update display
    document.getElementById('record-count').textContent = data.toLocaleString();

    // TODO: Update all charts
    console.log(`Filtered to ${data.length} records`);
}

function resetFilters() {
    currentFilters = {
        state: 'all',
        region: 'all',
        yearMin: 1991,
        yearMax: 2015,
        cause: 'all',
        sizeClass: 'all'
    };

    document.getElementById('filter-state').value = 'all';
    document.getElementById('filter-region').value = 'all';
    document.getElementById('filter-cause').value = 'all';
    document.getElementById('filter-size').value = 'all';

    updateFilteredData();
}
```

Update the `initDashboard()` function to call `initializeFilters()` after cleaning data:

```javascript
updateLoadingText('Preparing visualizations...', 'Almost ready!');
initializeFilters();  // Add this line

// Hide loading, show dashboard
setTimeout(() => {
```

**Verification Step 2:** Open `index.html` in browser
- Expected: See filter bar with 5 dropdowns
- Expected: State dropdown populated with all states alphabetically
- Expected: Cause dropdown has "All Causes", "Human-Caused", "Natural (Lightning)", plus individual causes
- Expected: Selecting different filters updates the "Showing: X fires" count
- Expected: "Reset All Filters" button resets all dropdowns to default
- Expected: Console logs "Filtered to X records" on each filter change

**Commit Step 2:**
```bash
git add index.html
git commit -m "Add filter bar with geography, cause, and size controls

- Create sticky filter bar with 5 filter controls
- Populate state dropdown from data (alphabetical)
- Populate cause dropdown with human/natural grouping
- Implement filter state management and cross-filtering logic
- Add reset filters functionality
- Display live count of filtered records
- State and region filters are mutually exclusive"
```

---

## Task 3: Build Temperature Before Fires Chart (Priority 1A)

**Files:**
- **Modify**: `index.html` (add first visualization)

### Step 3: Create temperature line chart

Add HTML container after `<div id="visualizations"></div>`, replace with:

```html
<!-- Visualizations -->
<div id="visualizations">
    <!-- Row 1: Weather & Fire Risk -->
    <div class="mb-8">
        <h2 class="text-2xl font-bold text-gray-800 mb-4">🌡️ Weather & Fire Risk</h2>
        <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
            <!-- Temperature Chart -->
            <div class="chart-container">
                <div class="flex justify-between items-center mb-3">
                    <h3 class="text-lg font-semibold text-gray-700">Temperature Before Fires</h3>
                    <button onclick="downloadChart('temp-chart')" class="text-sm text-blue-600 hover:text-blue-800">
                        📷 Save
                    </button>
                </div>
                <div id="temp-chart" style="height: 350px;"></div>
                <div class="mt-2 text-xs text-gray-500">
                    ⚠️ Based on fires with complete weather data
                </div>
            </div>

            <!-- Humidity/Precipitation Chart (placeholder) -->
            <div class="chart-container">
                <h3 class="text-lg font-semibold text-gray-700 mb-3">Humidity & Precipitation</h3>
                <div id="humidity-chart" style="height: 350px;" class="flex items-center justify-center text-gray-400">
                    Chart placeholder
                </div>
            </div>
        </div>
    </div>
</div>
```

Add JavaScript functions for rendering the temperature chart:

```javascript
function renderTemperatureChart(data) {
    // Filter data with valid temperature readings
    const validData = data.filter(r =>
        r.Temp_pre_30 !== null &&
        r.Temp_pre_15 !== null &&
        r.Temp_pre_7 !== null &&
        r.Temp_cont !== null
    );

    // Group by fire size class
    const sizeClasses = ['A', 'B', 'C', 'D', 'E', 'F', 'G'];
    const traces = [];

    sizeClasses.forEach(sizeClass => {
        const classData = validData.filter(r => r.fire_size_class === sizeClass);

        if (classData.length === 0) return;

        // Calculate averages for each time window
        const avgTemp30 = classData.reduce((sum, r) => sum + r.Temp_pre_30, 0) / classData.length;
        const avgTemp15 = classData.reduce((sum, r) => sum + r.Temp_pre_15, 0) / classData.length;
        const avgTemp7 = classData.reduce((sum, r) => sum + r.Temp_pre_7, 0) / classData.length;
        const avgTempCont = classData.reduce((sum, r) => sum + r.Temp_cont, 0) / classData.length;

        traces.push({
            x: ['-30 days', '-15 days', '-7 days', 'Containment'],
            y: [avgTemp30, avgTemp15, avgTemp7, avgTempCont],
            type: 'scatter',
            mode: 'lines+markers',
            name: `Class ${sizeClass} (${classData.length})`,
            line: { width: 2 },
            marker: { size: 8 }
        });
    });

    const layout = {
        xaxis: {
            title: 'Time Window',
            showgrid: true
        },
        yaxis: {
            title: 'Average Temperature (°C)',
            showgrid: true
        },
        hovermode: 'closest',
        showlegend: true,
        legend: {
            x: 1,
            xanchor: 'right',
            y: 1
        },
        margin: { l: 50, r: 50, t: 20, b: 50 }
    };

    const config = {
        responsive: true,
        displayModeBar: true,
        displaylogo: false,
        modeBarButtonsToRemove: ['pan2d', 'lasso2d', 'select2d']
    };

    Plotly.newPlot('temp-chart', traces, layout, config);
}

function downloadChart(chartId) {
    Plotly.downloadImage(chartId, {
        format: 'png',
        width: 1200,
        height: 600,
        filename: `smokey-${chartId}`
    });
}

function renderAllCharts() {
    const data = getFilteredData();
    renderTemperatureChart(data);
    // More charts will be added here
}

function getFilteredData() {
    let data = [...filteredData];

    // Apply all current filters
    if (currentFilters.state !== 'all') {
        data = data.filter(r => r.state === currentFilters.state);
    }
    if (currentFilters.region !== 'all') {
        data = data.filter(r => r.region === currentFilters.region);
    }
    if (currentFilters.cause === 'human') {
        data = data.filter(r => r.is_human_caused);
    } else if (currentFilters.cause === 'natural') {
        data = data.filter(r => r.stat_cause_descr === 'Lightning');
    } else if (currentFilters.cause !== 'all') {
        data = data.filter(r => r.stat_cause_descr === currentFilters.cause);
    }
    if (currentFilters.sizeClass !== 'all') {
        data = data.filter(r => r.fire_size_class === currentFilters.sizeClass);
    }
    data = data.filter(r => r.disc_pre_year >= currentFilters.yearMin && r.disc_pre_year <= currentFilters.yearMax);

    return data;
}
```

Update `updateFilteredData()` to call `renderAllCharts()`:

```javascript
function updateFilteredData() {
    let data = [...filteredData];

    // ... existing filter logic ...

    // Update display
    document.getElementById('record-count').textContent = data.toLocaleString();

    // Render all charts with filtered data
    renderAllCharts();  // Add this line
}
```

Update `initDashboard()` to render charts initially:

```javascript
updateLoadingText('Preparing visualizations...', 'Almost ready!');
initializeFilters();

// Initial chart render
setTimeout(() => {
    renderAllCharts();  // Add this line
}, 100);

// Hide loading, show dashboard
setTimeout(() => {
```

**Verification Step 3:** Open `index.html` in browser
- Expected: See temperature chart with multiple colored lines (one per fire size class)
- Expected: X-axis shows: "-30 days", "-15 days", "-7 days", "Containment"
- Expected: Y-axis shows temperature in °C
- Expected: Legend shows "Class A (count)", "Class B (count)", etc.
- Expected: Hovering over points shows exact temperature values
- Expected: Clicking "📷 Save" button downloads chart as PNG
- Expected: Changing filters updates the chart immediately

**Commit Step 3:**
```bash
git add index.html
git commit -m "Add temperature before fires visualization

- Create multi-line chart showing avg temp at 4 time windows
- Group by fire size class (A-G) with separate lines
- Filter out records with missing weather data (-1.0 values)
- Calculate averages per size class
- Add download chart functionality
- Integrate chart updates with filter system
- Show data quality warning badge"
```

---

## Task 4: Build Humidity & Precipitation Chart (Priority 1B)

**Files:**
- **Modify**: `index.html` (add humidity/precipitation chart)

### Step 4: Create dual-axis humidity and precipitation chart

Replace the humidity chart placeholder div content with actual chart container:

```html
<!-- Humidity/Precipitation Chart -->
<div class="chart-container">
    <div class="flex justify-between items-center mb-3">
        <h3 class="text-lg font-semibold text-gray-700">Humidity & Precipitation</h3>
        <button onclick="downloadChart('humidity-chart')" class="text-sm text-blue-600 hover:text-blue-800">
            📷 Save
        </button>
    </div>
    <div id="humidity-chart" style="height: 350px;"></div>
    <div class="mt-2 text-xs text-gray-500">
        Blue bars: Humidity (%), Green line: Precipitation (mm)
    </div>
</div>
```

Add the rendering function before `renderAllCharts()`:

```javascript
function renderHumidityPrecipChart(data) {
    // Filter data with valid humidity and precipitation readings
    const validData = data.filter(r =>
        r.Hum_pre_30 !== null && r.Hum_pre_15 !== null && r.Hum_pre_7 !== null && r.Hum_cont !== null &&
        r.Prec_pre_30 !== null && r.Prec_pre_15 !== null && r.Prec_pre_7 !== null && r.Prec_cont !== null
    );

    if (validData.length === 0) {
        document.getElementById('humidity-chart').innerHTML =
            '<div class="flex items-center justify-center h-full text-gray-400">No data with complete weather records</div>';
        return;
    }

    // Calculate averages
    const avgHum30 = validData.reduce((sum, r) => sum + r.Hum_pre_30, 0) / validData.length;
    const avgHum15 = validData.reduce((sum, r) => sum + r.Hum_pre_15, 0) / validData.length;
    const avgHum7 = validData.reduce((sum, r) => sum + r.Hum_pre_7, 0) / validData.length;
    const avgHumCont = validData.reduce((sum, r) => sum + r.Hum_cont, 0) / validData.length;

    const avgPrec30 = validData.reduce((sum, r) => sum + r.Prec_pre_30, 0) / validData.length;
    const avgPrec15 = validData.reduce((sum, r) => sum + r.Prec_pre_15, 0) / validData.length;
    const avgPrec7 = validData.reduce((sum, r) => sum + r.Prec_pre_7, 0) / validData.length;
    const avgPrecCont = validData.reduce((sum, r) => sum + r.Prec_cont, 0) / validData.length;

    const timeWindows = ['-30 days', '-15 days', '-7 days', 'Containment'];

    // Humidity bars
    const humidityTrace = {
        x: timeWindows,
        y: [avgHum30, avgHum15, avgHum7, avgHumCont],
        type: 'bar',
        name: 'Humidity (%)',
        marker: {
            color: 'rgba(54, 162, 235, 0.7)',
            line: { color: 'rgba(54, 162, 235, 1)', width: 2 }
        },
        yaxis: 'y'
    };

    // Precipitation line
    const precipTrace = {
        x: timeWindows,
        y: [avgPrec30, avgPrec15, avgPrec7, avgPrecCont],
        type: 'scatter',
        mode: 'lines+markers',
        name: 'Precipitation (mm)',
        line: { color: 'rgba(75, 192, 192, 1)', width: 3 },
        marker: { size: 10, color: 'rgba(75, 192, 192, 1)' },
        yaxis: 'y2'
    };

    const layout = {
        xaxis: {
            title: 'Time Window',
            showgrid: true
        },
        yaxis: {
            title: 'Humidity (%)',
            titlefont: { color: 'rgba(54, 162, 235, 1)' },
            tickfont: { color: 'rgba(54, 162, 235, 1)' },
            showgrid: true,
            range: [0, 100]
        },
        yaxis2: {
            title: 'Precipitation (mm)',
            titlefont: { color: 'rgba(75, 192, 192, 1)' },
            tickfont: { color: 'rgba(75, 192, 192, 1)' },
            overlaying: 'y',
            side: 'right',
            showgrid: false
        },
        hovermode: 'x unified',
        showlegend: true,
        legend: { x: 0, y: 1.1, orientation: 'h' },
        margin: { l: 50, r: 50, t: 20, b: 50 }
    };

    const config = {
        responsive: true,
        displayModeBar: true,
        displaylogo: false,
        modeBarButtonsToRemove: ['pan2d', 'lasso2d', 'select2d']
    };

    Plotly.newPlot('humidity-chart', [humidityTrace, precipTrace], layout, config);
}
```

Update `renderAllCharts()` to include the new chart:

```javascript
function renderAllCharts() {
    const data = getFilteredData();
    renderTemperatureChart(data);
    renderHumidityPrecipChart(data);  // Add this line
}
```

**Verification Step 4:** Open `index.html` in browser
- Expected: See humidity chart with blue bars (left y-axis) and green line (right y-axis)
- Expected: Left y-axis labeled "Humidity (%)", range 0-100
- Expected: Right y-axis labeled "Precipitation (mm)"
- Expected: Both traces share same x-axis time windows
- Expected: Hovering shows unified tooltip with both values
- Expected: Legend shows "Humidity (%)" and "Precipitation (mm)"
- Expected: Changing filters updates both weather charts

**Commit Step 4:**
```bash
git add index.html
git commit -m "Add humidity and precipitation dual-axis chart

- Create combination chart with bar (humidity) and line (precipitation)
- Use dual y-axis for different scales
- Filter records with complete weather data
- Color-code: blue for humidity, green for precipitation
- Add unified hover tooltip
- Integrate with filter system"
```

---

## Task 5: Build Seasonal Heatmap (Priority 2A)

**Files:**
- **Modify**: `index.html` (add temporal visualizations row)

### Step 5: Create month-year heatmap

Add HTML after the weather row:

```html
    <!-- Row 2: When Fires Strike -->
    <div class="mb-8">
        <h2 class="text-2xl font-bold text-gray-800 mb-4">📅 When Fires Strike</h2>
        <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
            <!-- Seasonal Heatmap -->
            <div class="chart-container">
                <div class="flex justify-between items-center mb-3">
                    <h3 class="text-lg font-semibold text-gray-700">Seasonal Patterns</h3>
                    <button onclick="downloadChart('heatmap-chart')" class="text-sm text-blue-600 hover:text-blue-800">
                        📷 Save
                    </button>
                </div>
                <div id="heatmap-chart" style="height: 500px;"></div>
                <div class="mt-2 text-xs text-gray-500">
                    Darker red = more fires. Click to explore specific periods.
                </div>
            </div>

            <!-- Year-over-Year Trends (placeholder) -->
            <div class="chart-container">
                <h3 class="text-lg font-semibold text-gray-700 mb-3">Year-over-Year Trends</h3>
                <div id="trends-chart" style="height: 500px;" class="flex items-center justify-center text-gray-400">
                    Chart placeholder
                </div>
            </div>
        </div>
    </div>
</div>
```

Add the heatmap rendering function:

```javascript
function renderSeasonalHeatmap(data) {
    // Create matrix: years (y) x months (x)
    const months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
    const years = [];
    for (let y = 1991; y <= 2015; y++) {
        years.push(y);
    }

    // Count fires for each month-year combination
    const matrix = [];
    const counts = {};

    data.forEach(row => {
        const year = row.disc_pre_year;
        const month = row.discovery_month;
        const key = `${year}-${month}`;
        counts[key] = (counts[key] || 0) + 1;
    });

    // Build matrix (years as rows, months as columns)
    years.forEach(year => {
        const row = [];
        months.forEach(month => {
            const key = `${year}-${month}`;
            row.push(counts[key] || 0);
        });
        matrix.push(row);
    });

    const trace = {
        z: matrix,
        x: months,
        y: years,
        type: 'heatmap',
        colorscale: [
            [0, 'rgb(255, 255, 255)'],
            [0.2, 'rgb(255, 237, 160)'],
            [0.4, 'rgb(254, 178, 76)'],
            [0.6, 'rgb(253, 141, 60)'],
            [0.8, 'rgb(240, 59, 32)'],
            [1, 'rgb(189, 0, 38)']
        ],
        colorbar: {
            title: 'Fire Count',
            thickness: 15
        },
        hovertemplate: '<b>%{y} %{x}</b><br>Fires: %{z}<extra></extra>'
    };

    const layout = {
        xaxis: {
            title: 'Month',
            side: 'bottom',
            tickmode: 'linear'
        },
        yaxis: {
            title: 'Year',
            autorange: 'reversed',
            tickmode: 'linear',
            dtick: 2
        },
        margin: { l: 60, r: 50, t: 20, b: 60 }
    };

    const config = {
        responsive: true,
        displayModeBar: true,
        displaylogo: false,
        modeBarButtonsToRemove: ['pan2d', 'lasso2d', 'select2d', 'zoom2d']
    };

    Plotly.newPlot('heatmap-chart', [trace], layout, config);
}
```

Update `renderAllCharts()`:

```javascript
function renderAllCharts() {
    const data = getFilteredData();
    renderTemperatureChart(data);
    renderHumidityPrecipChart(data);
    renderSeasonalHeatmap(data);  // Add this line
}
```

**Verification Step 5:** Open `index.html` in browser
- Expected: See heatmap with months (x-axis) and years (y-axis)
- Expected: Color gradient from white (0 fires) to dark red (many fires)
- Expected: Hover shows "YYYY Month" and fire count
- Expected: March-April columns should be visibly darker (more fires)
- Expected: Color bar on right shows scale
- Expected: Filtering by state/region updates heatmap to show regional patterns

**Commit Step 5:**
```bash
git add index.html
git commit -m "Add seasonal heatmap visualization

- Create month-year heatmap showing fire frequency
- Use white-to-red color scale for fire counts
- Display years 1991-2015 on y-axis, months on x-axis
- Add hover tooltips with exact counts
- Integrate with filter system
- Visually highlight peak fire seasons (March-April)"
```

---

## Task 6: Build Year-over-Year Trends Chart (Priority 2B)

**Files:**
- **Modify**: `index.html` (add trends chart)

### Step 6: Create time series showing yearly fire counts

Replace the trends chart placeholder:

```html
<!-- Year-over-Year Trends -->
<div class="chart-container">
    <div class="flex justify-between items-center mb-3">
        <h3 class="text-lg font-semibold text-gray-700">Year-over-Year Trends</h3>
        <button onclick="downloadChart('trends-chart')" class="text-sm text-blue-600 hover:text-blue-800">
            📷 Save
        </button>
    </div>
    <div id="trends-chart" style="height: 500px;"></div>
    <div class="mt-2 text-xs text-gray-500">
        Shows fire counts by year. Separate lines for top causes.
    </div>
</div>
```

Add the rendering function:

```javascript
function renderYearlyTrends(data) {
    // Count total fires per year
    const yearCounts = {};
    const causeCounts = {};

    data.forEach(row => {
        const year = row.disc_pre_year;
        yearCounts[year] = (yearCounts[year] || 0) + 1;

        const cause = row.stat_cause_descr || 'Unknown';
        if (!causeCounts[cause]) {
            causeCounts[cause] = {};
        }
        causeCounts[cause][year] = (causeCounts[cause][year] || 0) + 1;
    });

    // Get top 5 causes by total count
    const topCauses = Object.entries(causeCounts)
        .map(([cause, years]) => ({
            cause,
            total: Object.values(years).reduce((a, b) => a + b, 0)
        }))
        .sort((a, b) => b.total - a.total)
        .slice(0, 5)
        .map(c => c.cause);

    const years = [];
    for (let y = 1991; y <= 2015; y++) {
        years.push(y);
    }

    const traces = [];

    // Total fires trace
    traces.push({
        x: years,
        y: years.map(y => yearCounts[y] || 0),
        type: 'scatter',
        mode: 'lines+markers',
        name: 'Total Fires',
        line: { width: 3, color: 'rgb(0, 0, 0)' },
        marker: { size: 6 }
    });

    // Top causes traces
    const colors = [
        'rgb(255, 99, 71)',   // Red
        'rgb(255, 159, 64)',  // Orange
        'rgb(75, 192, 192)',  // Teal
        'rgb(54, 162, 235)',  // Blue
        'rgb(153, 102, 255)'  // Purple
    ];

    topCauses.forEach((cause, idx) => {
        traces.push({
            x: years,
            y: years.map(y => causeCounts[cause][y] || 0),
            type: 'scatter',
            mode: 'lines',
            name: cause,
            line: { width: 2, color: colors[idx] },
            visible: 'legendonly'  // Hidden by default, click to show
        });
    });

    const layout = {
        xaxis: {
            title: 'Year',
            showgrid: true,
            dtick: 2
        },
        yaxis: {
            title: 'Number of Fires',
            showgrid: true
        },
        hovermode: 'x unified',
        showlegend: true,
        legend: {
            x: 0,
            y: 1,
            bgcolor: 'rgba(255, 255, 255, 0.8)'
        },
        margin: { l: 60, r: 30, t: 20, b: 60 }
    };

    const config = {
        responsive: true,
        displayModeBar: true,
        displaylogo: false,
        modeBarButtonsToRemove: ['pan2d', 'lasso2d', 'select2d']
    };

    Plotly.newPlot('trends-chart', traces, layout, config);
}
```

Update `renderAllCharts()`:

```javascript
function renderAllCharts() {
    const data = getFilteredData();
    renderTemperatureChart(data);
    renderHumidityPrecipChart(data);
    renderSeasonalHeatmap(data);
    renderYearlyTrends(data);  // Add this line
}
```

**Verification Step 6:** Open `index.html` in browser
- Expected: See line chart with years (1991-2015) on x-axis
- Expected: Thick black line showing total fires per year
- Expected: Legend shows "Total Fires" and top 5 causes
- Expected: Cause lines are hidden by default (click legend to show)
- Expected: Hover shows all values for that year
- Expected: Can zoom and pan the chart
- Expected: Filtering updates the trends

**Commit Step 6:**
```bash
git add index.html
git commit -m "Add year-over-year trends visualization

- Create multi-line time series chart (1991-2015)
- Show total fires as bold black line
- Add top 5 causes as separate lines (hidden by default)
- Calculate top causes dynamically from filtered data
- Use unified hover mode for easy comparison
- Integrate with filter system"
```

---

## Task 7: Build Fire Causes Donut Chart (Priority 3A)

**Files:**
- **Modify**: `index.html` (add causation row)

### Step 7: Create cause breakdown donut chart

Add HTML after the temporal row:

```html
    <!-- Row 3: Fire Causes & Characteristics -->
    <div class="mb-8">
        <h2 class="text-2xl font-bold text-gray-800 mb-4">🔥 Fire Causes & Characteristics</h2>
        <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
            <!-- Cause Breakdown -->
            <div class="chart-container">
                <div class="flex justify-between items-center mb-3">
                    <h3 class="text-lg font-semibold text-gray-700">What Starts Fires?</h3>
                    <button onclick="downloadChart('cause-chart')" class="text-sm text-blue-600 hover:text-blue-800">
                        📷 Save
                    </button>
                </div>
                <div id="cause-chart" style="height: 400px;"></div>
                <div class="mt-2 text-sm text-gray-600 bg-yellow-50 p-3 rounded border-l-4 border-yellow-400">
                    <strong>Prevention Tip:</strong> <span id="prevention-tip">Most fires are human-caused and preventable!</span>
                </div>
            </div>

            <!-- Fire Size Distribution (placeholder) -->
            <div class="chart-container">
                <h3 class="text-lg font-semibold text-gray-700 mb-3">Fire Size Distribution</h3>
                <div id="size-chart" style="height: 400px;" class="flex items-center justify-center text-gray-400">
                    Chart placeholder
                </div>
            </div>
        </div>
    </div>
</div>
```

Add the donut chart rendering function:

```javascript
function renderCauseDonut(data) {
    // Count fires by cause
    const causeCounts = {};
    data.forEach(row => {
        const cause = row.stat_cause_descr || 'Unknown';
        causeCounts[cause] = (causeCounts[cause] || 0) + 1;
    });

    // Sort by count
    const sortedCauses = Object.entries(causeCounts)
        .sort((a, b) => b[1] - a[1]);

    const labels = sortedCauses.map(c => c[0]);
    const values = sortedCauses.map(c => c[1]);

    // Color by human vs natural
    const colors = labels.map(cause => {
        const humanCauses = ['Arson', 'Debris Burning', 'Equipment Use', 'Campfire',
                            'Children', 'Smoking', 'Fireworks', 'Railroad', 'Powerline', 'Structure'];
        if (humanCauses.includes(cause)) {
            return 'rgba(255, 99, 71, 0.8)';  // Red for human
        } else if (cause === 'Lightning') {
            return 'rgba(54, 162, 235, 0.8)';  // Blue for natural
        } else {
            return 'rgba(150, 150, 150, 0.8)';  // Gray for unknown
        }
    });

    const trace = {
        labels: labels,
        values: values,
        type: 'pie',
        hole: 0.4,  // Donut hole
        marker: {
            colors: colors,
            line: { color: 'white', width: 2 }
        },
        textposition: 'outside',
        textinfo: 'label+percent',
        hovertemplate: '<b>%{label}</b><br>Count: %{value}<br>Percentage: %{percent}<extra></extra>'
    };

    // Calculate human vs natural percentage
    const humanCount = data.filter(r => r.is_human_caused).length;
    const humanPercent = ((humanCount / data.length) * 100).toFixed(1);

    const layout = {
        annotations: [{
            text: `${humanPercent}%<br>Human<br>Caused`,
            x: 0.5,
            y: 0.5,
            font: { size: 18, color: 'rgb(255, 99, 71)' },
            showarrow: false
        }],
        showlegend: true,
        legend: {
            orientation: 'v',
            x: 1.1,
            y: 0.5
        },
        margin: { l: 20, r: 150, t: 20, b: 20 }
    };

    const config = {
        responsive: true,
        displayModeBar: true,
        displaylogo: false,
        modeBarButtonsToRemove: ['pan2d', 'lasso2d', 'select2d', 'zoom2d']
    };

    Plotly.newPlot('cause-chart', [trace], layout, config);

    // Update prevention tip
    updatePreventionTip(sortedCauses[0][0]);
}

function updatePreventionTip(topCause) {
    const tips = {
        'Debris Burning': 'Check local burn bans before burning debris. Keep fires small and attended.',
        'Arson': 'Report suspicious activity. Arson is a crime with serious consequences.',
        'Lightning': 'Natural fires are part of ecosystem cycles, but stay alert during dry storms.',
        'Miscellaneous': 'Stay aware of fire conditions and follow local fire safety guidelines.',
        'Equipment Use': 'Maintain equipment properly. Avoid using spark-producing tools in dry conditions.',
        'Campfire': 'Never leave campfires unattended. Drown fires completely before leaving.',
        'Children': 'Teach children about fire safety. Keep matches and lighters out of reach.',
        'Smoking': 'Dispose of cigarettes properly. Never throw lit cigarettes from vehicles.',
        'Railroad': 'Report track sparking and vegetation overgrowth near railways.',
        'Fireworks': 'Follow local fireworks laws. Use in clear areas away from dry vegetation.'
    };

    const tip = tips[topCause] || 'Most fires are human-caused and preventable!';
    document.getElementById('prevention-tip').textContent = tip;
}
```

Update `renderAllCharts()`:

```javascript
function renderAllCharts() {
    const data = getFilteredData();
    renderTemperatureChart(data);
    renderHumidityPrecipChart(data);
    renderSeasonalHeatmap(data);
    renderYearlyTrends(data);
    renderCauseDonut(data);  // Add this line
}
```

**Verification Step 7:** Open `index.html` in browser
- Expected: See donut chart with center showing "X% Human Caused"
- Expected: Segments colored red (human), blue (lightning), gray (unknown)
- Expected: Labels outside showing cause name and percentage
- Expected: Hover shows exact count and percentage
- Expected: Prevention tip updates based on top cause in filtered data
- Expected: Filtering updates the chart and tip

**Commit Step 7:**
```bash
git add index.html
git commit -m "Add fire causes donut chart with prevention tips

- Create donut chart showing cause breakdown
- Color-code: red for human causes, blue for natural, gray for unknown
- Display percentage of human-caused fires in center
- Add dynamic prevention tip based on top cause
- Show label+percent on segments
- Integrate with filter system
- Include educational messaging"
```

---

## Task 8: Build Fire Size Distribution Histogram (Priority 3B)

**Files:**
- **Modify**: `index.html` (add size distribution chart)

### Step 8: Create histogram showing fire size distribution

Replace the size chart placeholder:

```html
<!-- Fire Size Distribution -->
<div class="chart-container">
    <div class="flex justify-between items-center mb-3">
        <h3 class="text-lg font-semibold text-gray-700">Fire Size Distribution</h3>
        <button onclick="downloadChart('size-chart')" class="text-sm text-blue-600 hover:text-blue-800">
            📷 Save
        </button>
    </div>
    <div id="size-chart" style="height: 400px;"></div>
    <div class="mt-2 text-xs text-gray-500">
        Most fires are small (Class A-B), but large fires (F-G) burn the most acres.
    </div>
</div>
```

Add the histogram rendering function:

```javascript
function renderSizeHistogram(data) {
    // Get fire sizes (in acres)
    const sizes = data.map(r => r.fire_size).filter(s => s && s > 0);

    if (sizes.length === 0) {
        document.getElementById('size-chart').innerHTML =
            '<div class="flex items-center justify-center h-full text-gray-400">No size data available</div>';
        return;
    }

    // Calculate statistics
    const mean = sizes.reduce((a, b) => a + b, 0) / sizes.length;
    const sorted = [...sizes].sort((a, b) => a - b);
    const median = sorted[Math.floor(sorted.length / 2)];
    const max = Math.max(...sizes);

    const trace = {
        x: sizes,
        type: 'histogram',
        nbinsx: 50,
        marker: {
            color: 'rgba(255, 159, 64, 0.7)',
            line: {
                color: 'rgba(255, 159, 64, 1)',
                width: 1
            }
        },
        hovertemplate: 'Size Range: %{x}<br>Count: %{y}<extra></extra>'
    };

    const layout = {
        xaxis: {
            title: 'Fire Size (acres)',
            type: 'log',  // Logarithmic scale for wide range
            showgrid: true
        },
        yaxis: {
            title: 'Number of Fires',
            showgrid: true
        },
        shapes: [
            // Mean line
            {
                type: 'line',
                x0: mean,
                x1: mean,
                y0: 0,
                y1: 1,
                yref: 'paper',
                line: {
                    color: 'red',
                    width: 2,
                    dash: 'dash'
                }
            },
            // Median line
            {
                type: 'line',
                x0: median,
                x1: median,
                y0: 0,
                y1: 1,
                yref: 'paper',
                line: {
                    color: 'blue',
                    width: 2,
                    dash: 'dot'
                }
            }
        ],
        annotations: [
            {
                x: mean,
                y: 0.9,
                yref: 'paper',
                text: `Mean: ${mean.toFixed(1)} acres`,
                showarrow: true,
                arrowhead: 2,
                ax: 40,
                ay: -40,
                bgcolor: 'rgba(255, 255, 255, 0.8)',
                bordercolor: 'red'
            },
            {
                x: median,
                y: 0.7,
                yref: 'paper',
                text: `Median: ${median.toFixed(1)} acres`,
                showarrow: true,
                arrowhead: 2,
                ax: -40,
                ay: -40,
                bgcolor: 'rgba(255, 255, 255, 0.8)',
                bordercolor: 'blue'
            }
        ],
        margin: { l: 60, r: 30, t: 20, b: 60 }
    };

    const config = {
        responsive: true,
        displayModeBar: true,
        displaylogo: false,
        modeBarButtonsToRemove: ['pan2d', 'lasso2d', 'select2d']
    };

    Plotly.newPlot('size-chart', [trace], layout, config);
}
```

Update `renderAllCharts()`:

```javascript
function renderAllCharts() {
    const data = getFilteredData();
    renderTemperatureChart(data);
    renderHumidityPrecipChart(data);
    renderSeasonalHeatmap(data);
    renderYearlyTrends(data);
    renderCauseDonut(data);
    renderSizeHistogram(data);  // Add this line
}
```

**Verification Step 8:** Open `index.html` in browser
- Expected: See histogram with logarithmic x-axis (fire sizes)
- Expected: Most bars on left side (small fires are common)
- Expected: Red dashed line showing mean, blue dotted line showing median
- Expected: Annotations pointing to mean and median values
- Expected: Hover shows size range and count
- Expected: Can see full range from 0.5 to 606K acres

**Commit Step 8:**
```bash
git add index.html
git commit -m "Add fire size distribution histogram

- Create histogram with 50 bins showing size distribution
- Use logarithmic x-axis for wide range (0.5 - 606K acres)
- Display mean and median as annotated lines
- Color-code bars in orange gradient
- Show most fires are small but range is extreme
- Integrate with filter system"
```

---

## Task 9: Build U.S. Geographic Map (Row 4)

**Files:**
- **Modify**: `index.html` (add map visualization)

### Step 9: Create interactive choropleth map

Add HTML after Row 3:

```html
    <!-- Row 4: Geographic Overview -->
    <div class="mb-8">
        <h2 class="text-2xl font-bold text-gray-800 mb-4">🗺️ Geographic Overview</h2>
        <div class="chart-container">
            <div class="flex justify-between items-center mb-3">
                <h3 class="text-lg font-semibold text-gray-700">U.S. Fire Density by State</h3>
                <button onclick="downloadChart('map-chart')" class="text-sm text-blue-600 hover:text-blue-800">
                    📷 Save
                </button>
            </div>
            <div id="map-chart" style="height: 500px;"></div>
            <div class="mt-2 text-xs text-gray-500">
                Click a state to filter entire dashboard to that region.
            </div>
        </div>
    </div>
</div>
```

Add the map rendering function:

```javascript
function renderUSMap(data) {
    // Count fires by state
    const stateCounts = {};
    const stateData = {};

    data.forEach(row => {
        const state = row.state;
        if (!state) return;

        if (!stateCounts[state]) {
            stateCounts[state] = 0;
            stateData[state] = {
                count: 0,
                totalSize: 0,
                causes: {}
            };
        }

        stateCounts[state]++;
        stateData[state].count++;
        stateData[state].totalSize += (row.fire_size || 0);

        const cause = row.stat_cause_descr || 'Unknown';
        stateData[state].causes[cause] = (stateData[state].causes[cause] || 0) + 1;
    });

    // Prepare data for choropleth
    const states = Object.keys(stateCounts);
    const counts = states.map(s => stateCounts[s]);

    // Get top cause per state
    const hoverText = states.map(state => {
        const info = stateData[state];
        const avgSize = info.totalSize / info.count;
        const topCause = Object.entries(info.causes)
            .sort((a, b) => b[1] - a[1])[0][0];

        return `<b>${state}</b><br>` +
               `Fires: ${info.count.toLocaleString()}<br>` +
               `Avg Size: ${avgSize.toFixed(1)} acres<br>` +
               `Top Cause: ${topCause}`;
    });

    const trace = {
        type: 'choropleth',
        locationmode: 'USA-states',
        locations: states,
        z: counts,
        text: hoverText,
        hovertemplate: '%{text}<extra></extra>',
        colorscale: [
            [0, 'rgb(255, 255, 255)'],
            [0.2, 'rgb(255, 237, 160)'],
            [0.4, 'rgb(254, 178, 76)'],
            [0.6, 'rgb(253, 141, 60)'],
            [0.8, 'rgb(240, 59, 32)'],
            [1, 'rgb(189, 0, 38)']
        ],
        colorbar: {
            title: 'Fire<br>Count',
            thickness: 15
        },
        marker: {
            line: {
                color: 'rgb(255, 255, 255)',
                width: 2
            }
        }
    };

    const layout = {
        geo: {
            scope: 'usa',
            projection: { type: 'albers usa' },
            showlakes: true,
            lakecolor: 'rgb(220, 240, 255)'
        },
        margin: { l: 0, r: 0, t: 0, b: 0 }
    };

    const config = {
        responsive: true,
        displayModeBar: true,
        displaylogo: false,
        modeBarButtonsToRemove: ['pan2d', 'lasso2d', 'select2d', 'zoom2d']
    };

    Plotly.newPlot('map-chart', [trace], layout, config);

    // Add click handler to filter by state
    document.getElementById('map-chart').on('plotly_click', function(eventData) {
        if (eventData.points.length > 0) {
            const clickedState = eventData.points[0].location;
            document.getElementById('filter-state').value = clickedState;
            currentFilters.state = clickedState;
            currentFilters.region = 'all';
            document.getElementById('filter-region').value = 'all';
            updateFilteredData();
        }
    });
}
```

Update `renderAllCharts()`:

```javascript
function renderAllCharts() {
    const data = getFilteredData();
    renderTemperatureChart(data);
    renderHumidityPrecipChart(data);
    renderSeasonalHeatmap(data);
    renderYearlyTrends(data);
    renderCauseDonut(data);
    renderSizeHistogram(data);
    renderUSMap(data);  // Add this line
}
```

**Verification Step 9:** Open `index.html` in browser
- Expected: See U.S. map with states colored by fire density
- Expected: Southern states (TX, GA, FL) appear darker red
- Expected: Hover shows state name, fire count, avg size, top cause
- Expected: Color scale from white (few fires) to dark red (many fires)
- Expected: Clicking a state filters entire dashboard to that state
- Expected: State filter dropdown updates when map is clicked

**Commit Step 9:**
```bash
git add index.html
git commit -m "Add interactive U.S. choropleth map

- Create state-level fire density map
- Color states by fire count (white to dark red scale)
- Calculate and display: total fires, avg size, top cause per state
- Add click handler to filter dashboard by state
- Sync map clicks with state dropdown filter
- Show geographic patterns (South/Southeast hotspots)
- Use Albers USA projection"
```

---

## Task 10: Add Educational Info Panels

**Files:**
- **Modify**: `index.html` (add collapsible info panels)

### Step 10: Create educational content panels

Add HTML before the visualizations section (after the filter bar):

```html
<!-- Educational Info Panels -->
<div class="mb-6 space-y-3">
    <!-- About This Data -->
    <div class="bg-white rounded-lg shadow-md overflow-hidden">
        <button onclick="togglePanel('about-panel')" class="w-full px-6 py-3 text-left font-semibold text-gray-700 hover:bg-gray-50 flex justify-between items-center">
            <span>ℹ️ About This Data</span>
            <span id="about-icon" class="text-gray-400">▼</span>
        </button>
        <div id="about-panel" class="hidden px-6 py-4 bg-gray-50 border-t border-gray-200">
            <p class="text-sm text-gray-700 mb-3">
                This dashboard visualizes <strong>55,367 wildfire incidents</strong> across the United States from <strong>1991-2015</strong>.
                Data combines fire incident reports from the U.S. Forest Service with weather data from NOAA weather stations
                and vegetation classifications from USGS land cover datasets.
            </p>
            <p class="text-sm text-gray-700 mb-3">
                <strong>Data Sources:</strong>
            </p>
            <ul class="text-sm text-gray-700 list-disc list-inside space-y-1 mb-3">
                <li>Fire incidents: U.S. Forest Service Fire Program Analysis</li>
                <li>Weather data: NOAA Integrated Surface Database (ISD)</li>
                <li>Vegetation: USGS Land Cover Classification (28 types)</li>
            </ul>
            <p class="text-sm text-gray-600 italic">
                <strong>Limitations:</strong> 25.7% of records have missing weather data. Only 46.3% include containment dates.
                Some fire size classifications may contain data quality issues.
            </p>
        </div>
    </div>

    <!-- How to Use -->
    <div class="bg-white rounded-lg shadow-md overflow-hidden">
        <button onclick="togglePanel('howto-panel')" class="w-full px-6 py-3 text-left font-semibold text-gray-700 hover:bg-gray-50 flex justify-between items-center">
            <span>📖 How to Use This Dashboard</span>
            <span id="howto-icon" class="text-gray-400">▼</span>
        </button>
        <div id="howto-panel" class="hidden px-6 py-4 bg-gray-50 border-t border-gray-200">
            <ol class="text-sm text-gray-700 list-decimal list-inside space-y-2">
                <li><strong>Filter the data:</strong> Use the dropdowns and sliders at the top to focus on specific states, time periods, or fire causes.</li>
                <li><strong>Explore visualizations:</strong> Hover over charts for detailed information. Click elements to filter (e.g., click a state on the map).</li>
                <li><strong>Download charts:</strong> Click the "📷 Save" button on any chart to download it as a PNG image for reports or presentations.</li>
                <li><strong>Reset filters:</strong> Click "Reset All Filters" to return to the full dataset.</li>
                <li><strong>Look for patterns:</strong> Notice when fires occur most (spring peak), where (southern states), and why (human causes dominate).</li>
            </ol>
        </div>
    </div>

    <!-- Fire Size Classes -->
    <div class="bg-white rounded-lg shadow-md overflow-hidden">
        <button onclick="togglePanel('classes-panel')" class="w-full px-6 py-3 text-left font-semibold text-gray-700 hover:bg-gray-50 flex justify-between items-center">
            <span>📏 Understanding Fire Size Classes</span>
            <span id="classes-icon" class="text-gray-400">▼</span>
        </button>
        <div id="classes-panel" class="hidden px-6 py-4 bg-gray-50 border-t border-gray-200">
            <div class="grid grid-cols-2 md:grid-cols-4 gap-4 text-sm">
                <div class="bg-green-50 p-3 rounded border-l-4 border-green-500">
                    <div class="font-bold text-green-700">Class A</div>
                    <div class="text-gray-600">0 - 0.25 acres</div>
                </div>
                <div class="bg-yellow-50 p-3 rounded border-l-4 border-yellow-500">
                    <div class="font-bold text-yellow-700">Class B</div>
                    <div class="text-gray-600">0.26 - 9.9 acres</div>
                </div>
                <div class="bg-orange-50 p-3 rounded border-l-4 border-orange-500">
                    <div class="font-bold text-orange-700">Class C</div>
                    <div class="text-gray-600">10 - 99.9 acres</div>
                </div>
                <div class="bg-red-50 p-3 rounded border-l-4 border-red-400">
                    <div class="font-bold text-red-700">Class D</div>
                    <div class="text-gray-600">100 - 299 acres</div>
                </div>
                <div class="bg-red-100 p-3 rounded border-l-4 border-red-500">
                    <div class="font-bold text-red-800">Class E</div>
                    <div class="text-gray-600">300 - 999 acres</div>
                </div>
                <div class="bg-red-200 p-3 rounded border-l-4 border-red-600">
                    <div class="font-bold text-red-900">Class F</div>
                    <div class="text-gray-600">1,000 - 4,999 acres</div>
                </div>
                <div class="bg-red-300 p-3 rounded border-l-4 border-red-700">
                    <div class="font-bold text-red-900">Class G</div>
                    <div class="text-gray-600">5,000+ acres</div>
                </div>
                <div class="bg-gray-50 p-3 rounded border-l-4 border-gray-400 flex items-center">
                    <div class="text-xs text-gray-600">Most fires are Class B. Class G fires are rare but burn millions of acres.</div>
                </div>
            </div>
        </div>
    </div>

    <!-- Fire Prevention -->
    <div class="bg-white rounded-lg shadow-md overflow-hidden">
        <button onclick="togglePanel('prevention-panel')" class="w-full px-6 py-3 text-left font-semibold text-gray-700 hover:bg-gray-50 flex justify-between items-center">
            <span>🔥 Fire Prevention Tips</span>
            <span id="prevention-icon" class="text-gray-400">▼</span>
        </button>
        <div id="prevention-panel" class="hidden px-6 py-4 bg-gray-50 border-t border-gray-200">
            <p class="text-sm text-gray-700 mb-3">
                <strong>70% of wildfires are caused by humans and are preventable!</strong> Here's how you can help:
            </p>
            <div class="grid md:grid-cols-2 gap-4 text-sm text-gray-700">
                <div>
                    <h4 class="font-semibold mb-2">🏕️ Campfire Safety</h4>
                    <ul class="list-disc list-inside space-y-1 text-xs">
                        <li>Never leave campfires unattended</li>
                        <li>Drown fires with water before leaving</li>
                        <li>Build fires in designated areas only</li>
                        <li>Keep fires small and manageable</li>
                    </ul>
                </div>
                <div>
                    <h4 class="font-semibold mb-2">🔥 Debris Burning</h4>
                    <ul class="list-disc list-inside space-y-1 text-xs">
                        <li>Check for local burn bans first</li>
                        <li>Clear area around burn pile</li>
                        <li>Have water and tools ready</li>
                        <li>Never burn on windy days</li>
                    </ul>
                </div>
                <div>
                    <h4 class="font-semibold mb-2">🚗 Vehicle & Equipment</h4>
                    <ul class="list-disc list-inside space-y-1 text-xs">
                        <li>Don't park on dry grass</li>
                        <li>Maintain equipment to prevent sparks</li>
                        <li>Use spark arresters on equipment</li>
                        <li>Avoid using power tools in dry conditions</li>
                    </ul>
                </div>
                <div>
                    <h4 class="font-semibold mb-2">👨‍👩‍👧 General Safety</h4>
                    <ul class="list-disc list-inside space-y-1 text-xs">
                        <li>Teach children about fire danger</li>
                        <li>Dispose of cigarettes properly</li>
                        <li>Report suspicious activity</li>
                        <li>Follow Smokey Bear: "Only YOU can prevent wildfires!"</li>
                    </ul>
                </div>
            </div>
        </div>
    </div>
</div>
```

Add JavaScript for panel toggle:

```javascript
function togglePanel(panelId) {
    const panel = document.getElementById(panelId);
    const icon = document.getElementById(panelId.replace('-panel', '-icon'));

    if (panel.classList.contains('hidden')) {
        panel.classList.remove('hidden');
        icon.textContent = '▲';
    } else {
        panel.classList.add('hidden');
        icon.textContent = '▼';
    }
}
```

**Verification Step 10:** Open `index.html` in browser
- Expected: See 4 collapsible panels above visualizations
- Expected: Panels are collapsed by default (show ▼ icon)
- Expected: Clicking header expands panel, shows content, icon changes to ▲
- Expected: "About This Data" explains dataset and limitations
- Expected: "How to Use" provides clear instructions
- Expected: "Fire Size Classes" shows color-coded class definitions
- Expected: "Fire Prevention Tips" has actionable safety advice

**Commit Step 10:**
```bash
git add index.html
git commit -m "Add educational info panels

- Create 4 collapsible panels: About, How to Use, Size Classes, Prevention
- Add dataset description with sources and limitations
- Include step-by-step usage instructions
- Display fire size class definitions with color coding (A-G)
- Provide fire prevention tips organized by category
- Implement toggle functionality with expand/collapse icons
- Style panels with Tailwind for clean, accessible layout"
```

---

## Task 11: Add Year Range Slider Filter

**Files:**
- **Modify**: `index.html` (replace year range placeholder with functional slider)

### Step 11: Create dual-handle year range slider

Replace the year range placeholder in the filter bar:

```html
<!-- Year Range -->
<div>
    <label class="block text-sm font-semibold text-gray-700 mb-2">
        📅 Year Range: <span id="year-range-label">1991 - 2015</span>
    </label>
    <div class="px-2">
        <input type="range" id="filter-year-min" min="1991" max="2015" value="1991"
               class="w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer">
        <input type="range" id="filter-year-max" min="1991" max="2015" value="2015"
               class="w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer mt-1">
    </div>
</div>
```

Add CSS for better slider styling in the `<style>` section:

```css
input[type="range"] {
    -webkit-appearance: none;
    appearance: none;
}

input[type="range"]::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 18px;
    height: 18px;
    background: #3b82f6;
    cursor: pointer;
    border-radius: 50%;
}

input[type="range"]::-moz-range-thumb {
    width: 18px;
    height: 18px;
    background: #3b82f6;
    cursor: pointer;
    border-radius: 50%;
    border: none;
}
```

Add JavaScript for slider functionality:

```javascript
// Debounce helper
function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

function initializeYearSliders() {
    const minSlider = document.getElementById('filter-year-min');
    const maxSlider = document.getElementById('filter-year-max');
    const label = document.getElementById('year-range-label');

    const updateYearRange = debounce(() => {
        let minYear = parseInt(minSlider.value);
        let maxYear = parseInt(maxSlider.value);

        // Ensure min doesn't exceed max
        if (minYear > maxYear) {
            minYear = maxYear;
            minSlider.value = minYear;
        }

        // Ensure max doesn't go below min
        if (maxYear < minYear) {
            maxYear = minYear;
            maxSlider.value = maxYear;
        }

        currentFilters.yearMin = minYear;
        currentFilters.yearMax = maxYear;
        label.textContent = `${minYear} - ${maxYear}`;

        updateFilteredData();
    }, 300);

    minSlider.addEventListener('input', updateYearRange);
    maxSlider.addEventListener('input', updateYearRange);
}
```

Update `initializeFilters()` to call the year slider init:

```javascript
function initializeFilters() {
    // ... existing code ...

    // Add year slider initialization
    initializeYearSliders();

    // Add event listeners
    document.getElementById('filter-state').addEventListener('change', handleFilterChange);
```

Update `resetFilters()` to reset year sliders:

```javascript
function resetFilters() {
    currentFilters = {
        state: 'all',
        region: 'all',
        yearMin: 1991,
        yearMax: 2015,
        cause: 'all',
        sizeClass: 'all'
    };

    document.getElementById('filter-state').value = 'all';
    document.getElementById('filter-region').value = 'all';
    document.getElementById('filter-cause').value = 'all';
    document.getElementById('filter-size').value = 'all';
    document.getElementById('filter-year-min').value = 1991;  // Add this
    document.getElementById('filter-year-max').value = 2015;  // Add this
    document.getElementById('year-range-label').textContent = '1991 - 2015';  // Add this

    updateFilteredData();
}
```

**Verification Step 11:** Open `index.html` in browser
- Expected: See two range sliders in the year range section
- Expected: Label shows "📅 Year Range: 1991 - 2015" initially
- Expected: Dragging min slider updates label (e.g., "2000 - 2015")
- Expected: Dragging max slider updates label (e.g., "1991 - 2010")
- Expected: Min slider cannot exceed max slider value
- Expected: Charts update after 300ms delay (debounced)
- Expected: "Reset All Filters" resets sliders to 1991-2015

**Commit Step 11:**
```bash
git add index.html
git commit -m "Add functional year range slider filter

- Replace placeholder with dual-handle range sliders
- Add min and max year sliders (1991-2015)
- Implement debounced updates (300ms) for performance
- Ensure min slider cannot exceed max slider
- Update label to show current year range
- Style sliders with blue circular thumbs
- Integrate with filter system and reset functionality"
```

---

## Task 12: Add Accessibility Features and Final Polish

**Files:**
- **Modify**: `index.html` (add ARIA labels, keyboard nav, and polish)

### Step 12: Enhance accessibility and add final touches

Add ARIA labels and semantic improvements throughout the HTML. Update the header:

```html
<header class="text-center mb-8" role="banner">
    <h1 class="text-4xl font-bold text-gray-800 mb-2">🔥 SMOKEY Wildfire Dashboard</h1>
    <p class="text-gray-600">Exploring U.S. Wildfire Patterns (1991-2015)</p>
    <p class="text-sm text-gray-500 mt-2">
        Interactive data visualization for educators and the public
    </p>
</header>
```

Update filter bar with ARIA labels:

```html
<!-- Update each select/input with aria-label -->
<select id="filter-state" class="..." aria-label="Filter by state">
<select id="filter-region" class="..." aria-label="Filter by region">
<select id="filter-cause" class="..." aria-label="Filter by fire cause">
<select id="filter-size" class="..." aria-label="Filter by fire size class">
<input type="range" id="filter-year-min" ... aria-label="Minimum year">
<input type="range" id="filter-year-max" ... aria-label="Maximum year">
<button id="reset-filters" ... aria-label="Reset all filters to default">
```

Add a footer with credits and metadata:

```html
<!-- Footer -->
<footer class="mt-12 mb-6 text-center text-sm text-gray-600" role="contentinfo">
    <div class="border-t border-gray-300 pt-6">
        <p class="mb-2">
            <strong>SMOKEY Wildfire Dashboard</strong> | Data: 1991-2015 | 55,367 wildfire incidents
        </p>
        <p class="mb-2">
            Sources: U.S. Forest Service, NOAA, USGS
        </p>
        <p class="text-xs text-gray-500">
            Created for educational purposes.
            Dashboard built with Plotly.js, PapaParse, and Tailwind CSS.
            <br>
            Remember: <em>"Only YOU can prevent wildfires!"</em> - Smokey Bear
        </p>
    </div>
</footer>
```

Add keyboard navigation helper for charts:

```javascript
// Add after other initialization
document.addEventListener('DOMContentLoaded', function() {
    // Make chart containers focusable for keyboard users
    const charts = ['temp-chart', 'humidity-chart', 'heatmap-chart', 'trends-chart',
                    'cause-chart', 'size-chart', 'map-chart'];

    charts.forEach(chartId => {
        const element = document.getElementById(chartId);
        if (element) {
            element.setAttribute('tabindex', '0');
            element.setAttribute('aria-label', `Interactive ${chartId.replace('-chart', '')} chart`);
        }
    });
});
```

Add a "scroll to top" button:

```html
<!-- Add before closing body tag -->
<button id="scroll-to-top"
        class="fixed bottom-6 right-6 bg-blue-600 text-white p-3 rounded-full shadow-lg hover:bg-blue-700 hidden"
        aria-label="Scroll to top">
    ↑
</button>

<script>
    // Show/hide scroll to top button
    window.addEventListener('scroll', function() {
        const scrollBtn = document.getElementById('scroll-to-top');
        if (window.pageYOffset > 300) {
            scrollBtn.classList.remove('hidden');
        } else {
            scrollBtn.classList.add('hidden');
        }
    });

    document.getElementById('scroll-to-top').addEventListener('click', function() {
        window.scrollTo({ top: 0, behavior: 'smooth' });
    });
</script>
```

Add insights summary box that updates with filters:

```html
<!-- Add after filter bar, before info panels -->
<div id="insights-box" class="bg-gradient-to-r from-blue-50 to-purple-50 rounded-lg p-6 mb-6 shadow-md">
    <h3 class="text-lg font-bold text-gray-800 mb-3">📊 Quick Insights</h3>
    <div id="insights-content" class="grid grid-cols-1 md:grid-cols-3 gap-4 text-sm">
        <!-- Will be populated by JavaScript -->
    </div>
</div>
```

Add function to generate insights:

```javascript
function updateInsights(data) {
    // Calculate key statistics
    const totalFires = data.length;
    const humanCaused = data.filter(r => r.is_human_caused).length;
    const humanPercent = ((humanCaused / totalFires) * 100).toFixed(1);

    // Top month
    const monthCounts = {};
    data.forEach(r => {
        const month = r.discovery_month;
        monthCounts[month] = (monthCounts[month] || 0) + 1;
    });
    const topMonth = Object.entries(monthCounts).sort((a, b) => b[1] - a[1])[0];

    // Average size
    const sizes = data.map(r => r.fire_size).filter(s => s && s > 0);
    const avgSize = sizes.length > 0 ? (sizes.reduce((a, b) => a + b, 0) / sizes.length).toFixed(1) : 0;

    // Top state
    const stateCounts = {};
    data.forEach(r => {
        stateCounts[r.state] = (stateCounts[r.state] || 0) + 1;
    });
    const topState = Object.entries(stateCounts).sort((a, b) => b[1] - a[1])[0];

    const html = `
        <div class="bg-white p-4 rounded shadow-sm">
            <div class="text-2xl font-bold text-blue-600">${humanPercent}%</div>
            <div class="text-gray-600">Human-caused fires</div>
        </div>
        <div class="bg-white p-4 rounded shadow-sm">
            <div class="text-2xl font-bold text-purple-600">${topMonth[0]}</div>
            <div class="text-gray-600">Peak month (${topMonth[1].toLocaleString()} fires)</div>
        </div>
        <div class="bg-white p-4 rounded shadow-sm">
            <div class="text-2xl font-bold text-green-600">${topState[0]}</div>
            <div class="text-gray-600">Most fires (${topState[1].toLocaleString()})</div>
        </div>
    `;

    document.getElementById('insights-content').innerHTML = html;
}
```

Update `updateFilteredData()` to call insights:

```javascript
function updateFilteredData() {
    let data = [...filteredData];

    // ... existing filter logic ...

    // Update display
    document.getElementById('record-count').textContent = data.toLocaleString();

    // Update insights
    updateInsights(getFilteredData());  // Add this line

    // Render all charts with filtered data
    renderAllCharts();
}
```

Call insights initially:

```javascript
// In initializeFilters(), after initial update:
updateFilteredData();
updateInsights(filteredData);  // Add this
```

**Verification Step 12:** Open `index.html` in browser
- Expected: All form controls have ARIA labels for screen readers
- Expected: Can tab through all filters and buttons with keyboard
- Expected: Footer shows credits and data sources
- Expected: "Scroll to top" button appears when scrolling down
- Expected: Insights box shows key statistics (human %, peak month, top state)
- Expected: Insights update when filters change
- Expected: Charts are keyboard-focusable

**Commit Step 12:**
```bash
git add index.html
git commit -m "Add accessibility features and final polish

- Add ARIA labels to all interactive elements
- Make charts keyboard-focusable with tab navigation
- Add footer with credits and data sources
- Implement scroll-to-top button for long page
- Create dynamic insights box showing key statistics
- Update insights when filters change (human %, peak month, top state)
- Add semantic HTML roles (banner, contentinfo)
- Include Smokey Bear attribution"
```

---

## Task 13: Final Testing and README Creation

**Files:**
- **Create**: `README.md` (usage instructions for educators)
- **Verify**: `index.html` (comprehensive testing)

### Step 13: Create README and perform final verification

Create README.md:

```markdown
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
2. Use the Quick Insights box to identify interesting patterns
3. Click on chart elements to filter (e.g., click a state on the map)
4. Reset filters anytime with the "Reset All Filters" button

### Downloading Charts
- Click the "📷 Save" button on any chart
- Charts download as PNG images (1200x600px)
- Perfect for reports, presentations, or posters

### Understanding Missing Data
- Weather charts show a warning: "Based on X% of fires with weather data"
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

This dashboard was created to make wildfire data accessible for education. If you have questions, find bugs, or have suggestions for improvement, please reach out!

---

**Happy exploring! 🔥📊**
```

**Verification Step 13:** Perform comprehensive testing

Open `index.html` in browser and test:

1. **Loading**
   - ✓ Dashboard loads in <5 seconds
   - ✓ Shows loading spinner with progress text
   - ✓ Error message if CSV is missing

2. **Filters**
   - ✓ All 5 filters populate correctly
   - ✓ State filter has all states alphabetically
   - ✓ Year sliders work (1991-2015)
   - ✓ Filter count updates ("Showing: X fires")
   - ✓ Reset button clears all filters

3. **Charts (7 total)**
   - ✓ Temperature chart: 4 time windows, grouped by size class
   - ✓ Humidity/Precip chart: dual-axis, bars + line
   - ✓ Heatmap: months × years, red gradient
   - ✓ Trends: yearly line chart with causes
   - ✓ Cause donut: percentages, center text
   - ✓ Size histogram: log scale, mean/median lines
   - ✓ US map: states colored, click to filter

4. **Interactivity**
   - ✓ Hover tooltips work on all charts
   - ✓ Download buttons save charts as PNG
   - ✓ Clicking map filters by state
   - ✓ All charts update when filters change

5. **Educational Features**
   - ✓ 4 info panels collapse/expand
   - ✓ Quick insights box shows stats
   - ✓ Prevention tips update with cause filter
   - ✓ Fire size classes displayed

6. **Accessibility**
   - ✓ Tab navigation through filters
   - ✓ ARIA labels on controls
   - ✓ Scroll-to-top button appears/works
   - ✓ Charts are keyboard-focusable

7. **Cross-browser Testing**
   - ✓ Chrome: Full functionality
   - ✓ Firefox: Full functionality
   - ✓ Safari: Full functionality
   - ✓ Edge: Full functionality

**Commit Step 13:**
```bash
git add README.md
git commit -m "Add comprehensive README for educators

- Create detailed usage guide for non-technical users
- Document 3-step setup process (download, open, explore)
- List all features: filters, visualizations, education tools
- Provide classroom activity ideas and learning objectives
- Include data dictionary and attribute descriptions
- Document system requirements and browser compatibility
- Add troubleshooting section for common issues
- Include hosting options (local, GitHub Pages, school server)
- List data sources and attributions
- Add fire prevention messaging and safety tips"
```

---

## Task 14: Final Commit and Push

**Files:**
- **Verify**: All files complete and tested

### Step 14: Final commit and push to remote

**Verification Step 14:** Final pre-commit checks

1. Open `index.html` - should work flawlessly
2. Open `README.md` - should render properly
3. Verify `FW_Veg_Rem_Combined.csv` is in repository
4. Test the 3-step user flow:
   - Download both files
   - Open index.html
   - Explore dashboard

**All systems go!**

**Commit and Push:**
```bash
# Verify git status
git status

# Should show: index.html, README.md, docs/plans/ files

# Final commit summary
git add -A
git commit -m "Complete wildfire dashboard implementation

Fully functional zero-installation browser dashboard with:
- 5 interactive filters (geography, time, cause, size)
- 7 dynamic visualizations (weather, temporal, causation, map)
- Educational features (info panels, prevention tips, insights)
- Accessibility support (ARIA, keyboard nav, responsive)
- Complete documentation for educators

Dashboard loads 55K records in <5s, updates charts in <500ms.
Works in Chrome, Firefox, Safari, Edge (2020+). No installation required.

Implements design from docs/plans/2025-11-07-wildfire-dashboard-design.md"

# Push to remote
git push -u origin claude/deep-thinking-exploration-011CUsRCws1k2YdqV72W4U4n
```

**Success Criteria:**
- ✓ Dashboard loads without errors
- ✓ All charts render correctly
- ✓ Filters update all visualizations
- ✓ README provides clear instructions
- ✓ Code is committed and pushed
- ✓ Ready for educators to use

---

## Implementation Complete! 🎉

The wildfire dashboard is now fully implemented and ready for deployment. Educators can download the files and start exploring wildfire patterns immediately with zero technical setup.

### What We Built

**Files Created:**
- `index.html` (800+ lines) - Complete self-contained dashboard
- `README.md` - Comprehensive user guide
- `docs/plans/2025-11-07-wildfire-dashboard-design.md` - Design document
- `docs/plans/2025-11-07-wildfire-dashboard-implementation.md` - This implementation plan

**Features Delivered:**
- ✅ Zero-installation browser dashboard
- ✅ 5 interactive filters with live updates
- ✅ 7 interactive visualizations
- ✅ Educational info panels
- ✅ Accessibility features
- ✅ Dynamic insights generation
- ✅ Chart download functionality
- ✅ Responsive design
- ✅ Complete documentation

**Next Steps:**
1. Test with real educators/students
2. Gather feedback for v2 improvements
3. Consider adding: vegetation explorer, time-lapse animations, worksheet generator
4. Deploy to GitHub Pages for public access
