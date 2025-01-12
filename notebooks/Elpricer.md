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

# Data Insights

Tomorrow's price slots visualised according to the regions as defined on [Nordpoolgroup.com](
https://data.nordpoolgroup.com/map?deliveryDate=2025-01-12&currency=SEK&market=DayAhead&mapDataType=Price&resolution=60
)

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
tags: [remove-input]
---
from datetime import datetime, timedelta
from enum import Enum
from typing import Dict, List, Any
from dataclasses import dataclass
import pandas as pd
import matplotlib.pyplot as plt
import requests

class Region(str, Enum):
    LULEA = "SE1"
    SUNDSVALL = "SE2"
    GOTEBORG = "SE3"
    MALMO = "SE4"

@dataclass
class Labels:
    """Centralized configuration for labels and units"""
    unit: str = "öre/kWh"
    price_column: str = f"Price ({unit})"
    avg_price_column: str = f"Average Price ({unit})"
    min_price_column: str = f"Min Price ({unit})"
    max_price_column: str = f"Max Price ({unit})"
    delivery_start: str = "Delivery Start"
    delivery_end: str = "Delivery End"
    block_name: str = "Block Name"
    
    # Plot titles and labels
    hourly_title: str = "Hourly Day Ahead Prices for {area}"
    block_title: str = "Average Prices for Blocks"
    time_label: str = "Delivery Start Time"
    price_label: str = f"Price ({unit})"
    
    # Legend labels
    hourly_legend: str = "Hourly Prices"
    block_avg_legend: str = "Average Price"
    block_range_legend: str = "Min-Max Range"
    
    # Output message
    cheapest_block_msg: str = "The cheapest block for region {area} is '{block}' with an average price of {price:.2f} {unit}"

@dataclass
class PriceData:
    """Container for processed price data"""
    hourly_df: pd.DataFrame
    block_df: pd.DataFrame
    cheapest_block: Dict[str, float]

def process_hourly_data(data: Dict[str, Any], chosen_area: str, labels: Labels) -> pd.DataFrame:
    """
    Process hourly electricity price entries.
    
    Args:
        data: Raw data dictionary containing multiAreaEntries
        chosen_area: Selected area code for price analysis
        labels: Label configuration object
    
    Returns:
        DataFrame with processed hourly price data
    """
    try:
        hourly_entries = data['multiAreaEntries']
        hourly_data = [
            {
                labels.delivery_start: entry['deliveryStart'],
                labels.delivery_end: entry['deliveryEnd'],
                labels.price_column: entry['entryPerArea'][chosen_area] / 10
            }
            for entry in hourly_entries
        ]
        
        df = pd.DataFrame(hourly_data)
        df[labels.delivery_start] = pd.to_datetime(df[labels.delivery_start])
        df[labels.delivery_end] = pd.to_datetime(df[labels.delivery_end])
        return df
    
    except KeyError as e:
        raise ValueError(f"Missing required data field: {str(e)}")
    except Exception as e:
        raise RuntimeError(f"Error processing hourly data: {str(e)}")

def process_block_data(data: Dict[str, Any], chosen_area: str, labels: Labels) -> pd.DataFrame:
    """
    Process block price aggregates.
    
    Args:
        data: Raw data dictionary containing blockPriceAggregates
        chosen_area: Selected area code for price analysis
        labels: Label configuration object
    
    Returns:
        DataFrame with processed block price data
    """
    try:
        block_aggregates = data['blockPriceAggregates']
        block_data = [
            {
                labels.block_name: block['blockName'],
                labels.avg_price_column: block['averagePricePerArea'][chosen_area]['average'] / 10,
                labels.min_price_column: block['averagePricePerArea'][chosen_area]['min'] / 10,
                labels.max_price_column: block['averagePricePerArea'][chosen_area]['max'] / 10
            }
            for block in block_aggregates
        ]
        return pd.DataFrame(block_data)
    
    except KeyError as e:
        raise ValueError(f"Missing required data field: {str(e)}")
    except Exception as e:
        raise RuntimeError(f"Error processing block data: {str(e)}")

def find_cheapest_block(block_df: pd.DataFrame, labels: Labels) -> Dict[str, Any]:
    """
    Identify the block with lowest average price.
    
    Args:
        block_df: DataFrame containing block price data
        labels: Label configuration object
    
    Returns:
        Dictionary containing cheapest block information
    """
    return block_df.loc[block_df[labels.avg_price_column].idxmin()].to_dict()

