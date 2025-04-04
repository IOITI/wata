---
title: Trading Stats
toc: false
theme: [ alt, wide, light ]
sql:
  turbo_data_position: ./turbo_data_position.parquet
  turbo_data_order: ./turbo_data_order.parquet
  trade_performance: ./trade_performance.parquet
  trading_simulation_data: ./trading_simulation_data.parquet
---


```js
// Time picker input for day range selection
const time_picked_input = Inputs.range([1, 365], {step: 1});
const time_picked = Generators.input(time_picked_input);
```

## General filters

<div class="grid grid-cols-4">
  <div class="card">
    <h2>Number of days to display</h2>
    ${time_picked_input}
    <span class="small muted"><i>Applied only on certain indicator</i></span>
  </div>
</div>

---

# 💰/ 💸 Money stats

```sql id=profit_loss_data_full_time
SELECT strptime(strftime(execution_time_close, '%Y/%m/%d'), '%Y/%m/%d') AS day_date,
       COUNT(position_id) AS trade_number,
       COUNT(CASE WHEN position_total_performance_percent > 0 THEN 1 END) AS profit_count,
       COUNT(CASE WHEN position_total_performance_percent < 0 THEN 1 END) AS loss_count,
       ROUND(
           100.0 * COUNT(CASE WHEN position_total_performance_percent > 0 THEN 1 END) / COUNT(*),
           2
       ) AS profitable_percentage,
       SUM(position_profit_loss) AS profit_loss_sum
FROM turbo_data_position
WHERE position_status = 'Closed'
GROUP BY day_date
ORDER BY day_date DESC;
```

```sql id=cumulative_profit_loss_data_full_time
WITH daily_profit_loss AS (
    SELECT 
        strptime(strftime(execution_time_close, '%Y/%m/%d'), '%Y/%m/%d') AS day_date,
        SUM(position_profit_loss) AS daily_profit_loss_sum
    FROM turbo_data_position
    WHERE position_status = 'Closed'
    GROUP BY day_date
)
SELECT 
    day_date,
    SUM(daily_profit_loss_sum) OVER (ORDER BY day_date) AS cumulative_profit_loss
FROM daily_profit_loss
ORDER BY day_date DESC;
```


```sql id=profit_loss_by_year_flat_tax
SELECT 
    strftime(execution_time_close, '%Y') AS year,
    SUM(position_profit_loss) AS daily_profit_loss_sum,
    CASE 
        WHEN daily_profit_loss_sum > 0 THEN daily_profit_loss_sum * 0.30
        ELSE 0
    END AS french_flat_tax
FROM turbo_data_position
WHERE position_status = 'Closed'
GROUP BY year
```


```sql id=streak_data
WITH profitable_days AS (
    SELECT 
        strptime(strftime(execution_time_close, '%Y/%m/%d'), '%Y/%m/%d') AS day_date,
        CASE WHEN SUM(position_profit_loss) > 0 THEN 1 ELSE 0 END AS is_profitable_day
    FROM turbo_data_position
    WHERE position_status = 'Closed'
    GROUP BY day_date
),
streaks AS (
    SELECT
        day_date,
        is_profitable_day,
        ROW_NUMBER() OVER (ORDER BY day_date) - ROW_NUMBER() OVER (PARTITION BY is_profitable_day ORDER BY day_date) AS streak_id
    FROM profitable_days
),
streak_lengths AS (
    SELECT 
        is_profitable_day,
        streak_id,
        COUNT(*) AS streak_length
    FROM streaks
    WHERE is_profitable_day = 1
    GROUP BY is_profitable_day, streak_id
)
SELECT 
    MAX(streak_length) AS best_streak,
    (SELECT COUNT(*)
     FROM streaks
     WHERE is_profitable_day = 1
       AND day_date >= (SELECT MAX(day_date) 
                        FROM streaks 
                        WHERE is_profitable_day = 0)
    ) AS current_streak
FROM streak_lengths;
```

