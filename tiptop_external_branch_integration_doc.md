# External Branch Partner API Documentation

## Overview

This document describes the API specification that partner development teams must implement to enable synchronization of their product catalog with the TipTop Market API system.

## API Requirements

### Endpoint Structure

Partners must provide a single endpoint that returns paginated product data:

```
GET {base_url}?page={page_number}
```

**Parameters:**
- `page`: Zero-based page number (starts from 0)

**Headers:**
- `TX-Api-Key`: API key for authentication (if required)

### Response Format

The endpoint must return a JSON response with the following structure:

```json
{
  "totalItemsCount": 1500,
  "items": [
    {
      "barcode": "1234567890123",
      "sku": "PROD-001",
      "price": 29.99,
      "quantity": 100,
      "translation": {
        "en": {
          "name": "PlayStation 5 Controller",
          "description": "Wireless controller for PS5"
        },
        "ar": {
          "name": "يد تحكم بلايستيشن 5",
          "description": "يد تحكم لاسلكية لجهاز بلايستيشن 5"
        },
        "ckb": {
          "name": "دەسکی پلەی ستەیشن 5",
          "description": "دەسکی بێ وایەر بۆ پلەی ستەیشن 5"
        }
      },
      "images": {
        "main": "https://example.com/images/main.jpg",
        "other": [
          "https://example.com/images/side1.jpg",
          "https://example.com/images/side2.jpg"
        ]
      },
      "category": "Console -> PlayStation",
      "brand": "Sony"
    }
  ]
}
```

## Data Model Specification

### Root Response Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `totalItemsCount` | integer | **Yes** | Total number of items across all pages. This value must remain consistent across all page requests. |
| `items` | array | **Yes** | Array of product items for the current page. Can be empty for pages beyond the last item. |

### Item Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `barcode` | string | **Yes** | Unique product barcode/identifier. Items without barcodes will be skipped. |
| `sku` | string | No | Stock Keeping Unit identifier. |
| `price` | decimal | **Yes** | Unit price of the item. |
| `quantity` | integer | **Yes** | Available inventory quantity. Set to 0 if out of stock. |
| `translation` | object | **Yes** | Translations for product name and description in supported languages. |
| `images` | object | **Yes** | Product images including main image and additional gallery images. |
| `category` | string | **Yes** | Product category hierarchy (see Category Format below). |
| `brand` | string | No | Brand name of the product. |

### Translation Object

The translation object supports four languages: English (`en`), Arabic (`ar`), Central Kurdish (`ckb`), and Kurmanji (`kbd`).

**English (`en`) translation is mandatory.** Providing translations for all languages is highly recommended to serve all customers.

```json
{
  "en": {
    "name": "Product Name",
    "description": "Product description"
  },
  "ar": {
    "name": "اسم المنتج",
    "description": "وصف المنتج"
  },
  "ckb": {
    "name": "ناوی بەرهەم",
    "description": "وەسفی بەرهەم"
  },
  "kbd": {
    "name": "ناڤێ بەرهەمێ",
    "description": "وەسفێ بەرهەمێ"
  }
}
```

**Translation Detail Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | **Yes** | Product name in the specified language. |
| `description` | string | No | Product description in the specified language. |

### Images Object

```json
{
  "main": "https://example.com/main-image.jpg",
  "other": [
    "https://example.com/gallery-1.jpg",
    "https://example.com/gallery-2.jpg"
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `main` | string | Yes | URL to the main product image. |
| `other` | array of strings | No | URLs to additional product gallery images. Can be empty array. |

### Category Format

Categories can be specified in two formats:

**Single-level category:**
```
"Console"
```
This creates/uses only a category named "Console".

**Two-level category (Main and subcategory):**
```
"Console -> PlayStation"
```
This creates/uses:
- Main category: "Console"
- Subcategory: "PlayStation" (child of Console)

**Format Rules:**
- Use `->` as separator
- Maximum two levels are supported
- Category names will be trimmed automatically
- Invalid formats will be logged and skipped

**Examples:**
- ✅ `"Electronics"`
- ✅ `"Electronics -> Gaming"`
- ❌ `"Electronics -> Gaming -> PlayStation"` (3 levels not supported)

## Pagination Mechanism

### How Pagination Works

1. **Initial Request (Page 0):**
   - System fetches page 0
   - Captures `totalItemsCount` from response
   - Calculates total number of pages needed

2. **Subsequent Requests:**
   - System continues fetching pages sequentially (1, 2, 3, ...)
   - Stops when either:
     - Total processed items >= `totalItemsCount`
     - OR a page returns empty `items` array

3. **Page Size:**
   - Partners determine their own page size
   - Recommended: 50-200 items per page
   - Page size should remain consistent across all pages

### Example Pagination Flow

**Scenario:** 250 total items, 100 items per page

```
Request 1: GET /products?page=0
Response: { "totalItemsCount": 250, "items": [...100 items...] }