def plot_hourly_prices(hourly_df: pd.DataFrame, chosen_area: str, labels: Labels) -> None:
    """
    Create visualization of hourly prices.
    
    Args:
        hourly_df: DataFrame containing hourly price data
        chosen_area: Selected area code for labeling
        labels: Label configuration object
    """
    plt.figure(figsize=(14, 7))
    plt.plot(hourly_df[labels.delivery_start], hourly_df[labels.price_column], 
             marker='o', label=labels.hourly_legend)
    plt.title(labels.hourly_title.format(area=chosen_area))
    plt.xlabel(labels.time_label)
    plt.ylabel(labels.price_label)
    plt.xticks(rotation=45)
    plt.grid()
    plt.legend()
    plt.tight_layout()
    plt.show()

def plot_block_prices(block_df: pd.DataFrame, labels: Labels) -> None:
    """
    Create visualization of block price averages with error bars.
    
    Args:
        block_df: DataFrame containing block price data
        labels: Label configuration object
    """
    plt.figure(figsize=(10, 5))
    plt.bar(block_df[labels.block_name], block_df[labels.avg_price_column], 
            color='skyblue', label=labels.block_avg_legend)
    
    plt.errorbar(
        block_df[labels.block_name],
        block_df[labels.avg_price_column],
        yerr=[
            block_df[labels.avg_price_column] - block_df[labels.min_price_column],
            block_df[labels.max_price_column] - block_df[labels.avg_price_column]
        ],
        fmt='o',
        color='black',
        label=labels.block_range_legend
    )
    
    plt.title(labels.block_title)
    plt.ylabel(labels.price_label)
    plt.grid(axis='y')
    plt.legend()
    plt.tight_layout()
    plt.show()

def analyze_prices(data: Dict[str, Any], chosen_area: str, labels: Labels) -> PriceData:
    """
    Main function to analyze electricity prices.
    
    Args:
        data: Raw data dictionary containing price information
        chosen_area: Selected area code for analysis
        labels: Label configuration object
    
    Returns:
        PriceData object containing processed data and analysis results
    """
    hourly_df = process_hourly_data(data, chosen_area, labels)
    block_df = process_block_data(data, chosen_area, labels)
    cheapest_block = find_cheapest_block(block_df, labels)
    
    return PriceData(
        hourly_df=hourly_df,
        block_df=block_df,
        cheapest_block=cheapest_block
    )

def process(data: Dict[str, Any], chosen_area: str) -> None:
    """
    Main execution function.
    
    Args:
        data: Raw data dictionary containing price information
        chosen_area: Selected area code for analysis
    """
    try:
        # Initialize labels with desired unit
        labels = Labels()
        
        # Process data and create visualizations
        price_data = analyze_prices(data, chosen_area.value, labels)
        plot_hourly_prices(price_data.hourly_df, chosen_area.name, labels)
        plot_block_prices(price_data.block_df, labels)
        
        # Print analysis results
        cheapest = price_data.cheapest_block
        print(labels.cheapest_block_msg.format(
            area=chosen_area.name,
            block=cheapest[labels.block_name],
            price=cheapest[labels.avg_price_column],
            unit=labels.unit
        ))
        
    except Exception as e:
        print(f"Error analyzing prices: {str(e)}")

def fetch(regions: list[Region]):
    selected_areas = ",".join([region.value for region in regions])
    tomorrow = (datetime.now() + timedelta(days=1)).strftime('%Y-%m-%d')
    url = f'https://dataportal-api.nordpoolgroup.com/api/DayAheadPrices?date={tomorrow}&market=DayAhead&deliveryArea={selected_areas}&currency=SEK'
    response = requests.get(url)
    return response.json()
```

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
tags: [remove-input]
---
regions = list(Region)
data = fetch(regions)
```

+++ {"editable": true, "slideshow": {"slide_type": ""}}

## Malmö

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
tags: [remove-input]
---
process(data, Region.MALMO)
```

+++ {"editable": true, "slideshow": {"slide_type": ""}}

## Stockholm / Göteborg

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
tags: [remove-input]
---
process(data, Region.GOTEBORG)
```

+++ {"editable": true, "slideshow": {"slide_type": ""}}

## Sundsvall

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
tags: [remove-input]
---
process(data, Region.SUNDSVALL)
```

+++ {"editable": true, "slideshow": {"slide_type": ""}}

## Luleå

```{code-cell} ipython3
---
editable: true
slideshow:
  slide_type: ''
tags: [remove-input]
---
process(data, Region.LULEA)
```

---

See the [README](../README.md) on why, what and how.