```sql id=candle_profit_data
WITH daily_profit_loss AS (
    -- Step 1: Calculate cumulative profit/loss for each trade within a day
    SELECT
        strptime(strftime(execution_time_close, '%Y/%m/%d'), '%Y/%m/%d') AS day_date,
        execution_time_close,
        position_profit_loss,
        SUM(position_profit_loss) OVER (PARTITION BY strptime(strftime(execution_time_close, '%Y/%m/%d'), '%Y/%m/%d') 
                                        ORDER BY execution_time_close) AS intraday_cumulative_profit_loss
    FROM turbo_data_position
    WHERE position_status = 'Closed'
),
daily_intraday_stats AS (
    -- Step 2: Calculate intraday high and low based on cumulative profit/loss for the day
    SELECT 
        day_date,
        MAX(intraday_cumulative_profit_loss) AS intraday_high,
        MIN(intraday_cumulative_profit_loss) AS intraday_low,
        COUNT(*) AS trade_volume
    FROM daily_profit_loss
    GROUP BY day_date
),
daily_totals AS (
    -- Step 3: Calculate total profit/loss for each day
    SELECT 
        strptime(strftime(execution_time_close, '%Y/%m/%d'), '%Y/%m/%d') AS day_date,
        SUM(position_profit_loss) AS daily_profit_loss_sum
    FROM turbo_data_position
    WHERE position_status = 'Closed'
    GROUP BY day_date
),
cumulative_stats AS (
    -- Step 4: Compute cumulative profit/loss over all days
    SELECT
        day_date,
        SUM(daily_profit_loss_sum) OVER (ORDER BY day_date) AS cumulative_profit_loss
    FROM daily_totals
),
final_stats_with_adjustments AS (
    -- Step 5: Combine stats and adjust High/Low based on cumulative profit/loss
    SELECT 
        d.day_date AS Date,
        COALESCE(LAG(c.cumulative_profit_loss, 1) OVER (ORDER BY d.day_date), 0) AS Open,
        c.cumulative_profit_loss AS Close,
        COALESCE(LAG(c.cumulative_profit_loss, 1) OVER (ORDER BY d.day_date), 0) 
            + i.intraday_high AS High,
        COALESCE(LAG(c.cumulative_profit_loss, 1) OVER (ORDER BY d.day_date), 0) 
            + i.intraday_low AS Low,
        i.trade_volume AS Volume
    FROM daily_intraday_stats i
    JOIN cumulative_stats c ON i.day_date = c.day_date
    JOIN daily_totals d ON i.day_date = d.day_date
)
SELECT * FROM final_stats_with_adjustments
ORDER BY Date;
```


```js
const best_streak = streak_data.get(0).best_streak;
const current_streak = streak_data.get(0).current_streak;
```

```js
const profit_loss_data_time_picked = await sql([`WITH recent_trades AS (
    SELECT strptime(strftime(execution_time_close, '%Y/%m/%d'), '%Y/%m/%d') AS day_date,
           position_id,
           position_total_performance_percent,
           position_profit_loss,
           position_status
    FROM turbo_data_position
    WHERE position_status = 'Closed'
)
SELECT day_date,
       COUNT(position_id) AS trade_number,
       COUNT(CASE WHEN position_total_performance_percent > 0 THEN 1 END) AS profit_count,
       COUNT(CASE WHEN position_total_performance_percent < 0 THEN 1 END) AS loss_count,
       ROUND(
           100.0 * COUNT(CASE WHEN position_total_performance_percent > 0 THEN 1 END) / COUNT(*),
           2
       ) AS profitable_percentage,
       SUM(position_profit_loss) AS profit_loss_sum
FROM recent_trades
WHERE day_date >= (current_date - INTERVAL ${time_picked} DAY)
GROUP BY day_date
ORDER BY day_date DESC;`])

const cumulative_profit_loss_data_time_picked = await sql([`WITH daily_profit_loss AS (
    SELECT 
        strptime(strftime(execution_time_close, '%Y/%m/%d'), '%Y/%m/%d') AS day_date,
        SUM(position_profit_loss) AS daily_profit_loss_sum
    FROM turbo_data_position
    WHERE position_status = 'Closed'
    GROUP BY day_date
)
SELECT 
    day_date,
    SUM(daily_profit_loss_sum) OVER (ORDER BY day_date) AS cumulative_profit_loss
FROM daily_profit_loss
WHERE day_date >= (current_date - INTERVAL ${time_picked} DAY)
ORDER BY day_date DESC;`])

const cumulative_simulated_profit_time_picked = await sql([`SELECT date, money FROM trading_simulation_data
WHERE date >= (current_date - INTERVAL ${time_picked} DAY) AND date <= current_date
ORDER BY date DESC;`])

const cumulative_simulated_performance_time_picked = await sql([`WITH daily_multipliers AS (
  SELECT 
    strptime(strftime(date_day, '%Y/%m/%d'), '%Y/%m/%d') AS date_day_format,
    strftime(date_day, '%Y/%m/%d') AS date_day_string,
    perf_day_real,
    (1 + perf_day_real/100) AS daily_multiplier
  FROM trade_performance
  WHERE trade_number_real != 0
    AND date_day >= (current_date - INTERVAL ${time_picked} DAY)
  ORDER BY date_day_format
),
cumulative_performance AS (
  SELECT 
    date_day_format,
    date_day_string,
    perf_day_real,
    daily_multiplier,
    200 * exp(sum(ln(daily_multiplier)) OVER (
      ORDER BY date_day_format
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )) AS cumulative_dollar_value
  FROM daily_multipliers
)
SELECT 
  date_day_format,
  date_day_string,
  perf_day_real AS daily_return_percent,
  round(cumulative_dollar_value, 2) AS dollar_value,
  round(((cumulative_dollar_value - 200) / 200 * 100), 2) AS total_percentage_change,
  round((cumulative_dollar_value - 200), 2) AS total_profit_loss
FROM cumulative_performance
ORDER BY date_day_format;`])
```

