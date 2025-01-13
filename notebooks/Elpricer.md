---
jupytext:
  formats: md:myst
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.6
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# Power prices visualised

> [!important] This data is updated once a day around 13.00 UTC+1. Make sure to check the date.

There's 2 graphs for each region:

1. **Hourly Evolution of Electricity Prices**

    This graph displays the hourly prices using a bar chart, providing a clear view of how electricity prices fluctuate throughout the day.

2. **Average Price with Deviation**

    This graph visualizes the average price for each time block using a bar chart, accompanied by markers that indicate the range between the minimum and maximum prices, giving a comprehensive view of the price distribution within each block.

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
tags: [remove-cell]
---
from datetime import datetime, timedelta
from enum import Enum
from typing import Dict, List, Any
from dataclasses import dataclass
import pandas as pd
import requests

class Region(str, Enum):
    LULEA = "SE1"
    SUNDSVALL = "SE2"
    GOTEBORG = "SE3"
    MALMO = "SE4"

@dataclass
class Units:
    """Centralized configuration for units."""
    electricity: str = "öre/kWh"

@dataclass
class Labels:
    """Centralized configuration for labels."""
    units: Units

    @property
    def price_column(self) -> str:
        return f"Price ({self.units.electricity})"

    @property
    def avg_price_column(self) -> str:
        return f"Average Price ({self.units.electricity})"

    @property
    def min_price_column(self) -> str:
        return f"Min Price ({self.units.electricity})"

    @property
    def max_price_column(self) -> str:
        return f"Max Price ({self.units.electricity})"

    @property
    def delivery_start(self) -> str:
        return "Delivery Start"

    @property
    def delivery_end(self) -> str:
        return "Delivery End"

    @property
    def block_name(self) -> str:
        return "Block Name"

    @property
    def hourly_title(self) -> str:
        return "Hourly Day Ahead Prices for {area}"

    @property
    def block_title(self) -> str:
        return "Average Prices for Blocks"

    @property
    def time_label(self) -> str:
        return "Delivery Start Time"

    @property
    def price_label(self) -> str:
        return f"Price ({self.units.electricity})"

    @property
    def hourly_legend(self) -> str:
        return "Hourly Prices"

    @property
    def block_avg_legend(self) -> str:
        return "Average Price"

    @property
    def block_range_legend(self) -> str:
        return "Min-Max Range"

    @property
    def cheapest_block_msg(self) -> str:
        return ("The cheapest block for region {area} is '{block}' with an average price of "
                "{price:.2f} {unit}")

@dataclass
class PriceData:
    """Container for processed price data."""
    hourly_df: pd.DataFrame
    block_df: pd.DataFrame
    cheapest_block: Dict[str, float]

class Fetcher:
    def __init__(self):
        self.cache = {}

    def fetch(self, regions: List[Region]) -> Dict[str, Any]:
        """Fetch data for the specified regions, using cache if available."""
        selected_areas = ",".join([region.value for region in regions])

        # Check if data for these regions is already in the cache
        if selected_areas in self.cache:
            print("Using cached data")
            return self.cache[selected_areas]

        # If no valid cache, fetch new data from the API
        print("Fetching new data")
        tomorrow = (datetime.now() + timedelta(days=1)).strftime('%Y-%m-%d')
        url = f'https://dataportal-api.nordpoolgroup.com/api/DayAheadPrices?date={tomorrow}&market=DayAhead&deliveryArea={selected_areas}&currency=SEK'
        response = requests.get(url)
        response.raise_for_status()  # Ensure proper error handling for HTTP issues
        data = response.json()

        # Store the fetched data in cache
        self.cache[selected_areas] = data
        return data

class PriceProcessor:
    """Processes price data from raw format into DataFrames."""

    def __init__(self, labels: Labels):
        self.labels = labels

    def process_hourly_data(self, data: Dict[str, Any], region: str) -> pd.DataFrame:
        """Process hourly electricity price entries."""
        try:
            hourly_entries = data['multiAreaEntries']
            hourly_data = [
                {
                    self.labels.delivery_start: entry['deliveryStart'],
                    self.labels.delivery_end: entry['deliveryEnd'],
                    self.labels.price_column: entry['entryPerArea'][region] / 10,
                }
                for entry in hourly_entries
            ]

            df = pd.DataFrame(hourly_data)
            df[self.labels.delivery_start] = pd.to_datetime(df[self.labels.delivery_start], utc=True)
            df[self.labels.delivery_end] = pd.to_datetime(df[self.labels.delivery_end], utc=True)

            stockholm_tz = "Europe/Stockholm"
            df[self.labels.delivery_start] = df[self.labels.delivery_start].dt.tz_convert(stockholm_tz)
            df[self.labels.delivery_end] = df[self.labels.delivery_end].dt.tz_convert(stockholm_tz)

            return df
        except KeyError as e:
            raise ValueError(f"Missing required data field: {e}")
        except Exception as e:
            raise RuntimeError(f"Error processing hourly data: {e}")

    def process_block_data(self, data: Dict[str, Any], region: str) -> pd.DataFrame:
        """Process block price aggregates."""
        try:
            block_aggregates = data['blockPriceAggregates']
            block_data = [
                {
                    self.labels.block_name: block['blockName'],
                    self.labels.avg_price_column: block['averagePricePerArea'][region]['average'] / 10,
                    self.labels.min_price_column: block['averagePricePerArea'][region]['min'] / 10,
                    self.labels.max_price_column: block['averagePricePerArea'][region]['max'] / 10,
                }
                for block in block_aggregates
            ]
            return pd.DataFrame(block_data)
        except KeyError as e:
            raise ValueError(f"Missing required data field: {e}")
        except Exception as e:
            raise RuntimeError(f"Error processing block data: {e}")

    def find_cheapest_block(self, block_df: pd.DataFrame) -> Dict[str, Any]:
        """Identify the block with the lowest average price."""
        return block_df.loc[block_df[self.labels.avg_price_column].idxmin()].to_dict()

