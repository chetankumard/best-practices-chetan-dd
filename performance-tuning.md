# React Performance Optimization Guide

The complete practical guide to React performance tuning, profiling, and optimization with modern tools like Lighthouse and Chrome DevTools.

## Table of Contents

- [Core Performance Techniques](#core-performance-techniques)
- [React 18 Concurrent Features](#react-18-concurrent-features)
- [Code Splitting & Lazy Loading](#code-splitting--lazy-loading)
- [Profiling & Monitoring Tools](#profiling--monitoring-tools)
- [Lighthouse Optimization](#lighthouse-optimization)
- [Bundle Analysis](#bundle-analysis)
- [Advanced Techniques](#advanced-techniques)
- [Performance Testing Workflow](#performance-testing-workflow)
- [Checklists](#checklists)

## Core Performance Techniques

### 1. Prevent Unnecessary Re-renders

#### React.memo for Pure Components
```jsx
// Basic usage - prevents re-renders when props haven't changed
const UserCard = React.memo(function UserCard({ user, onEdit }) {
  return (
    <div className="user-card">
      <h3>{user.name}</h3>
      <button onClick={onEdit}>Edit</button>
    </div>
  );
});

// Custom comparison for complex props
const ExpensiveComponent = React.memo(
  function ExpensiveComponent({ data }) {
    return <ComplexVisualization data={data} />;
  },
  (prevProps, nextProps) => {
    // Only re-render if data ID changes
    return prevProps.data.id === nextProps.data.id;
  }
);
```

#### useMemo for Expensive Calculations
```jsx
function DataProcessor({ items, filters }) {
  // Only recalculate when items or filters change
  const processedData = React.useMemo(() => {
    return items
      .filter(item => filters.includes(item.category))
      .sort((a, b) => b.score - a.score)
      .slice(0, 100); // Expensive operation
  }, [items, filters]);

  return <DataTable data={processedData} />;
}

// For multiple expensive calculations
function Dashboard({ rawData, userSettings, dateRange }) {
  const filteredData = React.useMemo(() => {
    return rawData.filter(item =>
      item.date >= dateRange.start && item.date <= dateRange.end
    );
  }, [rawData, dateRange]);

  const aggregatedStats = React.useMemo(() => {
    return calculateComplexStats(filteredData, userSettings);
  }, [filteredData, userSettings]);

  return (
    <div>
      <StatsPanel stats={aggregatedStats} />
      <DataChart data={filteredData} />
    </div>
  );
}
```

#### useCallback for Stable Function References
```jsx
function TodoList({ todos, onDelete, onToggle }) {
  // Stable function reference prevents child re-renders
  const handleDelete = React.useCallback((id) => {
    onDelete(id);
  }, [onDelete]);

  const handleToggle = React.useCallback((id) => {
    onToggle(id);
  }, [onToggle]);

  // Memoized filter function
  const handleFilter = React.useCallback((filterType) => {
    switch (filterType) {
      case 'completed':
        return todos.filter(t => t.completed);
      case 'active':
        return todos.filter(t => !t.completed);
      default:
        return todos;
    }
  }, [todos]);

  return (
    <div>
      {todos.map(todo => (
        <TodoItem 
          key={todo.id} 
          todo={todo} 
          onDelete={handleDelete} 
          onToggle={handleToggle} 
        />
      ))}
    </div>
  );
}
```

### 2. Optimize Component Structure

#### Avoid Inline Objects and Functions
```jsx
// ❌ Bad - Creates new objects on every render
function BadComponent({ items }) {
  return (
    <div>
      {items.map(item => (
        <ItemComponent
          key={item.id}
          item={item}
          style={{ margin: '10px', padding: '5px' }} // New object every render
          onClick={() => handleClick(item.id)} // New function every render
        />
      ))}
    </div>
  );
}

// ✅ Good - Stable references
const ITEM_STYLE = { margin: '10px', padding: '5px' };

function GoodComponent({ items }) {
  const handleClick = React.useCallback((id) => {
    // Handle click
  }, []);

  return (
    <div>
      {items.map(item => (
        <ItemComponent 
          key={item.id} 
          item={item} 
          style={ITEM_STYLE} 
          onClick={handleClick} 
        />
      ))}
    </div>
  );
}
```

## React 18 Concurrent Features

### 1. useTransition for Non-urgent Updates
```jsx
function SearchInterface() {
  const [query, setQuery] = React.useState('');
  const [results, setResults] = React.useState([]);
  const [isPending, startTransition] = React.useTransition();

  const handleSearch = (newQuery) => {
    setQuery(newQuery); // Urgent - update input immediately

    startTransition(() => {
      // Non-urgent - defer expensive filtering
      const filtered = performExpensiveSearch(newQuery);
      setResults(filtered);
    });
  };

  return (
    <div>
      <input
        value={query}
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="Search..."
      />
      {isPending && <div className="loading">Searching...</div>}
      <ResultsList results={results} />
    </div>
  );
}

// Real-world example with filtering
function ProductFilter({ products }) {
  const [filter, setFilter] = React.useState('');
  const [filteredProducts, setFilteredProducts] = React.useState(products);
  const [isPending, startTransition] = React.useTransition();

  const handleFilterChange = (newFilter) => {
    setFilter(newFilter);

    startTransition(() => {
      // Expensive filtering operation
      const filtered = products.filter(product => 
        product.name.toLowerCase().includes(newFilter.toLowerCase()) ||
        product.description.toLowerCase().includes(newFilter.toLowerCase())
      );
      setFilteredProducts(filtered);
    });
  };

  return (
    <div>
      <input
        value={filter}
        onChange={(e) => handleFilterChange(e.target.value)}
        placeholder="Filter products..."
      />
      {isPending && <span>Filtering...</span>}
      <ProductGrid products={filteredProducts} />
    </div>
  );
}
```

### 2. useDeferredValue for Smooth UX
```jsx
function LiveSearch({ searchTerm }) {
  // Defer expensive operations while keeping input responsive
  const deferredSearchTerm = React.useDeferredValue(searchTerm);

  const results = React.useMemo(() => {
    return searchDatabase(deferredSearchTerm);
  }, [deferredSearchTerm]);

  return <SearchResults results={results} />;
}

// Combined with useTransition for complex UIs
function DataAnalytics({ rawData }) {
  const [chartType, setChartType] = React.useState('line');
  const [timeRange, setTimeRange] = React.useState('30d');
  const [isPending, startTransition] = React.useTransition();

  const deferredChartType = React.useDeferredValue(chartType);
  const deferredTimeRange = React.useDeferredValue(timeRange);

  const processedData = React.useMemo(() => {
    return processAnalyticsData(rawData, deferredChartType, deferredTimeRange);
  }, [rawData, deferredChartType, deferredTimeRange]);

  const handleChartChange = (newType) => {
    startTransition(() => {
      setChartType(newType);
    });
  };

  return (
    <div>
      <ChartControls 
        chartType={chartType} 
        onChartTypeChange={handleChartChange} 
        timeRange={timeRange} 
        onTimeRangeChange={setTimeRange} 
      />
      {isPending && <LoadingOverlay />}
      <Chart data={processedData} type={deferredChartType} />
    </div>
  );
}
```

## Code Splitting & Lazy Loading

### 1. Route-Level Code Splitting
```jsx
// Lazy load heavy components
const Dashboard = React.lazy(() => import('./Dashboard'));
const Settings = React.lazy(() => import('./Settings'));
const Reports = React.lazy(() => import('./Reports'));
const Analytics = React.lazy(() => import('./Analytics'));

function App() {
  return (
    <Router>
      <React.Suspense fallback={<LoadingSpinner />}>
        <Routes>
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
          <Route path="/reports" element={<Reports />} />
          <Route path="/analytics" element={<Analytics />} />
        </Routes>
      </React.Suspense>
    </Router>
  );
}

// Custom loading component with better UX
function LoadingSpinner() {
  return (
    <div className="loading-container">
      <div className="spinner" />
      <p>Loading...</p>
    </div>
  );
}
```

### 2. Component-Level Code Splitting
```jsx
// Conditional feature loading
function ConditionalFeature({ shouldLoad, userRole }) {
  const [Component, setComponent] = React.useState(null);
  const [isLoading, setIsLoading] = React.useState(false);

  React.useEffect(() => {
    if (shouldLoad && userRole === 'admin') {
      setIsLoading(true);
      import('./AdminPanel').then(module => {
        setComponent(() => module.default);
        setIsLoading(false);
      });
    }
  }, [shouldLoad, userRole]);

  if (isLoading) return <div>Loading admin panel...</div>;
  return Component ? <Component /> : null;
}

// Dynamic chart loading
function ChartContainer({ data, chartType }) {
  const [ChartComponent, setChartComponent] = React.useState(null);

  React.useEffect(() => {
    let importPromise;

    switch (chartType) {
      case 'line':
        importPromise = import('./LineChart');
        break;
      case 'bar':
        importPromise = import('./BarChart');
        break;
      case 'pie':
        importPromise = import('./PieChart');
        break;
      default:
        return;
    }

    importPromise.then(module => {
      setChartComponent(() => module.default);
    });
  }, [chartType]);

  if (!ChartComponent) {
    return <div>Loading {chartType} chart...</div>;
  }

  return <ChartComponent data={data} />;
}
```

### 3. Library Code Splitting
```jsx
// Split heavy libraries
const LazyRichTextEditor = React.lazy(() =>
  import('react-quill').then(module => ({
    default: module.default
  }))
);

function DocumentEditor({ content, onChange }) {
  const [showEditor, setShowEditor] = React.useState(false);

  if (!showEditor) {
    return (
      <button onClick={() => setShowEditor(true)}>
        Open Rich Editor
      </button>
    );
  }

  return (
    <React.Suspense fallback={<div>Loading editor...</div>}>
      <LazyRichTextEditor value={content} onChange={onChange} />
    </React.Suspense>
  );
}
```

## Profiling & Monitoring Tools

### 1. Chrome DevTools Performance

#### Recording Performance Profiles
1. Open Chrome DevTools (F12)
2. Go to Performance tab
3. Click record button (or reload icon for page load)
4. Interact with your app for 5-10 seconds
5. Stop recording and analyze the timeline

#### Key Metrics to Watch
- **Scripting time** - JavaScript execution (should be < 50% of total time)
- **Rendering time** - Layout and paint operations
- **Long tasks** - Operations > 50ms (break these up)
- **Frame drops** - Missed 60fps targets (causes jank)

### 2. React DevTools Profiler

#### Installation and Setup
```bash
# Install React DevTools browser extension
# Available for Chrome, Firefox, Edge
```

#### Profiler Usage Workflow
1. Open React DevTools
2. Click "Profiler" tab
3. Start recording (record button or ⏺️)
4. Interact with your components
5. Stop recording and analyze

#### Key Profiler Features
- **Flamegraph view** - Visual render hierarchy and timing
- **Ranked view** - Components sorted by render duration
- **Component details** - Props and state change analysis
- **Why did this render?** - Root cause identification

### 3. Performance Monitoring Setup
```jsx
// React Profiler API for custom monitoring
function ProfiledApp() {
  const onRenderCallback = (id, phase, actualDuration, baseDuration, startTime, commitTime) => {
    // Send metrics to analytics
    console.log('Profile data:', {
      id,
      phase,
      actualDuration,
      baseDuration,
      startTime,
      commitTime
    });
  };

  return (
    <React.Profiler id="App" onRender={onRenderCallback}>
      <App />
    </React.Profiler>
  );
}

// why-did-you-render setup (development only)
if (process.env.NODE_ENV === 'development') {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true,
    logOnDifferentValues: true,
  });
}
```

## Lighthouse Optimization

### 1. Core Web Vitals

#### Largest Contentful Paint (LCP) - Target: < 2.5s
```jsx
// Optimize images with modern formats
function OptimizedHeroImage() {
  return (
    <picture>
      <source srcSet="/hero.avif" type="image/avif" />
      <source srcSet="/hero.webp" type="image/webp" />
      <img
        src="/hero.jpg"
        loading="eager" // For above-fold images
        alt="Hero image"
        width={1200}
        height={600}
      />
    </picture>
  );
}

// Preload critical resources
function App() {
  React.useEffect(() => {
    // Preload critical CSS
    const link = document.createElement('link');
    link.rel = 'preload';
    link.href = '/critical.css';
    link.as = 'style';
    document.head.appendChild(link);

    // Preload hero image
    const imgLink = document.createElement('link');
    imgLink.rel = 'preload';
    imgLink.href = '/hero.webp';
    imgLink.as = 'image';
    document.head.appendChild(imgLink);
  }, []);

  return <MainApp />;
}
```

#### First Input Delay (FID) / Interaction to Next Paint (INP) - Target: < 100ms
```jsx
// Reduce JavaScript execution time
function OptimizedComponent() {
  // Avoid blocking main thread
  const heavyCalculation = React.useMemo(() => {
    return performExpensiveOperation();
  }, [dependencies]);

  // Use scheduling APIs for heavy work
  const [data, setData] = React.useState([]);

  React.useEffect(() => {
    const scheduler = new MessageChannel();
    scheduler.port2.onmessage = () => {
      const result = processLargeDataset();
      setData(result);
    };
    scheduler.port1.postMessage(null);
  }, []);

  // Break up long tasks
  const processInChunks = React.useCallback(async (items) => {
    const CHUNK_SIZE = 100;
    const results = [];

    for (let i = 0; i < items.length; i += CHUNK_SIZE) {
      const chunk = items.slice(i, i + CHUNK_SIZE);
      const processed = await new Promise(resolve => {
        setTimeout(() => resolve(processChunk(chunk)), 0);
      });
      results.push(...processed);
    }

    return results;
  }, []);

  return <DataVisualization data={data} />;
}
```

#### Cumulative Layout Shift (CLS) - Target: < 0.1
```css
/* Reserve space for dynamic content */
.skeleton-loader {
  width: 300px;
  height: 200px;
  background: linear-gradient(90deg, #f0f0f0, #e0e0e0, #f0f0f0);
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
}

@keyframes loading {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

/* Use aspect-ratio for images */
.responsive-image {
  aspect-ratio: 16/9;
  width: 100%;
  object-fit: cover;
}

/* Font loading optimization */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap; /* Prevent layout shift */
}
```

### 2. Lighthouse CI Setup

#### GitHub Actions Configuration
```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lhci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build application
        run: npm run build
      
      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          configPath: './lighthouserc.json'
          uploadArtifacts: true
          temporaryPublicStorage: true
```

#### lighthouserc.json Configuration
```json
{
  "ci": {
    "collect": {
      "url": [
        "http://localhost:3000",
        "http://localhost:3000/dashboard",
        "http://localhost:3000/profile"
      ],
      "startServerCommand": "npm start",
      "numberOfRuns": 3,
      "settings": {
        "chromeFlags": "--no-sandbox --disable-dev-shm-usage"
      }
    },
    "assert": {
      "preset": "lighthouse:recommended",
      "assertions": {
        "categories:performance": ["error", {"minScore": 0.9}],
        "categories:accessibility": ["error", {"minScore": 0.9}],
        "categories:best-practices": ["error", {"minScore": 0.9}],
        "categories:seo": ["error", {"minScore": 0.9}],
        "first-contentful-paint": ["warn", {"maxNumericValue": 2000}],
        "largest-contentful-paint": ["error", {"maxNumericValue": 2500}],
        "cumulative-layout-shift": ["error", {"maxNumericValue": 0.1}]
      }
    },
    "upload": {
      "target": "temporary-public-storage"
    },
    "server": {
      "port": 9001,
      "storage": "./lighthouse-reports"
    }
  }
}
```

## Bundle Analysis

### 1. Webpack Bundle Analyzer Setup

#### Installation
```bash
npm install --save-dev webpack-bundle-analyzer

# For Create React App
npm install --save-dev @craco/craco
```

#### CRACO Configuration
```js
// craco.config.js
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  webpack: {
    plugins: {
      add: [
        new BundleAnalyzerPlugin({
          analyzerMode: 'server',
          openAnalyzer: true,
          generateStatsFile: true,
          statsOptions: { source: false }
        })
      ]
    }
  }
};
```

#### Package.json Scripts
```json
{
  "scripts": {
    "analyze": "npm run build && npx webpack-bundle-analyzer build/static/js/*.js",
    "build:analyze": "GENERATE_SOURCEMAP=false craco build",
    "build:stats": "npm run build -- --analyze"
  }
}
```

### 2. Bundle Optimization Strategies

#### Tree Shaking Optimization
```jsx
// ✅ Import only what you need
import { debounce, throttle } from 'lodash-es'; // Good - specific imports
import { format } from 'date-fns'; // Good - tree-shakable

// ❌ Avoid full library imports
import _ from 'lodash'; // Bad - entire library
import * as dateFns from 'date-fns'; // Bad - everything imported

// Babel plugin for automatic optimization
// .babelrc
{
  "plugins": [
    ["import", {
      "libraryName": "antd",
      "libraryDirectory": "es",
      "style": "css"
    }],
    ["import", {
      "libraryName": "date-fns",
      "libraryDirectory": "",
      "camel2DashComponentName": false
    }, "date-fns"]
  ]
}
```

#### Dynamic Imports for Large Libraries
```jsx
// Chart library splitting
function LazyChart({ data, type }) {
  const [ChartComponent, setChartComponent] = React.useState(null);
  const [isLoading, setIsLoading] = React.useState(false);

  React.useEffect(() => {
    if (data?.length > 0) {
      setIsLoading(true);

      // Load chart library only when needed
      import('react-chartjs-2').then(module => {
        setChartComponent(() => module[type] || module.Line);
        setIsLoading(false);
      });
    }
  }, [data, type]);

  if (isLoading) return <div>Loading chart...</div>;
  if (!ChartComponent) return null;

  return <ChartComponent data={data} />;
}

// PDF generation splitting
function PDFExporter({ data }) {
  const handleExport = React.useCallback(async () => {
    const { jsPDF } = await import('jspdf');
    const pdf = new jsPDF();

    // Generate PDF
    pdf.text('Report Data', 10, 10);
    pdf.save('report.pdf');
  }, [data]);

  return <button onClick={handleExport}>Export PDF</button>;
}
```

## Advanced Techniques

### 1. Virtual Scrolling for Large Lists

#### react-window Implementation
```bash
npm install react-window react-window-infinite-loader
```

```jsx
import { FixedSizeList } from 'react-window';

function VirtualizedList({ items }) {
  const Row = React.memo(({ index, style }) => (
    <div style={style} className="list-item">
      <ItemComponent data={items[index]} />
    </div>
  ));

  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={80}
      width="100%"
      overscanCount={5} // Render 5 extra items for smoother scrolling
    >
      {Row}
    </FixedSizeList>
  );
}

// Variable height items
import { VariableSizeList } from 'react-window';

function VariableHeightList({ items }) {
  const getItemSize = React.useCallback((index) => {
    // Return height based on content
    return items[index].expanded ? 120 : 60;
  }, [items]);

  const Row = ({ index, style }) => (
    <div style={style}>
      <ExpandableItem
        data={items[index]}
        onHeightChange={() => {
          // Recalculate sizes if needed
        }}
      />
    </div>
  );

  return (
    <VariableSizeList
      height={600}
      itemCount={items.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Row}
    </VariableSizeList>
  );
}
```

### 2. Image Optimization

#### Modern Image Formats with Fallbacks
```jsx
function OptimizedImage({ src, alt, ...props }) {
  const [imageError, setImageError] = React.useState(false);

  return (
    <picture>
      {!imageError && (
        <>
          <source srcSet={`${src}.avif`} type="image/avif" />
          <source srcSet={`${src}.webp`} type="image/webp" />
        </>
      )}
      <img
        src={`${src}.jpg`}
        alt={alt}
        loading="lazy"
        onError={() => setImageError(true)}
        {...props}
      />
    </picture>
  );
}

// Responsive images with srcset
function ResponsiveImage({ src, alt, sizes = "100vw" }) {
  return (
    <img
      src={`${src}-800.jpg`}
      srcSet={`
        ${src}-400.jpg 400w,
        ${src}-800.jpg 800w,
        ${src}-1200.jpg 1200w,
        ${src}-1600.jpg 1600w
      `}
      sizes={sizes}
      alt={alt}
      loading="lazy"
      style={{ width: '100%', height: 'auto' }}
    />
  );
}
```

#### Intersection Observer for Lazy Loading
```jsx
function LazyImage({ src, alt, placeholder }) {
  const [imageSrc, setImageSrc] = React.useState(placeholder);
  const [isLoaded, setIsLoaded] = React.useState(false);
  const imgRef = React.useRef();

  React.useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setImageSrc(src);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, [src]);

  return (
    <img
      ref={imgRef}
      src={imageSrc}
      alt={alt}
      onLoad={() => setIsLoaded(true)}
      className={`lazy-image ${isLoaded ? 'loaded' : 'loading'}`}
    />
  );
}
```

### 3. Service Workers for Caching
```js
// public/sw.js
const CACHE_NAME = 'react-app-v1';
const urlsToCache = [
  '/',
  '/static/js/bundle.js',
  '/static/css/main.css',
  '/manifest.json'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => cache.addAll(urlsToCache))
  );
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((response) => {
        // Return cached version or fetch from network
        return response || fetch(event.request);
      })
  );
});

// Register in index.js
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js');
}
```

## Performance Testing Workflow

### 1. Development Workflow
```bash
# Install performance tools
npm install --save-dev @lhci/cli why-did-you-render web-vitals

# Performance monitoring setup
npm install --save-dev @craco/craco webpack-bundle-analyzer
```

#### Package.json Scripts
```json
{
  "scripts": {
    "dev": "craco start",
    "build": "craco build",
    "analyze": "npm run build && npx webpack-bundle-analyzer build/static/js/*.js",
    "lighthouse": "lhci autorun",
    "lighthouse:desktop": "lighthouse http://localhost:3000 --view --preset=desktop",
    "perf:audit": "npm run lighthouse && npm run analyze",
    "test:perf": "npm run build && npm run lighthouse"
  }
}
```

### 2. Real User Monitoring (RUM)
```jsx
// Web Vitals monitoring
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

function sendToAnalytics({ name, value, id }) {
  // Send to your analytics service (GA, Mixpanel, etc.)
  if (typeof gtag !== 'undefined') {
    gtag('event', name, {
      event_category: 'Web Vitals',
      event_label: id,
      value: Math.round(name === 'CLS' ? value * 1000 : value),
      non_interaction: true,
    });
  }

  // Or send to custom endpoint
  fetch('/api/vitals', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name, value, id, url: location.href })
  });
}

// Initialize monitoring
getCLS(sendToAnalytics);
getFID(sendToAnalytics);
getFCP(sendToAnalytics);
getLCP(sendToAnalytics);
getTTFB(sendToAnalytics);

// Custom performance observer
function observePerformance() {
  if ('PerformanceObserver' in window) {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.entryType === 'navigation') {
          console.log('Navigation timing:', {
            domContentLoaded: entry.domContentLoadedEventEnd - entry.domContentLoadedEventStart,
            loadComplete: entry.loadEventEnd - entry.loadEventStart,
            pageLoad: entry.loadEventEnd - entry.fetchStart
          });
        }
      }
    });

    observer.observe({ entryTypes: ['navigation', 'paint'] });
  }
}

observePerformance();
```

### 3. Performance Budget Configuration
```js
// webpack.config.js or craco.config.js
module.exports = {
  performance: {
    maxAssetSize: 250000, // 250kb
    maxEntrypointSize: 400000, // 400kb
    hints: 'warning'
  },
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          maxSize: 244000 // ~240kb
        }
      }
    }
  }
};
```

## Checklists

### Development Phase Checklist
- [ ] Use React.memo() for pure components
- [ ] Implement useMemo for expensive calculations  
- [ ] Use useCallback for event handlers
- [ ] Avoid inline objects/functions in JSX
- [ ] Add code splitting with React.lazy()
- [ ] Optimize images (WebP/AVIF, lazy loading)
- [ ] Implement virtual scrolling for large lists (>100 items)
- [ ] Use React 18 concurrent features (useTransition, useDeferredValue)
- [ ] Set up proper key props for list items
- [ ] Minimize context provider scope

### Testing Phase Checklist
- [ ] Run Lighthouse audits (target 90+ scores)
- [ ] Analyze bundles with webpack-bundle-analyzer
- [ ] Profile with React DevTools Profiler
- [ ] Test on slow devices/networks (CPU 4x slowdown)
- [ ] Monitor Core Web Vitals in development
- [ ] Check for memory leaks in DevTools
- [ ] Validate accessible performance (screen readers)
- [ ] Test offline functionality with service workers

### Production Phase Checklist
- [ ] Set up Lighthouse CI in GitHub Actions
- [ ] Implement Real User Monitoring (RUM)
- [ ] Track bundle size over time
- [ ] Set performance budgets (< 250kb initial bundle)
- [ ] Configure CDN and compression (gzip/brotli)
- [ ] Implement proper caching headers
- [ ] Monitor Core Web Vitals in production
- [ ] Set up performance alerts and dashboards

### Performance Budget Guidelines
- **Initial Bundle Size**: < 170kb (gzipped)
- **Total Page Size**: < 500kb (gzipped)
- **Time to Interactive**: < 3.8s (3G)
- **First Contentful Paint**: < 1.8s
- **Largest Contentful Paint**: < 2.5s
- **First Input Delay**: < 100ms
- **Cumulative Layout Shift**: < 0.1

## Additional Resources

### Essential Tools
- **React DevTools Profiler** - Component performance analysis
- **Chrome DevTools Performance** - Runtime performance profiling  
- **Lighthouse CI** - Automated performance testing
- **webpack-bundle-analyzer** - Bundle size analysis
- **why-did-you-render** - Re-render debugging (dev only)
- **web-vitals** - Core Web Vitals monitoring

### Performance Libraries
- **react-window** - Virtual scrolling for large lists
- **react-intersection-observer** - Lazy loading and viewport detection
- **workbox** - Service worker and caching strategies
- **comlink** - Web Workers for heavy computations

### Monitoring Services
- **Google Analytics** - Core Web Vitals tracking
- **Sentry** - Error and performance monitoring
- **DataDog RUM** - Real User Monitoring
- **New Relic Browser** - Full-stack performance monitoring

---

## Final Notes

Remember the golden rule: **Measure first, optimize second**. Always:

1. **Profile before optimizing** - Use DevTools to identify actual bottlenecks
2. **Set performance budgets** - Define clear metrics and stick to them  
3. **Test on real devices** - Don't just test on your high-end development machine
4. **Monitor continuously** - Performance is not a one-time task

Performance optimization is an iterative process. Start with the biggest wins (code splitting, image optimization, unnecessary re-renders) and work your way down to micro-optimizations.