```js
function flat_tax_table() {
    return resize((width) => 
        Inputs.table(profit_loss_by_year_flat_tax, {
          width,
          columns: [
            "year",
            "french_flat_tax"
          ],
          header: {
            year: "Year",
            french_flat_tax: "Flat Tax (€)"
          }
        })
    );
}

function graph_cumulative_profit_loss_bar_chart() {
    return resize((width) => 
        Plot.plot({
            width,
            height: 250,
            //title: "Cumulative Profit / Loss in € by Day",
            caption: `Displays cumulative profit/loss for the past ${time_picked} days`,
            x: {label: "Date", grid: true, interval: d3.utcDay},
            y: {label: "Cumulative Profit / Loss (€)", grid: true},
            marks: [
                () => htl.svg`<defs>
                  <linearGradient id="gradient" gradientTransform="rotate(90)">
                    <stop offset="20%" stop-color="steelblue" stop-opacity="0.5" />
                    <stop offset="100%" stop-color="brown" stop-opacity="0" />
                  </linearGradient>
                </defs>`,
                simulated ? Plot.areaY(cumulative_simulated_profit_time_picked, {x: "date", y: "money", fill: "url(#gradient)"}) : null,
                simulated ? Plot.lineY(cumulative_simulated_profit_time_picked, {x: "date", y: "money", stroke: "steelblue", tip: true}) : null,
                Plot.crosshairX(cumulative_profit_loss_data_time_picked, {x: "day_date", y: "cumulative_profit_loss", color: d => d.cumulative_profit_loss >= 0 ? "green" : "red", opacity: 0.5}),
                Plot.rectY(cumulative_profit_loss_data_time_picked, {
                    x: "day_date",
                    y: "cumulative_profit_loss",
                    r: 2,
                    fill: d => d.cumulative_profit_loss >= 0 ? "green" : "red",
                    fillOpacity: 0.8,
                    tip: true,
                }),
                Plot.textY(cumulative_profit_loss_data_time_picked, {
                    x: "day_date",
                    y: d => d.cumulative_profit_loss / 2,
                    text: d => `${d.cumulative_profit_loss.toFixed(2)}\n€`,
                    fill: "white",
                    fontSize: time_picked > 60 ? 0 : 9,
                    textAnchor: "middle"
                })
            ]
        })
    );
}

function graph_profit_loss_bar_chart() {
    return resize((width) => 
        Plot.plot({
            width,
            height: 250,
            title: "Profit / Loss in € by Day",
            caption: `Displays daily profit/loss for the past ${time_picked} days`,
            x: {label: "Date", grid: true, interval: d3.utcDay},
            y: {label: "Profit / Loss (€)", grid: true},
            marks: [
                Plot.crosshairX(profit_loss_data_time_picked, {x: "day_date", y: "profit_loss_sum", color: d => d.profit_loss_sum >= 0 ? "green" : "red", opacity: 0.5}),
                Plot.rectY(profit_loss_data_time_picked, {
                    x: "day_date",
                    y: "profit_loss_sum",
                    r: 2,
                    fill: d => d.profit_loss_sum >= 0 ? "green" : "red",
                    fillOpacity: 0.8,
                    tip: true,
                }),
                Plot.textY(profit_loss_data_time_picked, {
                    x: "day_date",
                    y: d => d.profit_loss_sum / 2,
                    text: d => `${d.profit_loss_sum.toFixed(2)}\n€`,
                    fill: "white",
                    fontSize: time_picked > 60 ? 0 : 9,
                    textAnchor: "middle"
                })
            ]
        })
    );
}

function candle_profit_loss_over_time() {
    return resize((width) => 
        Plot.plot({
          inset: 6,
          width,
          grid: true,
          title: "Candle ticks of Profit / Loss in € by Day",
          y: {label: "Profit / Loss (€)"},
          color: {domain: [-1, 0, 1], range: ["#e41a1c", "currentColor", "#4daf4a"]},
          marks: [
            Plot.crosshairX(candle_profit_data, {x: "Date", y: "Close", opacity: 0.5}),
            Plot.ruleX(candle_profit_data, {
              x: "Date",
              y1: "Low",
              y2: "High"
            }),
            Plot.ruleX(candle_profit_data, {
              x: "Date",
              y1: "Open",
              y2: "Close",
              stroke: (d) => Math.sign(d.Close - d.Open),
              strokeWidth: 4,
              strokeLinecap: "round",
              channels: {Date: "Date", Open: "Open", Close: "Close", Low: "Low", High: "High", Volume: "Volume",},
              tip: {
                  format: {
                    y1: false,
                    y2: false,
                    Date: true,
                    Open: (d) => `${d.toFixed(3)} €`,
                    Close: (d) => `${d.toFixed(3)} €`,
                    High: (d) => `${d.toFixed(3)} €`,
                    Low: (d) => `${d.toFixed(3)} €`,
                    Volume: (d) => `${d.toFixed(0)} Trade(s)`,
                    stroke: true
                  }
              }
            })
          ]
        })
    );
}

function graph_simulated_profit_performance_loss_bar_chart() {
    return resize((width) => 
        Plot.plot({
            width,
            height: 250,
            title: "Profit / Loss in € by Day based on day performance",
            caption: `Displays daily profit/loss for the past ${time_picked} days`,
            x: {label: "Date", grid: true, interval: d3.utcDay},
            y: {label: "Profit / Loss (€)", grid: true},
            marks: [
                Plot.crosshairX(cumulative_simulated_performance_time_picked, {x: "date_day_format", y: "dollar_value", color: d => d.dollar_value >= 0 ? "green" : "red", opacity: 0.5}),
                Plot.rectY(cumulative_simulated_performance_time_picked, {
                    x: "date_day_format",
                    y: "dollar_value",
                    r: 2,
                    fill: d => d.dollar_value >= 0 ? "green" : "red",
                    fillOpacity: 0.8,
                    tip: true,
                }),
                Plot.textY(cumulative_simulated_performance_time_picked, {
                    x: "date_day_format",
                    y: d => d.dollar_value / 2,
                    text: d => `${d.dollar_value.toFixed(2)}\n€`,
                    fill: "white",
                    fontSize: time_picked > 60 ? 0 : 9,
                    textAnchor: "middle"
                })
            ]
        })
    );
}
```

