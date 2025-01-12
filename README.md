---
kernelspec: 
  name: python3
  display_name: Python 3
exports:
  - md
---

# Sweden's Electricity Prices

[Nordpoolgroup](https://nordpoolgroup.com) is the company responsible for the European power market. Each day, it publishes the prices for the next day around 13:00 (UTC+1).

By being aware of the pricing, you can schedule electricity-intensive tasks during time blocks that are more favorable for your wallet.

## Potential Benefits

If you integrate this information with your automation platform, you can automatically reschedule the following tasks to more economical times:

- Laundry
- Charging your electric vehicle
- Electrical heating
- Other electricity-intensive activities

## How It Works

This script has a `cron` schedule that triggers the generation of this notebook at 14:00.

When triggered, it:

1. Compiles a JupyterLab notebook into this HTML page.

## Notes

The Nordpoolgroup API is not openly available, so no historical data is kept in this repository. I have reached out to inquire whether I am allowed to use it in this manner.