Request 2: GET /products?page=1
Response: { "totalItemsCount": 250, "items": [...100 items...] }

Request 3: GET /products?page=2
Response: { "totalItemsCount": 250, "items": [...50 items...] }

Request 4: GET /products?page=3
Response: { "totalItemsCount": 250, "items": [] }

// Sync complete: 250 items processed
```

## Important Notes

### Required Fields
- **`barcode`**: Absolutely required. Items without barcodes will be skipped.
- **`totalItemsCount`**: Must be accurate and consistent across all page requests.
- **`category`**: Required. Use single-level (e.g., "Electronics") or two-level (e.g., "Electronics->Gaming") format.
- **`translation.en`**: English translation is mandatory. Must include at least the product name.
- **`images.main`**: Main product image URL is required.

### Optional But Recommended
- **`brand`**: Providing brand information helps organize products and enables brand-based filtering.
- **`sku`**: Helpful for inventory management and cross-referencing.
- **`translation.ar/ckb/kbd`**: Additional language translations improve accessibility for customers.
- **`images.other`**: Additional product images improve customer experience and reduce returns.

### Data Consistency
- `totalItemsCount` must remain the same across all page requests in a single sync operation.
- If products are added/removed during sync, they will be caught in the next sync cycle.

### Performance Recommendations
- Implement caching if possible to improve response times
- Keep page size between 50-200 items for optimal performance
- Ensure your API can handle multiple sequential requests

### Error Handling
- Return appropriate HTTP status codes (200 for success, 4xx/5xx for errors)
- If authentication fails, return 401 Unauthorized
- If the endpoint is unavailable, return 503 Service Unavailable

### Authentication
If your API requires authentication:
- Accept the API key via the `TX-Api-Key` header
- Provide your API key to the TipTop team during integration setup

## Integration Setup

To connect your system with TipTop Market API:

1. **Initial Contact:**
   - Contact TipTop team to request integration
   - Receive your `TX-Api-Key`
   
2. **Implement the API endpoint** according to this specification

3. **Self-Service Testing:**
   - Use the partner validation endpoint to test your implementation
   - Fix any validation errors identified
   - Iterate until validation passes
   
4. **Provide to TipTop team:**
   - Base URL of your endpoint
   - Confirmation that validation passes
   
5. **Go Live:**
   - TipTop team will perform final verification
   - Schedule initial sync testing
   - Set up sync frequency (hourly, daily, etc.)
   - Monitor initial syncs for any issues

## Validation Tool

TipTop provides an automated validation tool that checks your API implementation before going live. The validation tool will:

- Fetch sample data from your endpoint (first page)
- Validate the response structure (totalItemsCount, items array)
- Check each item's required and optional fields
- **Report only problematic fields** - valid fields are not included in the response
- Provide detailed error messages for issues found
- Provide a summary of valid, invalid, and items with warnings

### Self-Service Validation Endpoint

Partners can test their API integration themselves using the validation endpoint:

**Endpoint:**
```
POST /api/partner/v1/external-branches/check
```

**Headers:**
```
TX-Api-Key: your-api-key
Content-Type: application/json
```

**Example Request:**
```bash
curl -X POST "https://api.tiptop.com/api/partner/v1/external-branches/check" \
  -H "TX-Api-Key: your-api-key-here" \
  -H "Content-Type: application/json"