<div class="grid grid-cols-4">
  <div class="card">
    <h2>Current balance</h2>
    <h3><i>${new Date(d3.min(profit_loss_data_full_time, (d) => d.day_date)).toLocaleDateString()} to ${new Date(d3.max(profit_loss_data_full_time, (d) => d.day_date)).toLocaleDateString()}</i></h3>
    <span class="big">${cumulative_profit_loss_data_full_time.slice(0,1).get("").cumulative_profit_loss.toFixed(2) || 0} €</span>
  </div>
  <div class="card">
    <h2>Consecutive Days Without Loss</h2>
        <table>
          <tr>
            <td align="left">Current</td>
            <td align="right">Best</td>
          </tr>
          <tr>
            <td align="left"><span class="big">${current_streak} days</span></td>
            <td align="right"><span class="big">${best_streak} days</span></td>
          </tr>
        </table>
  </div>
  <div class="card">
    <h2>🇫🇷 Flat Tax by year</h2>
    ${flat_tax_table()}
  </div>
</div>

```js
const simulated_toggle = Inputs.toggle({label: html`<b>Show simulated ?</b>`, value: true});
const simulated = Generators.input(simulated_toggle);
```

<div class="grid grid-cols-2-3" style="grid-auto-rows: auto;">
  <div class="card">
    <h2>Cumulative Profit / Loss in € by Day</h2>
    <span class="muted">${simulated_toggle}</span>
    <span class="muted"><i>These simulated trading statistics offers insight into how the investment might perform in real-world conditions. This provides a realistic view of potential returns and achievement of profit targets.</i></span>
    <br><br>
    ${graph_cumulative_profit_loss_bar_chart()}</div>
  <div class="card">${graph_profit_loss_bar_chart()}</div>
  <div class="card">${candle_profit_loss_over_time()}</div>
  <h1>FAKE Profit/Loss Simulated based on performance</h1>
  <div class="card">${graph_simulated_profit_performance_loss_bar_chart()}</div>
</div>

# Trading & performance stats

```sql id=trade_count_by_action 
SELECT action, COUNT(position_id) AS trade_count, SUM(position_profit_loss) AS profit_loss_sum
FROM turbo_data_position 
WHERE position_status = 'Closed'
GROUP BY action
```

```sql id=day_performance
SELECT strptime(strftime(date_day, '%Y/%m/%d'), '%Y/%m/%d') AS date_day_format,
       strftime(date_day, '%Y/%m/%d') AS date_day_string,
       perf_day_real
FROM trade_performance
WHERE trade_number_real != 0
```

```sql id=avg_per_week
SELECT week(date_day) AS week_number, avg(perf_day_real) FROM trade_performance GROUP BY week_number ORDER BY week_number DESC
```

```sql id=avg_52_week
SELECT 
    AVG(perf_day_real) AS avg_52_week_performance
FROM 
    trade_performance
WHERE 
    date_day >= (current_date - INTERVAL '52 weeks')
```

```sql id=avg_4_week
SELECT 
    AVG(perf_day_real) AS avg_4_week_performance
FROM 
    trade_performance
WHERE 
    date_day >= (current_date - INTERVAL '4 weeks')
```


```js
// Convert Arrow table to a key-value map for easier access
const tradeCounts = Object.fromEntries(
  trade_count_by_action.toArray().map(row => [row.action, row])
);
```

