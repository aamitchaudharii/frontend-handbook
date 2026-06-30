# 05 — Project: Analytics Dashboard

> **"Dashboards fail in a specific, predictable way: they work perfectly with the demo dataset of 50 rows and grind to a halt the moment a real customer points it at a year of production data. The entire discipline of building a good dashboard is anticipating that gap before it becomes a production incident."**

This project guide builds a metrics dashboard: multiple chart widgets, date-range filtering that refetches efficiently, drill-down interactions, responsive grid layout, real-time metric updates, and the specific performance techniques that keep dashboards usable at scale.

---

## 📚 What You'll Build

A dashboard with: a customizable grid of chart widgets (line, bar, pie charts and KPI cards), a global date-range filter affecting all widgets, drill-down from aggregate charts into detailed tables, CSV export, and a real-time "live" mode for metrics that update continuously.

---

## Requirements

```
FUNCTIONAL:
  - Grid of widgets: KPI cards, line charts, bar charts, data tables
  - Global date-range picker that filters all widgets simultaneously
  - Widgets can be added, removed, resized, and rearranged (drag-and-drop)
  - Drill-down: clicking a chart segment shows the underlying detail rows
  - Export any widget's data to CSV
  - Optional "live" mode: metrics update automatically every N seconds

NON-FUNCTIONAL:
  - Dashboard with 12+ widgets must load progressively (not block on the
    slowest widget's data)
  - Date range changes shouldn't trigger 12 simultaneous uncoordinated
    API calls if they can be batched
  - Large time-series datasets (100,000+ points) must render without
    freezing the chart library
  - Widget layout persists across sessions
```

---

## Architecture Overview

```
COMPONENT TREE:
  <DashboardPage>
    <DashboardHeader>
      <DateRangePicker />
      <LiveModeToggle />
      <ExportButton />
    <WidgetGrid>                  (drag-and-drop reorderable grid)
      <KpiCardWidget />
      <LineChartWidget />
      <BarChartWidget />
      <DataTableWidget />
    <DrillDownModal />            (opened on chart segment click)

DATA FLOW:
  Global date range (Context or URL state)
    → each widget's query depends on the date range
    → TanStack Query handles per-widget caching and refetching
    → widgets render independently as their OWN data arrives
      (no single "loading" gate blocking the whole dashboard)
```

---

## Step 1 — Independent Widget Loading (Avoiding the All-or-Nothing Trap)

```jsx
// ❌ NAIVE: one big loading state blocks the ENTIRE dashboard until
// the SLOWEST widget's data arrives
function Dashboard({ dateRange }) {
  const [data, setData] = useState(null);
  const [isLoading, setLoading] = useState(true);

  useEffect(() => {
    setLoading(true);
    Promise.all([
      fetchKpis(dateRange),
      fetchRevenue(dateRange),
      fetchTraffic(dateRange),
      fetchConversions(dateRange), // if THIS is slow, NOTHING renders
    ]).then(([kpis, revenue, traffic, conversions]) => {
      setData({ kpis, revenue, traffic, conversions });
      setLoading(false);
    });
  }, [dateRange]);

  if (isLoading) return <FullPageSpinner />; // user stares at a spinner
  // even though 3 of 4 widgets
  // already have their data
}

// ✅ BETTER: each widget independently fetches and renders its own data
function Dashboard({ dateRange }) {
  return (
    <WidgetGrid>
      <KpiWidget dateRange={dateRange} />
      <RevenueChartWidget dateRange={dateRange} />
      <TrafficChartWidget dateRange={dateRange} />
      <ConversionsWidget dateRange={dateRange} />
    </WidgetGrid>
  );
}

function RevenueChartWidget({ dateRange }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ["revenue", dateRange],
    queryFn: () => analyticsApi.getRevenue(dateRange),
  });

  if (isLoading) return <WidgetSkeleton title="Revenue" />;
  if (error) return <WidgetError title="Revenue" error={error} />;
  return <LineChart data={data} />;
}
```

**Key decision:** each widget owns its own `useQuery` call and renders independently — fast widgets show their data immediately while slow widgets show a skeleton, rather than gating the entire dashboard behind a single `Promise.all`. This is the single highest-impact change for perceived dashboard performance, since most real dashboards have a wide variance in per-widget query speed (a simple KPI count query vs. a complex multi-join revenue breakdown).

---

## Step 2 — Coordinated Date Range Without Request Storms

