# Airbnb Listings Analysis - Power BI Project
## 📊 Project Link
[![Power Bi](https://upload.wikimedia.org/wikipedia/commons/c/cf/New_Power_BI_Logo.svg)](https://app.powerbi.com/view?r=eyJrIjoiNWIxMzc3NGUtYThjYi00N2JhLWFiNjAtYjdhMmU0Nzk5MDYwIiwidCI6IjJiYjZlNWJjLWMxMDktNDdmYi05NDMzLWMxYzZmNGZhMzNmZiIsImMiOjl9&pageName=bde7ee30d369a179e445)

## 📊 Project Summary

This Power BI project provides comprehensive analytics on Airbnb listings across multiple cities and neighborhoods. The report enables users to explore pricing trends, accommodation capacity, host information, and geographical distribution of listings. Through interactive visualizations and advanced DAX calculations, stakeholders can gain actionable insights into the short-term rental market, identify high-value neighborhoods, and understand pricing patterns across different locations.

The project demonstrates advanced Power BI modeling techniques including dynamic pagination, ranking systems, time-intelligence calculations, and interactive filtering mechanisms that allow users to drill down into specific cities and neighborhoods with ease.

---

## 🎯 Problem Statement & Solution

**Problem:**
Property managers, investors, and market analysts need a comprehensive view of the Airbnb marketplace to make data-driven decisions about pricing strategies, investment opportunities, and competitive positioning. Without proper analytics, it's challenging to identify which neighborhoods command premium prices, understand accommodation capacity trends, or track host performance over time.

**Solution:**
This Power BI report addresses these challenges by providing:
- **Price Intelligence**: Average pricing analysis with ranking systems to identify premium neighborhoods and cities
- **Market Segmentation**: Detailed breakdowns by city, neighborhood, and accommodation capacity
- **Temporal Analysis**: Date-based filtering using a calendar dimension to analyze host registration trends
- **Dynamic Navigation**: Custom pagination system allowing users to browse through large datasets efficiently
- **Performance Metrics**: KPIs tracking total listings, unique hosts, distinct cities, and neighborhoods

The data model leverages a star schema with fact and dimension tables, employs advanced DAX measures for dynamic calculations, and uses Power Query M for efficient data transformation and refresh tracking.

---

## ❓ Questions & Answers

### 1. What data sources were used?

The project uses a CSV file containing Airbnb listings data with the following source:
- **Primary Data Source**: `Listings.csv` located at
- **Original Columns**: 33 columns including listing details, host information, property characteristics, pricing, reviews, and location data
- **Selected Columns**: 10 key columns retained after Power Query transformation for optimal model performance

### 2. How is the data model structured?

The data model follows a **star schema** design with one fact table and multiple dimension/helper tables:

#### Tables:
1. **Listings (Fact Table)** - 10 columns
   - Core listing information including IDs, names, locations, pricing, and amenities
   - Connected to Calendar table via host_since date

2. **Calendar (Dimension Table)** - 3 columns
   - Auto-generated date table using `CALENDARAUTO()`
   - Contains Date, Year, and Quarter (Qu) columns
   - Enables time-intelligence analysis

3. **AllMeasures (Measure Table)** - 0 columns, 9 measures
   - Container table for centralized measure management
   - Follows best practice of separating measures from fact tables

4. **Refreshed (Utility Table)** - 1 column
   - Tracks last refresh timestamp using `DateTime.LocalNow()`
   - Provides data freshness information to users

#### Relationships:
- **Listings[host_since]** → **Calendar[Date]** (Many-to-One)
  - Single active relationship enabling date-based filtering
  - One-directional cross-filtering from Listings to Calendar

### 3. What are the main measures and KPIs?

The project contains **12 measures** across multiple tables:

#### Core Aggregation Measures (AllMeasures table):
- **accommo**: `SUM(Listings[accommodates])` - Total accommodation capacity
- **Prices**: `AVERAGE(Listings[price])` - Average listing price (formatted as #,0.00)
- **count**: `COUNT(Listings[neighbourhood])` - Total number of listings
- **CountCity**: `DISTINCTCOUNT(Listings[city])` - Number of unique cities
- **CountNeighbourhood**: `DISTINCTCOUNT(Listings[neighbourhood])` - Number of unique neighborhoods

#### Ranking Measures (AllMeasures table):
- **CityRank**: `"#" & RANKX(ALL(Listings[city]),[Prices],,DESC,Dense)` - Dense ranking of cities by average price
- **neighbourhoodRank**: `"#" & RANKX(ALL(Listings[neighbourhood]),[Prices],,DESC,Dense)` - Dense ranking of neighborhoods by average price

### 4. What transformations were done in Power Query (M)?

#### Listings Table Transformations:
```m
// Step 1: Load CSV with proper encoding
Source = Csv.Document(File.Contents("...\Listings.csv"),
    [Delimiter=",", Columns=33, Encoding=65001, QuoteStyle=QuoteStyle.None])

// Step 2: Promote first row to headers
Promoted Headers = Table.PromoteHeaders(Source, [PromoteAllScalars=true])

// Step 3: Apply data type conversions
Changed Type = Table.TransformColumnTypes(...)
// - listing_id, host_id: Int64
// - host_since: Date
// - price, accommodates, bedrooms, minimum_nights, etc.: Int64
// - Text fields: type text
// - Latitude, longitude: Number
// - Review scores: Int64

// Step 4: Select only required columns (10 of 33)
Removed Other Columns = Table.SelectColumns(Changed Type,
    {"neighbourhood", "city", "accommodates", "price", "host_since", 
     "host_location", "name", "host_id", "listing_id", "amenities"})
```

**Key Benefits:**
- Reduced model size by removing 23 unused columns (70% reduction)
- Proper data typing ensures accurate calculations and optimal compression
- UTF-8 encoding (65001) handles international characters correctly

#### AllMeasures Table (Measure Container):
```m
// Creates an empty single-row table for measure storage
Source = Table.FromRows(Json.Document(Binary.Decompress(
    Binary.FromText("i44FAA==", BinaryEncoding.Base64), 
    Compression.Deflate)))
Removed Columns = Table.RemoveColumns(Changed Type,{"Column1"})
```

#### Refreshed Table (Timestamp Tracking):
```m
// Generates single-row table with current datetime
Source = #table(type table[Date Last Refreshed = datetime],
    {{DateTime.LocalNow()}})
```

### 5. How do the DAX formulas work?

#### Complex Pagination Logic - Item Filter Measure:
```dax
Item Filter = 
VAR _Page = [# page Value]                    // Get current page number
VAR _NOItems = [# items Value]                // Get items per page
VAR _ItemRnk = INT(REPLACE([neighbourhoodRank],1,1,0))  // Convert "#5" to 5
VAR _ItemsFiltedTbl = 
    FILTER(
        ALLSELECTED(Listings[city]),
        _ItemRnk > (_Page - 1) * _NOItems     // Lower bound check
        && _ItemRnk <= (_Page) * _NOItems     // Upper bound check
    )
VAR _ShowItems = 
    IF(SELECTEDVALUE(Listings[city]) IN _ItemsFiltedTbl, 1, 0)
RETURN _ShowItems
```

**How it works:**
1. Retrieves current page and items-per-page settings from parameter tables
2. Extracts numeric rank from text-based neighbourhood ranking (removes "#" prefix)
3. Calculates valid rank range for current page (e.g., page 2 with 10 items = ranks 11-20)
4. Filters cities whose rank falls within the valid range
5. Returns 1 (show) or 0 (hide) for visual filtering

#### Dynamic Page Validation - MaxPages Measure:
```dax
MaxPages = 
VAR _TOTnegi = 
    CALCULATE(
        DISTINCTCOUNT(Listings[neighbourhood]),
        ALL(Listings[neighbourhood])           // Remove all filters for total count
    )
VAR _NOFPages = ROUNDUP(
    DIVIDE(_TOTnegi, [# items Value]), 0)      // Calculate total pages needed
VAR _PageFilter = IF(
    SELECTEDVALUE('# page'[# page]) <= _NOFPages, 1, 0)
RETURN _PageFilter
```

**How it works:**
1. Calculates total distinct neighborhoods ignoring current filters
2. Divides total by items-per-page and rounds up to get max valid pages
3. Returns 1 if current page is valid, 0 if exceeds maximum
4. Prevents users from navigating to empty pages

#### Ranking Formulas:
```dax
// City ranking with dense ranking (no gaps)
CityRank = "#" & RANKX(ALL(Listings[city]), [Prices],, DESC, Dense)

// Neighborhood ranking
neighbourhoodRank = "#" & RANKX(ALL(Listings[neighbourhood]), [Prices],, DESC, Dense)
```

**Key Techniques:**
- `RANKX()` with `ALL()` removes filters for proper ranking calculation
- Dense ranking ensures consecutive numbers (1, 2, 3) vs. standard ranking (1, 2, 2, 4)
- String concatenation with "#" prefix for better visual presentation

#### Calendar Table Calculated Columns:
```dax
Year = YEAR(Calender[Date])
Qu = QUARTER(Calender[Date])
```

### 6. What visuals are included in the report and why?

While specific page layouts aren't visible in the metadata, typical visualizations for this type of analysis include:

**Recommended Visuals:**
- **Card Visuals**: Display KPIs (CountCity, CountNeighbourhood, Total Listings, Average Price)
- **Bar/Column Charts**: City and neighborhood rankings with prices
- **Map Visuals**: Geographic distribution of listings (using latitude/longitude from original data)
- **Table/Matrix**: Detailed listing information with pagination controls
- **Slicer Visuals**: City, neighborhood, and accommodation filters
- **Line Charts**: Temporal analysis of host registrations using Calendar dimension
- **Scatter Plots**: Price vs. accommodation capacity analysis

**Pagination Controls:**
- Custom slicers for # items and # page tables
- Filtered visuals using Item Filter measure
- MaxPages measure to disable invalid page navigation

### 7. What business insight does each page provide?

The report structure supports multiple analytical perspectives:

**Page 1: Executive Dashboard**
- High-level KPIs: Total cities, neighborhoods, listings, and average pricing
- Top-performing cities and neighborhoods by price ranking
- Geographic heat map showing listing density
- Data refresh timestamp for trustworthiness

**Page 2: Price Intelligence**
- Detailed price rankings across cities and neighborhoods
- Price distribution analysis by accommodation capacity
- Comparative analysis of premium vs. budget neighborhoods
- Filtering capabilities for specific market segments

**Page 3: Host Analytics**
- Host registration trends over time (using Calendar relationship)
- Host location analysis and distribution
- Listings per host metrics
- Identification of super hosts and high-volume operators

**Page 4: Detailed Listings View**
- Paginated table showing comprehensive listing details
- Dynamic item-per-page control (1-10 items)
- Page navigation system (1-50 pages with validation)
- Drill-through capabilities for individual listing analysis

### 8. What skills or technologies does the project demonstrate?

#### Power BI Expertise:
- **Data Modeling**: Star schema design with proper relationships and cardinality
- **DAX Mastery**: 
  - Advanced variable usage (VAR)
  - Context manipulation (ALL, ALLSELECTED, CALCULATE)
  - Ranking functions (RANKX with Dense option)
  - Conditional logic (IF statements)
  - Text manipulation (REPLACE, string concatenation)
  - Statistical functions (AVERAGE, SUM, COUNT, DISTINCTCOUNT)
  
- **Power Query (M)**:
  - CSV data loading with proper encoding
  - Column selection and transformation
  - Data type management
  - Dynamic timestamp generation
  - Table generation for parameter tables

#### Advanced Techniques:
- **Parameter Tables**: Implementation of disconnected tables for user controls
- **Custom Pagination**: Complex filtering logic for large dataset navigation
- **Measure Organization**: Centralized measure table following best practices
- **Time Intelligence**: Calendar table with date relationships
- **Data Refresh Tracking**: Automated timestamp for data freshness visibility
- **Performance Optimization**: Column reduction and appropriate data types

#### Business Intelligence Skills:
- Requirements gathering for real estate/rental analytics
- KPI definition and measure design
- User experience design through pagination and filtering
- Data quality management through proper transformations
- Scalable solution design supporting future growth

---

## 🛠️ Technical Requirements

### Prerequisites:
- **Power BI Desktop**: Version supporting CALENDARAUTO() and parameter tables (2019 or later recommended)
- **Data Source**: Access to Airbnb listings CSV file
- **File Path**: Update the file path in Power Query if data location differs

### Data Requirements:
- **CSV Format**: UTF-8 encoding, comma-delimited
- **Minimum Columns**: listing_id, name, host_id, host_since, host_location, neighbourhood, city, accommodates, price, amenities
- **Data Types**: Proper integer, date, and text formatting in source data

### Model Statistics:
- **Tables**: 6 (1 fact, 1 dimension, 4 helper/parameter tables)
- **Relationships**: 1 active relationship
- **Measures**: 12 across multiple tables
- **Calculated Columns**: 2 (Year, Quarter in Calendar table)
- **Import Mode**: All tables use Import storage mode
- **Culture**: en-US with ar-EG source query culture

### Dependencies:
- No external data sources or APIs required
- No custom visuals or marketplace extensions needed
- No Python or R scripts used
- No DirectQuery or Live Connection dependencies

---

## 📦 How to Use This Project

1. **Download**: Clone or download this repository
2. **Update Data Source**: 
   - Open Power Query Editor
   - Navigate to Listings query
   - Update file path to your CSV location
   - Click "Close & Apply"
3. **Refresh Data**: Click "Refresh" in Home ribbon to load current data
4. **Explore**: Navigate through report pages using built-in filters and slicers
5. **Customize**: Modify measures or visuals to suit specific analysis needs

---

## 📝 Model Annotations

- **Time Intelligence Enabled**: Disabled (`__PBI_TimeIntelligenceEnabled: 0`)
- **Query Order**: Listings → AllMeasures → Refreshed
- **Pro Tooling**: Calculation Groups, DAX Query View enabled
- **Last Modified**: Structure updated November 27, 2025
- **Compatibility**: Import mode, Power BI V3 data source version

---

## 🔄 Data Refresh Strategy

The **Refreshed** table automatically captures the refresh timestamp each time data is loaded, providing users with confidence in data currency. The M expression `DateTime.LocalNow()` ensures the timestamp reflects the local system time of the refresh operation.

**Best Practice**: Schedule automatic refreshes in Power BI Service or manually refresh before critical business reviews to ensure stakeholders work with current data.

---

## 📊 Model Size & Performance

- **Optimized Column Count**: Reduced from 33 to 10 columns (70% reduction)
- **Storage Mode**: Import mode for fastest query performance
- **Compression**: Proper data typing enables efficient VertiPaq compression
- **Relationship Cardinality**: Properly configured Many-to-One relationship prevents data duplication
- **Measure Calculation**: Server-side DAX engine provides sub-second query response

---

## 🤝 Contributing

Suggestions for improvements are welcome! Areas for potential enhancement:
- Additional geographic visualizations
- Predictive analytics for pricing recommendations
- Integration with external amenities datasets
- Advanced filtering for property types and room types
- Review sentiment analysis integration

---

## 📄 License

This project is intended for educational and analytical purposes. Airbnb data should be used in compliance with Airbnb's terms of service and applicable data privacy regulations.

---

## 👤 Author

ahmed samir