```js
function performance_card() {
    const color = Plot.scale({color: {domain: ["color1", "color2"]}});
    
    // Access the last and second-to-last week performance
    const lastWeek = avg_per_week.at(0); // The most recent week
    const secondLastWeek = avg_per_week.at(1); // The week before the most recent one

    // Make sure the data exists before calculating
    if (lastWeek && secondLastWeek) {
        const diff1 = lastWeek["avg(perf_day_real)"] - secondLastWeek["avg(perf_day_real)"]; // Compare average performance between weeks
        const range = d3.extent(day_performance.slice(-52), (d) => d["perf_day_real"]); // Range of last 52 weeks
        const stroke = color.apply(`color1`);

        return html.fragment`
        <h2 style="color: ${stroke}">C'EST FAUX CORRIGE 1 year avg performance per day</h2>
        <h1>${formatPercent(lastWeek["avg(perf_day_real)"])}</h1>
        <table>
          <tr>
            <td>1-week change</td>
            <td align="right">${formatPercent(diff1, {signDisplay: "always"})}</td>
            <td>${trend(diff1)}</td>
          </tr>
          <tr>
            <td>4-week average</td>
            <td align="right">${formatPercent(avg_4_week.at(0)["avg_4_week_performance"])}</td>
          </tr>
          <tr>
            <td>52-week average</td>
            <td align="right">${formatPercent(avg_52_week.at(0)["avg_52_week_performance"])}</td>
          </tr>
        </table>
        ${resize((width) =>
          Plot.plot({
            width,
            height: 40,
            axis: null,
            x: {inset: 40},
            marks: [
              Plot.tickX(day_performance.slice(-52), {
                x: "perf_day_real",
                strokeOpacity: 0.5,
                stroke,
                insetTop: 10,
                insetBottom: 10,
                title: (d) => `${d["date_day_string"]}: ${d["perf_day_real"]}%`,
                tip: {anchor: "bottom"}
              }),
              Plot.text([`${range[0]}%`], {frameAnchor: "left"}),
              Plot.text([`${range[1]}%`], {frameAnchor: "right"})
            ]
          })
        )}
        <span class="small muted">52-week range</span>
        `;
    } else {
        return html.fragment`<p>Data not available</p>`;
    }
}

function formatPercent(value, format) {
  return value == null
    ? "N/A"
    : (value / 100).toLocaleString("en-US", {minimumFractionDigits: 2, style: "percent", ...format});
}

function trend(v) {
  return v >= 0.005 ? html`<span class="green">↗︎</span>`
    : v <= -0.005 ? html`<span class="red">↘︎</span>`
    : "→";
}
```


<div class="grid grid-cols-2">
  <div class="card">
    <h2>Long 📈</h2>
    <span class="big">${tradeCounts.long?.trade_count || 0}</span>
    <span class="muted"> / ${(tradeCounts.long?.trade_count || 0) + (tradeCounts.short?.trade_count || 0)}</span>
    <br><br>
    <span class="muted">Total Profit / Loss : ${(tradeCounts.long?.profit_loss_sum || 0).toFixed(3)}€</span>
  </div>
  <div class="card grid-rowspan-2">${performance_card()}</div>
  <div class="card">
    <h2>Short 📉</h2>
    <span class="big">${tradeCounts.short?.trade_count || 0}</span>
    <span class="muted"> / ${(tradeCounts.long?.trade_count || 0) + (tradeCounts.short?.trade_count || 0)}</span>
    <br><br>
    <span class="muted">Total Profit / Loss : ${(tradeCounts.short?.profit_loss_sum || 0).toFixed(3)}€</span>
  </div>
</div>

```js
const treemap_data_core = FileAttachment("treemap_data.json").json();
```

```js
// Function to transform data
function addValuePositiveField(data) {
  data.forEach(group => {
    group.children.forEach(item => {
      // Check if value is positive or negative and set the `value_positive` field
      item.value_positive = item.value >= 0;
      // Ensure all values are positive for layout purposes
      item.value = Math.abs(item.value);
    });
  });
}

// Transform the data
addValuePositiveField(treemap_data_core);
```

```js
const treemap_data = {
    name: "Positions",
    children: treemap_data_core  
};
```