```

**Response Codes:**
- `200 OK`: Validation completed (check response for detailed results)
- `401 Unauthorized`: Invalid or missing API key
- `404 Not Found`: External branch not found

**Benefits of Self-Service Validation:**
- Test your API integration anytime during development
- Get immediate feedback without contacting TipTop team
- Validate changes after updates to your API
- Identify and fix issues before going live

**Note:** The validation endpoint checks only the **first page (page 0)** of your API response. Ensure your first page is representative of your entire catalog's data quality.

### Troubleshooting Validation Issues

If you receive validation errors, here's how to troubleshoot:

1. **Check the `globalIssues` array:**
   - These are structural problems with your API response
   - Common issue: Missing or invalid `totalItemsCount`
   - Solution: Ensure your response structure matches the specification exactly

2. **Review `itemResults` for per-item issues:**
   - Each item shows `overallStatus`: `Valid`, `Warning`, or `Invalid`
   - **Items with `Valid` status have empty `fieldValidations` arrays** (no issues found)
   - **Items with issues have `fieldValidations` populated** with specific problems
   - Focus on items with `Invalid` status - these must be fixed before going live
   - Address items with `Warning` status to improve data quality

3. **Common fixes:**
   - **Missing barcode**: Ensure every item has a unique barcode
   - **Missing category**: Add category in format: `"Category"` or `"Parent->Child"`
   - **Missing English translation**: Add `translation.en.name` to all items
   - **Missing main image**: Add `images.main` URL to all items
   - **Invalid price/quantity**: Ensure price > 0 and quantity >= 0

4. **Test again:**
   - After fixing issues, run the validation endpoint again
   - Repeat until `isValid: true` with minimal warnings

### Validation Response Example

**Important:** The validation response **only includes fields that have issues**. If an item is valid, its `fieldValidations` array will be empty. This keeps the response concise and helps you focus on what needs to be fixed.

**Valid Item (No Issues):**
```json
{
  "isValid": true,
  "totalItemsReported": 250,
  "itemsChecked": 50,
  "globalIssues": [],
  "itemResults": [
    {
      "index": 1,
      "barcode": "1234567890123",
      "sku": "PS5-CTRL-001",
      "overallStatus": "Valid",
      "fieldValidations": []
    }
  ],
  "summary": {
    "totalItemsCount": 50,
    "validItems": 48,
    "invalidItems": 0,
    "itemsWithWarnings": 2,
    "issueTypeCounts": {
      "SKU (Warning)": 1,
      "Brand (Warning)": 1
    }
  }
}
```

**Item with Issues:**
```json
{
  "isValid": false,
  "totalItemsReported": 250,
  "itemsChecked": 50,
  "globalIssues": [],
  "itemResults": [
    {
      "index": 1,
      "barcode": "1234567890123",
      "sku": "PS5-CTRL-001",
      "overallStatus": "Valid",
      "fieldValidations": []
    },
    {
      "index": 2,
      "barcode": null,
      "sku": "HDMI-001",
      "overallStatus": "Invalid",
      "fieldValidations": [
        {
          "fieldName": "Barcode",
          "status": "Invalid",
          "message": "❌ Invalid: Barcode is required but missing or empty",
          "value": "null"
        },
        {
          "fieldName": "Translations",
          "status": "Invalid",
          "message": "❌ Invalid: English (en) translation is required",
          "value": "missing en"
        }
      ]
    },
    {
      "index": 3,
      "barcode": "9876543210987",
      "sku": null,
      "overallStatus": "Warning",
      "fieldValidations": [
        {
          "fieldName": "SKU",
          "status": "Warning",
          "message": "⚠ Warning: SKU is empty (optional but recommended)",
          "value": "null"
        },
        {
          "fieldName": "Brand",
          "status": "Warning",
          "message": "⚠ Warning: Brand is empty (optional but recommended)",
          "value": "null"
        }
      ]
    }
  ],
  "summary": {
    "totalItemsCount": 50,
    "validItems": 47,
    "invalidItems": 1,
    "itemsWithWarnings": 2,
    "issueTypeCounts": {
      "Barcode (Invalid)": 1,
      "Translations (Invalid)": 1,
      "SKU (Warning)": 2,
      "Brand (Warning)": 2
    }
  }
}
```

### Common Validation Issues

**Invalid Issues (Must Fix):**
- ❌ Missing or empty barcode
- ❌ Missing or empty category
- ❌ Category with more than 2 levels
- ❌ Price is 0 or negative
- ❌ Quantity is negative
- ❌ Missing English (en) translation - English is mandatory
- ❌ Missing main image

**Warning Issues (Recommended to Fix):**
- ⚠️ Missing SKU (optional but helpful for inventory management)
- ⚠️ Missing brand (optional but helps with product organization)
- ⚠️ Missing translations for other languages (ar, ckb, kbd are recommended but optional)

## Development Workflow

### Recommended Steps for Integration

1. **Initial Setup:**
   - Receive your `TX-Api-Key` from TipTop team
   - Implement your API endpoint according to the specification

2. **Development & Testing:**
   - Use the self-service validation endpoint to test your implementation
   - Run validation checks frequently during development
   - Fix any invalid issues (❌) before proceeding
   - Address warning issues (⚠️) for better data quality

3. **Pre-Production Testing:**
   - Ensure all validation checks pass with no invalid issues
   - Test with your full product catalog (or representative sample)
   - Verify that `totalItemsCount` is accurate
   - Confirm all required fields are populated

4. **Go Live:**
   - Contact TipTop team when validation passes consistently
   - TipTop team will perform final verification
   - Schedule initial sync and monitoring period
   - Set up sync frequency (e.g., hourly, daily)

### Example Development Iteration

```bash
# Step 1: Run initial validation
curl -X POST "https://api.tiptop.com/api/partner/v1/external-branches/check" \
  -H "TX-Api-Key: your-api-key"