```typescript
// Global date range lives in URL state (shareable, bookmarkable, survives refresh)
function useDashboardDateRange() {
  const [searchParams, setSearchParams] = useSearchParams();

  const dateRange = useMemo(() => ({
    start: searchParams.get('start') ?? defaultStartDate(),
    end:   searchParams.get('end')   ?? defaultEndDate(),
  }), [searchParams]);

  function setDateRange(newRange: { start: string; end: string }) {
    setSearchParams(prev => {
      const next = new URLSearchParams(prev);
      next.set('start', newRange.start);
      next.set('end', newRange.end);
      return next;
    });
  }

  return [dateRange, setDateRange] as const;
}

// Debounce date range CHANGES from a draggable range slider, so dragging
// the slider doesn't fire a new request on every pixel of movement
function useDebouncedDateRange(rawRange, delay = 400) {
  const [debouncedRange, setDebouncedRange] = useState(rawRange);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedRange(rawRange), delay);
    return () => clearTimeout(timer);
  }, [rawRange, delay]);

  return debouncedRange;
}

// Usage: widgets subscribe to the DEBOUNCED range, not the raw one
function Dashboard() {
  const [rawDateRange, setDateRange] = useDashboardDateRange();
  const dateRange = useDebouncedDateRange(rawDateRange);

  return (
    <>
      <DateRangePicker value={rawDateRange} onChange={setDateRange} />
      {/* All widgets receive the DEBOUNCED dateRange */}
      <WidgetGrid dateRange={dateRange} />
    </>
  );
}
```

**Key decision:** because each widget's query key includes `dateRange`, changing the date range naturally triggers exactly one refetch per widget via TanStack Query's reactivity — no manual coordination needed BEYOND debouncing the input itself, since a draggable date range slider can fire dozens of intermediate values per second while being dragged. Without debouncing, each of those intermediate values would trigger a full round of refetches across every widget.

---

## Step 3 — Drill-Down Interaction Pattern

```jsx
function RevenueChartWidget({ dateRange }) {
  const [drillDown, setDrillDown] =
    (useState < { date: string }) | (null > null);
  const { data } = useQuery({
    queryKey: ["revenue", dateRange],
    queryFn: () => analyticsApi.getRevenue(dateRange),
  });

  function handleBarClick(barData) {
    setDrillDown({ date: barData.date });
  }

  return (
    <>
      <BarChart data={data} onBarClick={handleBarClick} />
      {drillDown && (
        <DrillDownModal
          title={`Revenue detail: ${drillDown.date}`}
          onClose={() => setDrillDown(null)}
        >
          <RevenueDetailTable date={drillDown.date} />
        </DrillDownModal>
      )}
    </>
  );
}

function RevenueDetailTable({ date }) {
  // Separate, more granular query — only fetched when drill-down is opened
  const { data, isLoading } = useQuery({
    queryKey: ["revenue-detail", date],
    queryFn: () => analyticsApi.getRevenueDetail(date),
  });

  if (isLoading) return <TableSkeleton />;
  return <DataTable rows={data} columns={revenueDetailColumns} />;
}
```

**Key decision:** the drill-down detail query is a SEPARATE, more granular `useQuery` call that only fires when the modal actually opens — not pre-fetched alongside the aggregate chart data. This avoids fetching potentially thousands of detail rows for every bar in the chart when the user will likely only drill into one or two of them.

---

## Step 4 — Rendering Large Time-Series Datasets

```jsx
// Problem: a chart library rendering 100,000 raw data points (e.g.,
// per-minute metrics over a year) will choke — too many SVG/Canvas
// elements, too much layout/paint work, often literally freezes the tab

// SOLUTION 1: Server-side or client-side downsampling based on zoom level
function useDownsampledSeries(rawData, targetPointCount = 500) {
  return useMemo(() => {
    if (rawData.length <= targetPointCount) return rawData;

    // Largest-Triangle-Three-Buckets (LTTB) downsampling algorithm
    // preserves visual shape better than naive every-Nth-point sampling
    return lttbDownsample(rawData, targetPointCount);
  }, [rawData, targetPointCount]);
}

// SOLUTION 2: Canvas-based rendering instead of SVG for very dense charts
// SVG: each data point is a DOM element — scales poorly past ~1,000 points
// Canvas: pixels, not DOM elements — scales to tens of thousands of points
// Libraries like uPlot or Chart.js (canvas-based) handle this; Recharts/
// D3-with-SVG do not, by default

// SOLUTION 3: Progressive rendering — render a downsampled overview first,
// then fetch and render full-resolution data only for the user's
// current zoom/pan window
function useProgressiveTimeSeries(dateRange, zoomWindow) {
  const overview = useQuery({
    queryKey: ["metrics-overview", dateRange],
    queryFn: () => analyticsApi.getDownsampled(dateRange, 500), // always coarse
  });

  const detail = useQuery({
    queryKey: ["metrics-detail", zoomWindow],
    queryFn: () => analyticsApi.getRaw(zoomWindow),
    enabled: !!zoomWindow, // only fetch full detail once the user zooms in
  });

  return {
    chartData: detail.data ?? overview.data,
    isShowingDetail: !!detail.data,
  };
}
```