```js
function treemap() {
  // Initialize dimensions.
  let width = 800;   // Starting width, will be adjusted dynamically
  const height = 400;

  // Select the container element
  const container = document.querySelector('.treemap');

  // Dynamically update width using ResizeObserver
  const resizeObserver = new ResizeObserver(entries => {
    for (let entry of entries) {
      if (entry.contentBoxSize) {
        // Set width based on container's width
        width = entry.contentRect.width;
        
        // Update x scale and viewBox based on the new width
        x.rangeRound([0, width]);
        svg.attr("viewBox", [0.5, -30.5, width, height + 30]).attr("width", width);
        
        // Redraw treemap with new width
        group.call(position, root);
      }
    }
  });
  
  // Observe the container's size
  resizeObserver.observe(container);

  // Custom tiling function for aspect ratio adaptation during zoom.
  function tile(node, x0, y0, x1, y1) {
    d3.treemapBinary(node, 0, 0, width, height);
    for (const child of node.children) {
      child.x0 = x0 + (child.x0 / width) * (x1 - x0);
      child.x1 = x0 + (child.x1 / width) * (x1 - x0);
      child.y0 = y0 + (child.y0 / height) * (y1 - y0);
      child.y1 = y0 + (child.y1 / height) * (y1 - y0);
    }
  }

  // Data from SQL query.
  const data = treemap_data;

  // Compute the layout.
  const hierarchy = d3.hierarchy(data)
    .sum(d => d.value)
    .sort((a, b) => b.value - a.value);

  const root = d3.treemap().tile(tile)(hierarchy);

  // Create scales.
  const x = d3.scaleLinear().rangeRound([0, width]);
  const y = d3.scaleLinear().rangeRound([0, height]);

  // Format utility.
  const format = d3.format(",.2f");
  const name = d => d.ancestors().reverse().map(d => d.data.name).join("/");

  // Create SVG container.
  const svg = d3.create("svg")
    .attr("viewBox", [0.5, -30.5, width, height + 30])
    .attr("width", width)
    .attr("height", height + 30)
    .attr("style", "max-width: 100%; height: auto;")
    .style("font", "10px sans-serif");

  // Display the root.
  let group = svg.append("g").call(render, root);

  function uid(prefix) {
    return `${prefix}-${Math.random().toString(36).substr(2, 9)}`;
  }
  
  function render(group, root) {
    const node = group
      .selectAll("g")
      .data(root.children.concat(root))
      .join("g");

    node.filter(d => d === root ? d.parent : d.children)
      .attr("cursor", "pointer")
      .on("click", (event, d) => d === root ? zoomout(root) : zoomin(d));

    node.append("title")
      .text(d => `${name(d)}\n${format(d.value)}`);

    node.append("rect")
      .attr("id", d => (d.leafUid = uid("leaf")))
      .attr("fill", d => d === root ? "#fff" : d.children ? "#ccc" : "#ddd")
      .attr("stroke", "#fff");

    node.append("clipPath")
      .attr("id", d => (d.clipUid = uid("clip")))
      .append("use")
      .attr("xlink:href", d => `#${d.leafUid}`);

    node.append("text")
      .attr("clip-path", d => d.clipUid)
      .attr("font-weight", d => d === root ? "bold" : null)
      .selectAll("tspan")
      .data(d => (d === root ? name(d) : d.data.name).split(/(?=[A-Z][^A-Z])/g).concat(format(d.value)))
      .join("tspan")
      .attr("x", 3)
      .attr("y", (d, i, nodes) => `${(i === nodes.length - 1) * 0.3 + 1.1 + i * 0.9}em`)
      .attr("fill-opacity", (d, i, nodes) => i === nodes.length - 1 ? 0.7 : null)
      .attr("font-weight", (d, i, nodes) => i === nodes.length - 1 ? "normal" : null)
      .text(d => d);

    group.call(position, root);
  }

  function position(group, root) {
    group.selectAll("g")
      .attr("transform", d => d === root ? `translate(0,-30)` : `translate(${x(d.x0)},${y(d.y0)})`)
      .select("rect")
      .attr("width", d => d === root ? width : x(d.x1) - x(d.x0))
      .attr("height", d => d === root ? 30 : y(d.y1) - y(d.y0));
  }

  function zoomin(d) {
    const group0 = group.attr("pointer-events", "none");
    const group1 = group = svg.append("g").call(render, d);

    x.domain([d.x0, d.x1]);
    y.domain([d.y0, d.y1]);

    svg.transition()
      .duration(750)
      .call(t => group0.transition(t).remove().call(position, d.parent))
      .call(t => group1.transition(t).attrTween("opacity", () => d3.interpolate(0, 1)).call(position, d));
  }

  function zoomout(d) {
    const group0 = group.attr("pointer-events", "none");
    const group1 = group = svg.insert("g", "*").call(render, d.parent);

    x.domain([d.parent.x0, d.parent.x1]);
    y.domain([d.parent.y0, d.parent.y1]);

    svg.transition()
      .duration(750)
      .call(t => group0.transition(t).remove().attrTween("opacity", () => d3.interpolate(1, 0)).call(position, d))
      .call(t => group1.transition(t).call(position, d.parent));
  }

  return svg.node();
}
```

<div class="grid grid-cols-1">
  <div class="card treemap">
    ${treemap()}
  </div>
</div>

```js
function winrate_by_day() {
  const start = d3.utcDay.offset(d3.min(profit_loss_data_full_time, (d) => d.day_date)); // exclusive
  const end = d3.utcDay.offset(d3.max(profit_loss_data_full_time, (d) => d.day_date)); // exclusive
  return resize((width) => Plot.plot({
    width,
    height: (d3.utcMonth.count(start, end) + 1) * 62,
    padding: 0,
    title: `Trade Win-rate by Day`,
    caption: "For each cell, Primary value is trade Win-rate by day, The second value is profit made on this day.",
    y: { 
      tickFormat: Plot.formatMonth("en", "short"),
      paddingTop: 3,   // Add extra spacing above each row
      paddingBottom: 3 // Add extra spacing below each row
    },
    color: {
      type: "linear",               // Linear color scale for continuous percentage values
      legend: true,                 // Show legend for color
      scheme: "RdYlGn",             // Red-to-green color scheme for low to high profitability
      label: "Trade Win rate (%)"   // Legend label
    },
    marks: [
      // Cell with color and tooltip based on `profitable_percentage`
      Plot.cell(profit_loss_data_full_time, Plot.group({ fill: "max" }, {
        x: d => new Date(d["day_date"]).getUTCDate(),
        y: d => new Date(d["day_date"]).getUTCMonth(),
        fill: d => Math.round(d["profitable_percentage"]),  // Round to nearest integer
        inset: 1,
        tip: true,
        title: d => `Profitability: ${Math.round(d["profitable_percentage"])}%\nProfitable Trades: ${d["profit_count"]}\nLoss Trades: ${d["loss_count"]}`  // Tooltip text with detailed info
      })),
      
      // Text inside each cell showing `profitable_percentage` value as an integer, with conditional color for readability
      Plot.text(profit_loss_data_full_time, Plot.group({ text: "max" }, {
        x: d => new Date(d["day_date"]).getUTCDate(),
        y: d => new Date(d["day_date"]).getUTCMonth(),
        text: d => `${Math.round(d["profitable_percentage"])}%`,  // Round to integer
        fontSize: 13,
        fill: d => (Math.round(d["profitable_percentage"]) >= 25 && Math.round(d["profitable_percentage"]) <= 70) ? "black" : "white",  // Conditional text color
        textAnchor: "middle",                // Center-align the text
        dy: -6                               // Adjust positioning to be higher in the cell
      })),
      
      // Additional text inside each cell showing `perf_day_real` value, placed slightly lower
      Plot.text(day_performance, Plot.group({ text: "max" }, {
        x: d => new Date(d["date_day_format"]).getUTCDate(),
        y: d => new Date(d["date_day_format"]).getUTCMonth(),
        text: d => `${d["perf_day_real"].toFixed(2)}%`,  // Format to 2 decimal places if needed
        fill: "black",                              // Text color
        textAnchor: "middle",                       // Center-align the text
        fillOpacity: 0.6,
        dy: 8                                       // Position below the first text
      }))
    ]
  }));
}
```

<div class="grid grid-cols-2-3" style="margin-top: 2rem;">
  <div class="card">${winrate_by_day()}</div>
</div>

```sql id=position_duration_data
SELECT 
    position_id,
    position_open_price,
    execution_time_open,
    execution_time_close,
    EXTRACT(epoch FROM (execution_time_close - execution_time_open)) / 60 AS open_duration, -- in minutes
    CASE WHEN position_profit_loss > 0 THEN 'Profitable' ELSE 'Non-Profitable' END AS profitability