# Step 2: Review validation response
# - Check "isValid": should be true
# - Check "summary.invalidItems": should be 0
# - Look at items where "overallStatus" is "Invalid" or "Warning"
# - Only problematic fields will appear in "fieldValidations"

# Step 3: Fix issues in your API

# Step 4: Run validation again
curl -X POST "https://api.tiptop.com/api/partner/v1/external-branches/check" \
  -H "TX-Api-Key: your-api-key"

# Step 5: Repeat until isValid: true with zero invalid items
```

## Sync Behavior

### Product Creation
When a new product (barcode not found in system) is synced:
- Creates new item with all provided data
- Creates inventory record if quantity > 0
- Creates/assigns categories as specified
- Uploads images from provided URLs

### Product Updates
When an existing product (barcode found) is synced:
- Updates price, translations, and category
- Updates inventory quantity
- Preserves item ID and historical data

### Categories
- Categories are automatically created if they don't exist
- Main and subcategories are cached during sync
- Same categories are reused across multiple products

### Brands (Vendors)
- When a product has a brand, the system will:
  - Search for an existing brand with that brand name
  - If found, link the item to that brand
  - If not found, create a new brand with the brand name
- Brands are cached during sync for performance
- Brand names are case-insensitive and trimmed automatically

## Example Implementations

### Minimal Valid Response (Page 0)

```json
{
  "totalItemsCount": 1,
  "items": [
    {
      "barcode": "1234567890123",
      "sku": null,
      "price": 29.99,
      "quantity": 10,
      "translation": {
        "en": {
          "name": "Sample Product",
          "description": null
        },
        "ar": null,
        "ckb": null,
        "kbd": null
      },
      "images": {
        "main": null,
        "other": []
      },
      "category": "General",
      "brand": null
    }
  ]
}
```

### Full-Featured Response (Page 0)

```json
{
  "totalItemsCount": 2,
  "items": [
    {
      "barcode": "1234567890123",
      "sku": "PS5-CTRL-001",
      "price": 65000,
      "quantity": 50,
      "translation": {
        "en": {
          "name": "DualSense Wireless Controller",
          "description": "Experience haptic feedback and dynamic trigger effects with the DualSense wireless controller."
        },
        "ar": {
          "name": "يد تحكم لاسلكية DualSense",
          "description": "اختبر ردود الفعل اللمسية وتأثيرات الزناد الديناميكية مع وحدة التحكم اللاسلكية DualSense."
        },
        "ckb": {
          "name": "دەسکی بێ وایەری DualSense",
          "description": "ئەزموونی فیدباکی هاپتیک و کاریگەرییەکانی تریگەری داینامیکی بکە لەگەڵ دەسکی بێ وایەری DualSense."
        },
        "kbd": {
          "name": "دەستەی بێ تەلی DualSense",
          "description": "تەجروبەیا فیدباکا هاپتیک و کاریگەریێن تریگەرێ داینامیک بکە لەگەل دەستەی بێ تەلی DualSense."
        }
      },
      "images": {
        "main": "https://cdn.example.com/products/ps5-controller-main.jpg",
        "other": [
          "https://cdn.example.com/products/ps5-controller-side.jpg",
          "https://cdn.example.com/products/ps5-controller-back.jpg",
          "https://cdn.example.com/products/ps5-controller-charging.jpg"
        ]
      },
      "category": "Console->PlayStation",
      "brand": "Sony"
    },
    {
      "barcode": "9876543210987",
      "sku": "HDMI-CBL-2M",
      "price": 4500,
      "quantity": 200,
      "translation": {
        "en": {
          "name": "HDMI Cable 2m",
          "description": "High-speed HDMI cable, 2 meters length, supports 4K resolution."
        },
        "ar": {
          "name": "كابل HDMI 2 متر",
          "description": "كابل HDMI عالي السرعة، طول 2 متر، يدعم دقة 4K."
        },
        "ckb": {
          "name": "کێبڵی HDMI 2 مەتر",
          "description": "کێبڵی HDMI خێرا، درێژی 2 مەتر، پشتگیری وردبینی 4K دەکات."
        },
        "kbd": {
          "name": "کابلویا HDMI 2 مەتر",
          "description": "کابلویا HDMI خێرا، درێژی 2 مەتر، پشتگیریا وردبینییا 4K دکەت."
        }
      },
      "images": {
        "main": "https://cdn.example.com/products/hdmi-cable.jpg",
        "other": []
      },
      "category": "Accessories",
      "brand": "Generic"
    }
  ]
}
```

## Support

For questions or issues during integration, please contact the TipTop development team:
- Technical Support: [support email]
- Integration Issues: [integration email]

## Version History

- **v1.0-beta.1** (2025-11-11): Initial API specification

