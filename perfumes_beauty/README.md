# Perfumes & Beauty Category Scraper

Automated scraper for the Perfumes & Beauty category on **sheeel.com** with **concurrent subcategory scraping**.

## Category Details

- **URL:** https://www.sheeel.com/ar/perfumes-beauty.html
- **Category:** perfumes_beauty
- **Type:** Hierarchical (with subcategories)
- **S3 Folder:** `perfumes_beauty/`
- **Optimization:** Async scraping with semaphore (3 subcategories concurrently)

## Features

✅ **Concurrent Subcategory Scraping**
- **Performance:** ~2.5x faster than sequential scraping
- **Controlled:** Semaphore limits to 3 concurrent subcategories (configurable)
- **Stable:** Prevents server overload while maximizing speed

✅ **Incremental Image Upload**
- Images uploaded to S3 immediately after download
- ~30% faster than batch upload
- Lower risk with incremental success

✅ **Subcategory Support**
- Automatically extracts all subcategories from the main page
- Scrapes each subcategory independently with full pagination
- Each subcategory saved as a separate Excel sheet

✅ **Multi-Sheet Excel Output**
- One sheet per subcategory
- Additional "ALL_PRODUCTS" sheet with combined data
- Products tagged with `subcategory` field

✅ **Complete Data Extraction**
- Product ID, name, SKU, prices, availability
- Multiple product images (stored as arrays)
- Features & specifications
- Box contents, warranty info
- Discount badges, deal timers

✅ **AWS S3 Integration**
- Date partitioning: `year=YYYY/month=MM/day=DD/`
- Organized storage: separate folders for images and Excel files
- All images uploaded with indexed naming

✅ **Excel Safety**
- Automatic removal of illegal control characters
- Prevents `IllegalCharacterError` from Arabic text

## Data Structure

### S3 Path Structure

```
s3://{bucket}/sheeel_data/
    └── year=2026/
        └── month=04/
            └── day=15/
                └── perfumes_beauty/
                    ├── images/
                    │   ├── 12345_0.jpg
                    │   ├── 12345_1.jpg
                    │   └── ...
                    └── excel-files/
                        └── perfumes_beauty_20260415_120000.xlsx
```

### Excel Structure

**Multiple sheets:**
- One sheet per subcategory (e.g., perfumes, skincare, makeup, haircare, etc.)
- `ALL_PRODUCTS` sheet with combined data

**Columns (per product):**
- `product_id`, `name`, `sku`
- `old_price`, `special_price`
- `availability`, `times_bought`
- `description`
- `image_urls` (array - not flattened)
- `s3_image_paths` (array - not flattened)
- `features_specs` (array) + `feature_spec_0`, `feature_spec_1`, ... (flattened)
- `box_contents`, `warranty` (single values)
- `deal_time_left`, `discount_badge`
- `subcategory` (name of the subcategory)
- `url`, `scraped_at`

## Local Testing

```bash
cd RogueBit/perfumes_beauty

# Install dependencies
pip install -r requirements.txt
playwright install chromium

# Set environment variables
export AWS_ACCESS_KEY_ID="your_key"
export AWS_SECRET_ACCESS_KEY="your_secret"
export S3_BUCKET_NAME="your_bucket"

# Optional: Configure concurrency (default: 3)
export MAX_CONCURRENT_SUBCATEGORIES=3

# Run scraper
python scraper.py
```

## Performance Configuration

Control how many subcategories scrape simultaneously:

```bash
# Conservative (stable, slower)
export MAX_CONCURRENT_SUBCATEGORIES=2

# Default (balanced)
export MAX_CONCURRENT_SUBCATEGORIES=3

# Aggressive (faster, more memory)
export MAX_CONCURRENT_SUBCATEGORIES=5
```

**Recommendation:** Start with 3, reduce to 2 if you see timeouts or rate limiting.

## Automation

Runs automatically via GitHub Actions:
- **Schedule:** Every 2 days at 6:00 AM UTC
- **Manual Trigger:** Actions → RogueBit Scrapers → Run workflow

## Technical Details

**Base Class:** `PerfumesBeautyScraper`

**Key Methods:**
- `get_subcategories()` - Extract all subcategory links (async)
- `scrape_subcategory()` - Scrape one subcategory with pagination (async, semaphore-controlled)
- `scrape_all_subcategories()` - Main loop for concurrent subcategory scraping (async)
- `has_next_page()` - Check for Next button (reliable pagination, async)
- `download_all_images()` - Download all product images with incremental S3 upload (sync)
- `save_to_excel()` - Generate multi-sheet Excel file with character sanitization (sync)
- `upload_results_to_s3()` - Upload Excel to S3 (sync)

**Pagination Strategy:**
- Uses Next button detection (`.pages-item-next a.next`)
- No dependency on visible page count
- Works for unlimited pages

**Concurrency Strategy:**
1. Launch browser with single context
2. Extract all subcategory links from main page
3. Create semaphore with limit (default: 3)
4. Create tasks for all subcategories
5. Execute tasks concurrently with `asyncio.gather()`
6. Semaphore ensures only N subcategories scrape simultaneously
7. Results merged into multi-sheet Excel
8. ~2.5x faster than sequential scraping

## Selector Reference

| Element | Selector | Purpose |
|---------|----------|---------|
| Subcategory links | `.subcategory-link` | Extract subcategories |
| Product cards | `[id^="product-item-info_"] > a` | Product detail URLs |
| Next page | `.pages-item-next a.next` | Pagination |
| Product details | `#maincontent .product-info-main` | Main container |
| Images | `.product-gallery-image` | All product images |
| Features | `#more-info .attribute-info.label` | Features sections |

## Monitoring

Check S3 for latest data:
```bash
# List all data for today
aws s3 ls s3://{bucket}/sheeel_data/year=2026/month=04/day=15/perfumes_beauty/ --recursive

# Download Excel file
aws s3 cp s3://{bucket}/sheeel_data/year=2026/month=04/day=15/perfumes_beauty/excel-files/perfumes_beauty_*.xlsx .
```

## Notes

- **Concurrent Scraping:** ~2.5x faster than sequential (uses asyncio + semaphore)
- **Incremental Upload:** Images uploaded to S3 as downloaded (~30% faster)
- **Multi-sheet Excel:** Creates one sheet per subcategory
- **Arrays not flattened:** `image_urls` and `s3_image_paths` stored as JSON arrays
- **Features flattened:** `features_specs` array + individual `feature_spec_N` columns
- **Single values:** `box_contents` and `warranty` are single string values
- **Performance Guide:** See [CONCURRENT_OPTIMIZATION.md](../../CONCURRENT_OPTIMIZATION.md) for detailed information
- **Excel Safety:** Illegal control characters automatically removed before writing to Excel

---

**Last Updated:** April 15, 2026