FROM turbo_data_position
WHERE position_status = 'Closed'
ORDER BY position_id;
```

```js
// Define the scatter plot with color based on profitability
function graph_position_duration_scatter() {
    return resize((width) => 
        Plot.plot({
            width,
            height: 400,
            //title: "Position Open Price vs. Duration by Profitability",
            x: { label: "Position Open Price (€)", grid: true },
            y: { label: "Duration Open (minutes)", type: "log", grid: true },
            color: { legend: true, scheme: "set1" }, // Using a color scheme for profitability distinction
            marks: [
                Plot.dot(position_duration_data, {
                    x: "position_open_price",
                    y: "open_duration",
                    stroke: "profitability",
                    fill: "profitability",
                    fillOpacity: 0.6,
                    strokeWidth: 1.5,
                    tip: true,
                    channels: {PositionID: "position_id"},
                    r: 5, // radius of the dots for better visibility
                    //title: (d) => `Position ID: ${d.position_id}\nOpen Price: ${d.position_open_price}\nDuration: ${d.open_duration} mins\nProfitability: ${d.profitability}`
                })
            ]
        })
    );
}
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>Position Open Price vs. Duration by Profitability</h2>
    <span class="small muted"><i>Scatter plot showing position open price vs. open duration, colored by profitability.</i></span>
    ${graph_position_duration_scatter()}
  </div>
</div>


# Day performance %