class Visualizer:
    """Handles visualization logic for price data."""

    @staticmethod
    def plot_hourly_prices(hourly_df: pd.DataFrame, chosen_area: str, labels: Labels) -> None:
        """Create visualization of hourly prices using a step plot."""
        import plotly.graph_objects as go
        import pandas as pd

        fig = go.Figure()

        # Create a bar chart for hourly prices
        fig.add_trace(go.Bar(
            x=hourly_df[labels.delivery_start],
            y=hourly_df[labels.price_column],
            name=labels.hourly_legend,
            marker_color='skyblue'
        ))

        fig.update_layout(
            title=labels.hourly_title.format(area=chosen_area),
            xaxis_title=labels.time_label,
            yaxis_title=labels.price_label,
            xaxis_tickangle=45,
            showlegend=True
        )

        fig.show()

    @staticmethod
    def plot_block_prices(block_df: pd.DataFrame, labels: Labels) -> None:
        """Create visualization of block price averages with error bars."""
        import plotly.graph_objects as go
        import pandas as pd

        fig = go.Figure()

        # Calculate min and max range for error bars
        min_price = block_df[labels.min_price_column].min()
        max_price = block_df[labels.max_price_column].max()

        # Create bar chart for block averages
        fig.add_trace(go.Bar(
            x=block_df[labels.block_name],
            y=block_df[labels.avg_price_column],
            name=labels.block_avg_legend,
            marker_color='skyblue'
        ))

        # Add error bars, ensuring the average price is the center
        fig.add_trace(go.Scatter(
            x=block_df[labels.block_name],
            y=block_df[labels.avg_price_column],
            mode='markers',
            name=labels.block_range_legend,
            marker=dict(color='black'),
            error_y=dict(
                type='data',
                symmetric=False,
                array=block_df[labels.max_price_column] - block_df[labels.avg_price_column],  # Max - Avg for upper error
                arrayminus=block_df[labels.avg_price_column] - block_df[labels.min_price_column]  # Avg - Min for lower error
            )
        ))

        fig.update_layout(
            title=labels.block_title,
            xaxis_title=labels.block_name,
            yaxis_title=labels.price_label,
            showlegend=True,
            yaxis=dict(
                range=[min(min_price, block_df[labels.avg_price_column].min()), max(max_price, block_df[labels.avg_price_column].max())]
            )
        )

        fig.show()

class PriceAnalyzer:
    """The main orchestrator for processing and visualizing price data."""

    def __init__(self, processor: PriceProcessor, visualizer: Visualizer, labels: Labels):
        self.processor = processor
        self.visualizer = visualizer
        self.labels = labels

    def analyze(self, data: Dict[str, Any], region: Region):
        """Main analysis workflow for a single region."""
        try:
            # Process the fetched data for hourly and block prices
            hourly_price_df = self.processor.process_hourly_data(data, region)
            block_price_df = self.processor.process_block_data(data, region)

            # Visualize hourly prices and block prices
            self.visualizer.plot_hourly_prices(hourly_price_df, region, self.labels)
            self.visualizer.plot_block_prices(block_price_df, self.labels)

            # Displaying the cheapest block information using visualization
            cheapest = self.processor.find_cheapest_block(block_price_df)
            # self.visualizer.visualize_cheapest_block(cheapest, region)

        except Exception as e:
            print(f"Error during analysis: {e}")

labels = Labels(Units())
visualizer = Visualizer()
fetcher = Fetcher()
```

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
tags: [remove-cell]
---
data = fetcher.fetch(list(Region))

processor = PriceProcessor(labels=labels)
price_analyzer = PriceAnalyzer(processor=processor, visualizer=visualizer, labels=labels)
```

+++ {"editable": true, "slideshow": {"slide_type": ""}}

## Malmö

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
---
price_analyzer.analyze(data, Region.MALMO)
```

+++ {"editable": true, "slideshow": {"slide_type": ""}}

## Stockholm / Göteborg

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
---
price_analyzer.analyze(data, Region.GOTEBORG)
```

+++ {"editable": true, "slideshow": {"slide_type": ""}}

## Sundsvall

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
---
price_analyzer.analyze(data, Region.SUNDSVALL)
```

+++ {"editable": true, "slideshow": {"slide_type": ""}}

## Luleå

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
---
price_analyzer.analyze(data, Region.LULEA)
```

---

> [!note]
> The price slots visualised according to the regions as defined on [Nordpoolgroup.com](
> https://data.nordpoolgroup.com/map?deliveryDate=latest&currency=SEK&market=DayAhead&mapDataType=Price&resolution=60
> ).