**Key decision:** downsampling using LTTB (Largest-Triangle-Three-Buckets) rather than naive "take every Nth point" preserves visual features like spikes and dips that naive sampling would simply discard — a metric that briefly spiked to 10x normal and then returned to baseline would be invisible under naive sampling if the spike happened to fall on a skipped point, but LTTB specifically selects points that preserve the visual shape of the data.

---

## Step 5 — Widget Layout Persistence

```typescript
interface WidgetLayout {
  id:     string;
  type:   'kpi' | 'line-chart' | 'bar-chart' | 'table';
  x: number; y: number; width: number; height: number;
}

function useDashboardLayout(dashboardId: string) {
  const { data: layout, isLoading } = useQuery({
    queryKey: ['dashboard-layout', dashboardId],
    queryFn:  () => dashboardApi.getLayout(dashboardId),
  });

  const saveLayoutMutation = useMutation({
    mutationFn: (newLayout: WidgetLayout[]) =>
      dashboardApi.saveLayout(dashboardId, newLayout),
  });

  // Debounce layout saves — dragging a widget fires many intermediate
  // position updates; only persist the FINAL position
  const debouncedSave = useMemo(
    () => debounce(saveLayoutMutation.mutate, 1000),
    [saveLayoutMutation.mutate]
  );

  return { layout, isLoading, saveLayout: debouncedSave };
}

function WidgetGrid({ dashboardId }) {
  const { layout, saveLayout } = useDashboardLayout(dashboardId);

  function handleLayoutChange(newLayout: WidgetLayout[]) {
    saveLayout(newLayout); // debounced — won't fire on every drag frame
  }

  return (
    <ResponsiveGridLayout
      layout={layout}
      onLayoutChange={handleLayoutChange}
      // react-grid-layout or similar library handles the drag/resize mechanics
    >
      {layout?.map(widget => (
        <WidgetRenderer key={widget.id} widget={widget} />
      ))}
    </ResponsiveGridLayout>
  );
}
```

---

## Step 6 — Live Mode (Polling vs WebSocket)

```typescript
function useLiveMetrics(dateRange, isLiveMode) {
  return useQuery({
    queryKey: ["metrics", dateRange],
    queryFn: () => analyticsApi.getMetrics(dateRange),
    refetchInterval: isLiveMode ? 5000 : false, // poll every 5s only in live mode
    refetchIntervalInBackground: false, // pause polling when tab isn't visible
  });
}

// For TRUE real-time (sub-second) updates, polling becomes wasteful —
// switch to a WebSocket-pushed delta model instead, similar to the
// connection manager pattern from projects/01-realtime-chat-application.md
function useLiveMetricsWebSocket(dashboardId) {
  const queryClient = useQueryClient();

  useEffect(() => {
    const ws = new WebSocket(
      `wss://api.example.com/dashboards/${dashboardId}/live`,
    );
    ws.onmessage = (e) => {
      const update = JSON.parse(e.data);
      queryClient.setQueryData(["metrics", "live"], (prev) => ({
        ...prev,
        [update.metricKey]: update.value,
      }));
    };
    return () => ws.close();
  }, [dashboardId, queryClient]);
}
```

**Key decision:** `refetchIntervalInBackground: false` ensures polling pauses when the user switches to a different browser tab — there's no value in hammering the API for live updates the user isn't even looking at, and it reduces unnecessary server load across all users who leave dashboard tabs open in the background.

---

## Performance Checklist

```
☐ Each widget fetches and renders independently (no all-or-nothing loading gate)
☐ Date range changes debounced before triggering refetches
☐ Large time-series datasets downsampled (LTTB or similar) before charting
☐ Canvas-based charting for very high point-density visualizations
☐ Drill-down queries are separate and lazy (only fetch when opened)
☐ Live mode polling pauses when tab is in the background
☐ Widget layout drag operations debounced before persisting
```

---

## Extension Ideas

```
- Custom widget builder (let users define their own metric queries)
- Anomaly detection highlighting (flag unusual spikes/dips automatically)
- Scheduled email reports (export + send on a cron schedule)
- Comparison mode (this period vs previous period, overlaid)
- Annotations on charts (mark deploys, incidents, campaigns)
- Role-based widget visibility (different teams see different metrics)
```

---

## 🔗 Related Topics

- [`performance/12-large-data-rendering.md`](../performance/12-large-data-rendering.md) — Large dataset rendering techniques
- [`caching/`](../caching/) — Query caching strategy
- [`challenges/01-build-a-virtualized-list.md`](../challenges/01-build-a-virtualized-list.md) — Related virtualization concepts

---

<div align="center">

**Next:** [`projects/06-infinite-scroll-gallery.md`](./06-infinite-scroll-gallery.md) →

</div>