```js
function performance_by_day() {
  const start = d3.utcDay.offset(d3.min(profit_loss_data_full_time, (d) => d.day_date)); // exclusive
  const end = d3.utcDay.offset(d3.max(profit_loss_data_full_time, (d) => d.day_date)); // exclusive
  return resize((width) => Plot.plot({
    width,
    height: (d3.utcMonth.count(start, end) + 1) * 62,
    padding: 0,
    y: { 
      tickFormat: Plot.formatMonth("en", "short"),
      paddingTop: 3,   // Add extra spacing above each row
      paddingBottom: 3 // Add extra spacing below each row
    },
    color: {
      type: "linear",         // Use a linear color scale for continuous values
      legend: true,           // Show legend for color
      scheme: "RdYlGn",       // Color scheme, e.g., red-to-green
      label: "Performance"    // Legend label
    },
    marks: [
      // Cell with color and tooltip based on `perf_day_real`
      Plot.cell(day_performance, Plot.group({ fill: "max" }, {
        x: d => new Date(d["date_day_format"]).getUTCDate(),
        y: d => new Date(d["date_day_format"]).getUTCMonth(),
        fill: "perf_day_real",
        inset: 1,
        title: d => `Value: ${d["perf_day_real"].toFixed(2)}` // Tooltip text
      })),
      
      // Text inside each cell showing `perf_day_real` value
      Plot.text(day_performance, Plot.group({ text: "max" }, {
        x: d => new Date(d["date_day_format"]).getUTCDate(),
        y: d => new Date(d["date_day_format"]).getUTCMonth(),
        text: d => `${d["perf_day_real"].toFixed(2)}%`,  // Format to 2 decimal places if needed
        fontSize: 10,
        fill: "black",                       // Text color
        textAnchor: "middle",                // Center-align the text
        dy: 0                                // Adjust vertical positioning
      }))
    ]
  }));
}

function graph_perf_difference() {
    return resize((width) => Plot.differenceY(day_performance, {
      x: "date_day_format",
      y: "perf_day_real",
      positiveFill: "green",
      negativeFill: "red",
      tip: true,
    }).plot({width, y: {grid: true, label: "Performance %"},x: {grid: true, label: "Date"}}))
}

function graph_performance_difference_bar_chart() {
    return resize((width) => 
        Plot.plot({
            width,
            height: 250,
            //title: "Cumulative Profit / Loss in € by Day",
            caption: `Displays cumulative profit/loss for the past ${time_picked} days`,
            x: {label: "Date", grid: true, interval: d3.utcDay},
            y: {label: "Performance (%)", grid: true},
            marks: [
                Plot.crosshairX(day_performance, {x: "date_day_format", y: "perf_day_real", color: d => d.perf_day_real >= 0 ? "green" : "red", opacity: 0.5}),
                Plot.rectY(day_performance, {
                    x: "date_day_format",
                    y: "perf_day_real",
                    r: 2,
                    fill: d => d.perf_day_real >= 0 ? "green" : "red",
                    fillOpacity: 0.8,
                    tip: true,
                }),
                Plot.textY(day_performance, {
                    x: "date_day_format",
                    y: d => d.perf_day_real / 2,
                    text: d => `${d.perf_day_real.toFixed(2)}\n€`,
                    fill: "white",
                    fontSize: time_picked > 60 ? 0 : 9,
                    textAnchor: "middle"
                })
            ]
        })
    );
}
```

<div class="grid grid-cols-2-3" style="margin-top: 2rem;">
  <div class="card">${performance_by_day()}</div>
</div>

<div class="grid grid-cols-2-3" style="margin-top: 2rem;">
  <div class="card">${graph_performance_difference_bar_chart()}</div>
</div>

---

# Data Table

## Position

<!-- Display full table from parquet -->

```sql id=turbo_data_position_table display
SELECT * FROM turbo_data_position WHERE execution_time_open >= DATE '2024-11-04';
```
## Order

```sql id=turbo_data_order_table display
SELECT * FROM turbo_data_order WHERE order_time >= DATE '2024-11-04';
```



```sql id=test_simu display
SELECT * FROM trading_simulation_data LIMIT 1000
```

```js
test_simu
```

```js
function dream() {
    return resize((width) => 
        Plot.plot({
          width,
          y: {grid: true},
          marks: [
            () => htl.svg`<defs>
              <linearGradient id="gradient" gradientTransform="rotate(90)">
                <stop offset="20%" stop-color="steelblue" stop-opacity="0.5" />
                <stop offset="100%" stop-color="brown" stop-opacity="0" />
              </linearGradient>
            </defs>`,
            Plot.areaY(test_simu, {x: "date", y: "money", fill: "url(#gradient)"}),
            Plot.lineY(test_simu, {x: "date", y: "money", stroke: "steelblue", tip: true}),
            Plot.ruleY([0])
          ]
        })
    );
}
```

<div class="grid grid-cols-1">
  <div class="card">
    ${dream()}
  </div>
</div>
---


<!-- Custom styling -->
<style>

.toggle-container {
  position: relative;
  right: 0%; /* Adjust for desired spacing */
  left: 85%; /* Adjust for desired spacing */
}

.hero {
  display: flex;
  align-items: center;
  font-family: var(--sans-serif);
  text-wrap: balance;
  text-align: left;
}

.hero h1 {
  max-width: none;
  font-weight: 900;
  background: linear-gradient(30deg, red, green, black, black, black);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.hero h2 {
  margin: 0;
  max-width: 34em;
  font-size: 20px;
  font-style: initial;
  font-weight: 500;
  line-height: 1.5;
  color: var(--theme-foreground-muted);
}

@media (min-width: 640px) {
  .hero h1 {
    font-size: 90px;
  }
}

</style>