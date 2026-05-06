# Highbase Review Website — Documentation
**Version:** 2.0  
**Last Updated:** May 2026  
**File:** `index.html`  
**URL:** `https://oozz26.github.io/highbase-supplier/`

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [URL Parameters](#url-parameters)
4. [Configuration](#configuration)
5. [Data Structure](#data-structure)
6. [Page Sections](#page-sections)
7. [Core Functions Reference](#core-functions-reference)
8. [User Flows (Statement of Procedure)](#user-flows)
9. [Error Handling](#error-handling)
10. [Troubleshooting](#troubleshooting)

---

## 1. Overview

The Highbase Review Website is a **supplier-facing confirmation portal**. Suppliers receive a unique link via WhatsApp that opens this page. They review, edit, and confirm their submitted products — which then get written to the confirmed Google Sheet and stock database.

### What It Does
```
Supplier sends products via WhatsApp
        ↓
AI agent processes and stores in Pending Sheet
        ↓
Supplier receives review link
        ↓
Supplier opens this website
        ↓
Supplier reviews / edits / confirms each product
        ↓
Data written to Confirmed Sheet + Stock Database
```

### Tech Stack
| Component | Technology |
|-----------|------------|
| Hosting | GitHub Pages (static) |
| Language | Vanilla HTML + CSS + JavaScript |
| Fonts | Inter + Manrope (Google Fonts) |
| Data Source | n8n Webhooks |
| Storage | Google Sheets + Supabase |

---

## 2. Architecture

```
index.html
│
├── <style>          → All CSS (no external stylesheet)
│    ├── :root       → Design tokens (colors, fonts, spacing)
│    ├── Header      → Sticky top bar
│    ├── Progress    → Confirmation progress card
│    ├── Cards       → Product card grid
│    ├── Fields      → Input and select styling
│    ├── Buttons     → Primary, delete, mic, saved
│    ├── Recording   → Voice recording banner
│    ├── AI Hint     → Transcript display
│    ├── Bottom Bar  → Fixed confirm-all bar
│    ├── Screens     → Loading, empty, error, success
│    └── Toast       → Notification system
│
├── <body>
│    ├── <header>         → Logo + product count badge
│    ├── #toast           → Notification overlay
│    ├── #loadingScreen   → Shown while fetching data
│    ├── #errorScreen     → Shown on fetch failure
│    ├── #emptyScreen     → Shown when no products
│    ├── #successScreen   → Shown when all confirmed
│    ├── #mainContent     → Progress bar + card grid
│    └── .bottom-bar      → Fixed "Confirm All" button
│
└── <script>
     ├── CONFIG           → Webhook URLs
     ├── VALIDATION LISTS → Units, currencies, countries
     ├── STATE            → products[], confirmed, chatId, branchId
     ├── Boot             → DOMContentLoaded handler
     ├── loadProducts()   → Fetch + map data
     ├── makeCard()       → Render individual product card
     ├── confirmProduct() → Submit single product
     ├── confirmAll()     → Submit all remaining products
     ├── handleImageChange() → Compress + preview image
     ├── deleteProduct()  → Remove product from list
     ├── Voice Functions  → Recording + AI transcription
     └── Utils            → show/hide/toast/esc
```

---

## 3. URL Parameters

The page requires two URL parameters to function:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `chat_id` | Supplier's WhatsApp number | `97339392826` |
| `branch_id` | Branch identifier from Supabase | `55` |

**Full URL Example:**
```
https://oozz26.github.io/highbase-supplier/?chat_id=97339392826&branch_id=55
```

**How parameters are read:**
```javascript
const params = new URLSearchParams(window.location.search);
chatId   = params.get('chat_id');    // → "97339392826"
branchId = params.get('branch_id'); // → "55"
```

**What happens if missing:**
- Missing `chat_id` → Error screen: "No supplier ID found"
- Missing `branch_id` → Error screen: "No supplier Branch ID found"

---

## 4. Configuration

Located at the top of the `<script>` block:

```javascript
// ── CONFIG ────────────────────────────────────────────────────────────────
const WEBHOOK_GET   = 'https://n8n.srv1587311.hstgr.cloud/webhook/review-data';
const WEBHOOK_POST  = 'https://n8n.srv1587311.hstgr.cloud/webhook/review-confirm';
const WEBHOOK_VOICE = 'https://n8n.srv1587311.hstgr.cloud/webhook/voice-update';
```

| Constant | Purpose | Method |
|----------|---------|--------|
| `WEBHOOK_GET` | Fetch supplier's pending products | GET |
| `WEBHOOK_POST` | Submit a confirmed product | POST |
| `WEBHOOK_VOICE` | Send voice recording for AI transcription | POST |

**To update webhook URLs:** Change the values of these three constants.

---

## 5. Data Structure

### 5.1 Incoming Data (from WEBHOOK_GET)

The webhook returns an array of product objects. The page handles **two formats**:

**Old Format (combined size):**
```json
{
  "Product Name": "Grapes",
  "size": "10 kg",
  "Color": "Black",
  "Origin": "India",
  "price": "4.000",
  "image url": "https://drive.google.com/...",
  "Category": "Fruits",
  "Pack": 1,
  "Box": 1,
  "row_number": 5
}
```

**New Format (split size + currency):**
```json
{
  "Product Name": "Grapes",
  "size": "10",
  "size_unit": "kg",
  "Color": "Black",
  "Origin": "India",
  "price": "4.000",
  "price_unit": "BD",
  "image url": "https://drive.google.com/...",
  "Category": "Fruits",
  "Pack": 1,
  "Box": 1,
  "row_number": 5
}
```

> **Note:** The page auto-detects the format and handles both.

### 5.2 Internal Product Object

After mapping, each product is stored as:

```javascript
{
  // Identity
  product_name: "Grapes",
  color:        "Black",
  type:         "",
  
  // Size (split)
  size:         "10",        // numeric only
  size_unit:    "kg",        // unit only
  
  // Price (split)
  price:        "4.000",     // numeric only
  price_unit:   "BD",        // currency code
  
  // Origin
  Origin:       "India",
  
  // Metadata
  product_group: "Grapes",
  pack_format:   "",
  variant_name:  "Black Grapes 10kg - India",
  image:         "https://drive.google.com/...",
  Category:      "Fruits",
  Pack:          1,
  Box:           1,
  
  // System fields
  brand_id:     "",
  Brand:        "",
  row_number:   5,           // used to delete from pending sheet
  
  // Runtime flags
  _newImage:    false        // true when user uploads new image
}
```

### 5.3 Outgoing Data (to WEBHOOK_POST)

When a product is confirmed, this payload is sent:

```javascript
{
  // Product data
  brand_id:      "",
  Brand:         "",
  Barcode:       "",
  product_group: "Grapes",
  Product_Name:  "Grapes",
  type:          "",
  Color:         "Black",
  size:          "10",           // numeric only
  size_unit:     "kg",           // unit only
  Origin:        "India",
  Pack_Formate:  "",
  variant_name:  "Black Grapes 10kg - India",
  image_url:     "https://...",
  description:   "",
  Category:      "Fruits",
  Pack:          1,
  Box:           1,
  price:         "4.000",        // numeric only
  price_unit:    "BD",           // currency code
  
  // Routing data
  chat_id:       "97339392826",
  branch_id:     "55",
  row_number:    5,
  confirmed_at:  "2026-05-06T10:30:00.000Z"
}
```

---

## 6. Page Sections

### 6.1 Header
- **Sticky** at top of page (stays visible when scrolling)
- Contains: **HIGHBASE** logo + tagline + product count badge
- Product count badge updates after data loads

### 6.2 Loading Screen (`#loadingScreen`)
- Shown immediately on page load
- Hidden after data fetch completes (success or error)
- Shows spinning loader + "Loading your products..."

### 6.3 Error Screen (`#errorScreen`)
- Shown when: fetch fails, or URL parameters missing
- Message is dynamic — set by `showError(msg)`

### 6.4 Empty Screen (`#emptyScreen`)
- Shown when: fetch succeeds but returns 0 products
- Message: "No products today"

### 6.5 Progress Bar (inside `#mainContent`)
- Shows `confirmed / total` count
- Progress bar fills as products are confirmed
- Updates after each confirmation

### 6.6 Product Cards (`.product-card`)

Each card contains:

```
┌─────────────────────────────────────┐
│ [Image]  │ Product Name             │
│          │ [Color] [Size] [Price]   │
│          │ [Origin]                 │
├─────────────────────────────────────┤
│ Product Name  [text input]          │
│ Color         [text input]          │
│ Size          [number input]        │
│ Unit          [dropdown]            │
│ Price         [number input]        │
│ Currency      [dropdown]            │
│ Origin Country [dropdown]           │
├─────────────────────────────────────┤
│ [🎤 Mic]  [✓ Confirm product]       │
└─────────────────────────────────────┘
```

**Card States:**
| State | Visual |
|-------|--------|
| Default | White border |
| Hover | Blue soft border + shadow |
| Confirmed | Green border + green background gradient |

### 6.7 Bottom Bar (`.bottom-bar`)
- Fixed at bottom of screen
- Shows: confirmed count + "Confirm All" button
- "Confirm All" button disabled when all confirmed

### 6.8 Success Screen (`#successScreen`)
- Shown when all products confirmed
- Lists all confirmed products with their details
- Replaces main content entirely

### 6.9 Toast Notifications (`#toast`)
- Small popup at bottom center
- Auto-hides after 3 seconds
- Three types: `success` (green), `error` (red), `info` (dark)

---

## 7. Core Functions Reference

### 7.1 `loadProducts()`
**Purpose:** Fetch supplier's pending products from n8n webhook  
**Trigger:** Called on page load after URL validation  
**Flow:**
```
1. GET request to WEBHOOK_GET?chat_id={chatId}
2. Parse JSON response
3. Filter out empty rows
4. Map each row to internal product object
   - Auto-detect old/new size format
   - Default price_unit to "BD" if missing
   - Clean newlines from all string fields
5. Update product count badge
6. Show mainContent + bottomBar
7. Call renderCards() + updateProgress()
```

---

### 7.2 `makeCard(p, i)`
**Purpose:** Build HTML for a single product card  
**Parameters:**
- `p` → product object
- `i` → index in products array

**Key behaviors:**
- Converts Google Drive URLs to thumbnail format via `fixImageUrl()`
- Pre-selects current size_unit in dropdown
- Pre-selects current price_unit in currency dropdown
- Pre-selects current Origin in country dropdown
- Attaches inline event handlers for all inputs

---

### 7.3 `confirmProduct(i)`
**Purpose:** Validate and submit a single product  
**Trigger:** User clicks "Confirm product" button  
**Validation checks (in order):**
1. No invalid inputs (red border inputs)
2. Product name not empty
3. Size not empty
4. Unit selected
5. Price not empty
6. Currency selected
7. Origin selected
8. No active voice recording

**On success:**
- Adds card index to `confirmed` Set
- Adds `.confirmed` class to card (turns card green)
- Calls `updateProgress()`
- Shows success toast
- If all confirmed → calls `showAllConfirmed()` after 800ms

**Payload sent:** See Section 5.3

---

### 7.4 `confirmAll()`
**Purpose:** Submit all unconfirmed products sequentially  
**Trigger:** User clicks "Confirm all" button in bottom bar  
**Behavior:**
- Loops through all unconfirmed product indices
- Calls `confirmProduct(i)` for each with 300ms delay between
- Re-enables button when done

---

### 7.5 `handleImageChange(event, i)`
**Purpose:** Handle image upload with auto-compression  
**Trigger:** User clicks "Change" on image, selects file  
**Process:**
```
1. Check product not already confirmed
2. Validate file is an image type
3. Reject files over 5MB
4. Show "Processing image..." toast
5. Read file as DataURL
6. Draw onto canvas at max 800×800px
7. Export as JPEG at 80% quality
8. Check compressed size < 700KB
9. Update card image preview
10. Store base64 in products[i].image
11. Set products[i]._newImage = true
12. Show "Image updated (XXkb)" toast
```

**Why compression?** Base64 encoded images sent via JSON trigger CORS preflight requests if the payload is too large. Compression keeps payload under ~500KB.

---

### 7.6 `deleteProduct(i)`
**Purpose:** Remove a product from the review list  
**Trigger:** User clicks ✕ button on card  
**Behavior:**
- Blocks deletion of already-confirmed products
- Removes from `products[]` array
- Shifts all `confirmed` indices down by 1
- Re-renders all cards
- Shows empty screen if no products remain

---

### 7.7 `updateSummary(i)`
**Purpose:** Update the summary chips on a card in real-time  
**Trigger:** Any field changes on the card  
**Chips shown:** Color, Size+Unit, Price+Currency, Origin

---

### 7.8 `fixImageUrl(url)`
**Purpose:** Convert Google Drive share URLs to thumbnail URLs  
**Handles two formats:**

| Input Format | Output |
|-------------|--------|
| `/file/d/{ID}/view` | `drive.google.com/thumbnail?id={ID}&sz=w400` |
| `?id={ID}` | `drive.google.com/thumbnail?id={ID}&sz=w400` |
| Any other URL | Returned as-is |

---

### 7.9 Voice Recording Functions

#### `toggleRecording(i)`
- If not recording → start
- If recording this card → stop
- If recording another card → show error

#### `startRecording(i)`
```
1. Request microphone permission
2. Detect best audio format (webm/opus > webm > mp4)
3. Start MediaRecorder
4. Show recording banner with timer
5. Auto-stop after 30 seconds
```

#### `stopRecording(i)`
```
1. Stop MediaRecorder
2. Hide recording banner
3. Show "processing" state on mic button
4. Trigger sendVoiceUpdate()
```

#### `sendVoiceUpdate(i, audioBlob)`
```
1. Convert audio to base64
2. POST to WEBHOOK_VOICE with:
   - audio_base64, audio_mime
   - chat_id, branch_id, row_number
   - current field values
3. Receive AI result
4. Call applyVoiceUpdates()
```

#### `applyVoiceUpdates(i, result)`
- Updates fields that AI extracted: product_name, color, size, size_unit, price, price_unit, Origin
- Highlights updated fields with blue pulse animation
- Shows transcript in AI hint banner

---

### 7.10 Utility Functions

| Function | Purpose |
|----------|---------|
| `show(id)` | Show element (block or flex based on element) |
| `hide(id)` | Hide element (`display: none`) |
| `showError(msg)` | Hide loader, show error screen with message |
| `esc(s)` | Escape HTML special characters (XSS protection) |
| `showToast(msg, type)` | Show notification (success/error/info) |
| `blobToBase64(blob)` | Convert audio Blob to base64 string |
| `validateNumber(input)` | Validate numeric input (digits + decimal) |
| `validatePrice(input)` | Validate price input (digits + decimal) |

---

## 8. User Flows (Statement of Procedure)

### SOP-01: Supplier Opens Review Link

**Actor:** Supplier  
**Trigger:** Receives WhatsApp message with link  
**Steps:**

```
Step 1: Supplier receives message:
        "Here's your review link:
        https://oozz26.github.io/highbase-supplier/?chat_id=97339392826&branch_id=55"

Step 2: Supplier taps the link
        → Page opens in browser

Step 3: Loading screen appears
        → Spinner shows "Loading your products..."

Step 4: Page fetches products from n8n
        → GET https://n8n.srv1587311.hstgr.cloud/webhook/review-data?chat_id=97339392826

Step 5a: If products found → Product cards appear
Step 5b: If no products → Empty screen: "No products today"
Step 5c: If fetch fails → Error screen with error message
```

---

### SOP-02: Supplier Reviews and Confirms a Product

**Actor:** Supplier  
**Trigger:** Product cards are visible  
**Steps:**

```
Step 1: Supplier sees product card with pre-filled data:
        - Product name (from AI extraction)
        - Color (if detected)
        - Size number + unit dropdown
        - Price + currency dropdown
        - Origin country dropdown
        - Product image (from Google Drive)

Step 2: Supplier reviews each field
        - If correct → proceed to Step 4
        - If incorrect → Step 3

Step 3: Supplier edits field
        - Types in text inputs
        - Selects from dropdowns
        - Changed fields turn blue (visual feedback)

Step 4: Supplier taps "Confirm product"
        - If any required field missing → Red toast shows missing fields
        - If all valid → Button shows loading spinner

Step 5: Data submitted to n8n webhook

Step 6: Card turns green with "Confirmed & saved" badge
        - Inputs become read-only
        - Confirm button replaced with "Saved" button
        - Progress bar updates

Step 7: If all products confirmed → Success screen appears
```

---

### SOP-03: Supplier Uses "Confirm All"

**Actor:** Supplier  
**Trigger:** Multiple products all look correct  
**Steps:**

```
Step 1: Supplier reviews all cards quickly

Step 2: Taps "Confirm all" in bottom bar

Step 3: System validates and submits each product sequentially
        (300ms delay between each submission)

Step 4: Each card turns green as it's confirmed

Step 5: Bottom bar button becomes disabled
        - Shows "Saving..." during process

Step 6: All confirmed → Success screen
```

---

### SOP-04: Supplier Updates Product Image

**Actor:** Supplier  
**Trigger:** Product image is wrong or missing  
**Steps:**

```
Step 1: Supplier taps "Change" button on image thumbnail

Step 2: Device file picker opens

Step 3: Supplier selects photo from camera or gallery

Step 4: System auto-compresses image:
        - Resizes to max 800×800px
        - Converts to JPEG at 80% quality
        - Validates final size < 700KB

Step 5a: If compression successful → Image updates in preview
          Toast: "Image updated (XXkb)"

Step 5b: If image too large after compression:
          Toast: "Image still too large. Try a smaller image."
          File selection cleared

Step 6: Supplier confirms product normally
        → New image sent as base64 in confirmation payload
```

---

### SOP-05: Supplier Uses Voice to Update Fields

**Actor:** Supplier  
**Trigger:** Wants to update fields without typing  
**Steps:**

```
Step 1: Supplier taps 🎤 mic button on product card
        → Recording banner appears with timer
        → Mic button turns red (recording indicator)

Step 2: Supplier speaks product details:
        Example: "Black Grapes, 10 kg, 4 BD, India"

Step 3: Recording auto-stops after 30 seconds
        OR supplier taps "Stop" button

Step 4: Mic button shows processing animation

Step 5: Audio sent to n8n → OpenAI Whisper → Field extraction

Step 6: Updated fields appear with blue highlight animation

Step 7: AI hint banner shows transcript:
        'AI heard: "Black Grapes, 10 kg, 4 BD, India"'

Step 8: Supplier reviews AI-extracted fields

Step 9: Supplier confirms product
```

---

### SOP-06: Supplier Deletes a Wrong Product

**Actor:** Supplier  
**Trigger:** Product was submitted by mistake  
**Steps:**

```
Step 1: Supplier taps ✕ (delete) button on card

Step 2: Product removed from view immediately

Step 3: Progress bar updates

Step 4: If last product deleted → Empty screen appears

Note: CANNOT delete already-confirmed products
       Toast shows: "Cannot delete confirmed product"
```

---

### SOP-07: Admin Sends Review Link (n8n Side)

**Actor:** n8n workflow  
**Trigger:** Supplier asks for review link  
**Steps:**

```
Step 1: Supplier sends message like "ممكن رابط المراجعة؟"

Step 2: Basic LLM Chain classifies as "CONFIRMATION"

Step 3: Switch routes to confirmation flow

Step 4: CHECK_BRANCH queries Supabase:
        GET /user_branches?phone_number=eq.{phone}

Step 5a: 1 branch found → Send direct link
         URL: https://oozz26.github.io/highbase-supplier/?chat_id={wa_id}&branch_id={branch_id}

Step 5b: Multiple branches found → Supplier selects branch via form
         → After selection → Send link with selected branch_id

Step 5c: No branches found → Send "assign branch" message
```

---

## 9. Error Handling

### 9.1 Page Load Errors

| Error | Cause | What User Sees |
|-------|-------|---------------|
| Missing chat_id | Bad URL | Error: "No supplier ID found" |
| Missing branch_id | Bad URL | Error: "No supplier Branch ID found" |
| Fetch failed | n8n down / network | Error: "Could not connect: [error message]" |
| No products | Empty pending sheet | Empty: "No products today" |

### 9.2 Confirmation Errors

| Error | Cause | What User Sees |
|-------|-------|---------------|
| Missing fields | Incomplete form | Toast: "Missing: Size, Unit..." |
| Invalid input | Non-numeric in number field | Red input border + field error text |
| Webhook failed | n8n error | Toast: "Could not save. Please try again." |
| Active recording | Mic still recording | Toast: "Stopping recording first..." |

### 9.3 Image Errors

| Error | Cause | What User Sees |
|-------|-------|---------------|
| Not an image | Wrong file type | Toast: "Please upload an image file" |
| Over 5MB | File too large | Toast: "Image too large. Please upload under 5MB" |
| Still too large after compression | Very dense image | Toast: "Image still too large. Try a smaller image." |
| Image load failed | Broken URL | Fallback "No image" placeholder shows |

### 9.4 Voice Recording Errors

| Error | Cause | What User Sees |
|-------|-------|---------------|
| No microphone | Permission denied | Toast: "Microphone permission denied" |
| Recording failed | Device error | Toast: "Could not start recording" |
| Already recording | Two cards | Toast: "Already recording another product" |
| Webhook failed | n8n error | Toast: "Could not process voice. Try again." |

---

## 10. Troubleshooting

### Problem: Page shows "Could not connect"
**Possible causes:**
1. n8n server is down
2. CORS policy blocking request
3. Webhook URL changed

**Check:**
- Open browser DevTools → Network tab
- Look for failed request to `review-data` webhook
- Check error message (CORS? 404? 500?)

---

### Problem: Products load but images don't show
**Possible causes:**
1. Google Drive sharing permissions not set to "Anyone with link"
2. Drive URL format not recognized

**Check:**
- Open the image URL directly in browser
- Make sure sharing is set to "Anyone with link can view"
- URL should contain `/file/d/{ID}/` or `?id={ID}`

---

### Problem: Confirm button does nothing
**Possible causes:**
1. Required fields missing (check for red borders)
2. Voice recording still active
3. Network request failing silently

**Check:**
- Look for red-bordered input fields
- Check if mic button is red (recording active)
- Open DevTools → Console for error messages

---

### Problem: Image upload fails with CORS error
**Cause:** Base64 image too large triggers preflight request  
**Fix:** Already handled by auto-compression. If still failing:
- Image may still be too large after compression
- Try a different, smaller image
- Ideal image: phone photo under 2MB

---

### Problem: "Confirm all" skips some products
**Cause:** Some products failed validation  
**Fix:** Check individual cards for:
- Red-bordered inputs (invalid data)
- Empty required fields
- Those products will need manual confirmation

---

## Appendix A: CSS Design Tokens

```css
:root {
  /* Colors */
  --primary:      #1E5BB8;   /* Main blue */
  --primary-dk:   #164A99;   /* Darker blue (hover) */
  --primary-lt:   #E8F0FB;   /* Light blue background */
  --ink:          #0F1E36;   /* Main text */
  --ink-3:        #5A6B85;   /* Secondary text */
  --success:      #16A34A;   /* Green */
  --danger:       #DC2626;   /* Red */
  --border:       #E2E8F0;   /* Standard border */

  /* Typography */
  --font-body:    'Inter', sans-serif;
  --font-display: 'Manrope', sans-serif;

  /* Radii */
  --r:            14px;      /* Card radius */
  --r-sm:         10px;      /* Input radius */
  --r-pill:       999px;     /* Badge/button radius */
}
```

---

## Appendix B: Supported Currencies

| Code | Currency |
|------|----------|
| BD | Bahraini Dinar (default) |
| USD | US Dollar |
| EGP | Egyptian Pound |
| INR | Indian Rupee |
| SAR | Saudi Riyal |
| AED | UAE Dirham |

---

## Appendix C: Supported Size Units

`kg, g, lb, oz, L, ml, pcs, piece, eggs, pack, box, carton`

---

## Appendix D: Webhook API Reference

### GET /webhook/review-data
**Purpose:** Fetch pending products for a supplier

**Request:**
```
GET https://n8n.srv1587311.hstgr.cloud/webhook/review-data?chat_id={chatId}
```

**Response:**
```json
[
  {
    "Product Name": "Grapes",
    "size": "10",
    "size_unit": "kg",
    "Color": "Black",
    "Origin": "India",
    "price": "4.000",
    "price_unit": "BD",
    "image url": "https://drive.google.com/...",
    "Category": "Fruits",
    "Pack": 1,
    "Box": 1,
    "row_number": 5
  }
]
```

---

### POST /webhook/review-confirm
**Purpose:** Submit a confirmed product

**Request:**
```
POST https://n8n.srv1587311.hstgr.cloud/webhook/review-confirm
Content-Type: application/json

{
  "Product_Name": "Grapes",
  "Color": "Black",
  "size": "10",
  "size_unit": "kg",
  "Origin": "India",
  "price": "4.000",
  "price_unit": "BD",
  "variant_name": "Black Grapes 10kg - India",
  "image_url": "https://...",
  "Category": "Fruits",
  "Pack": 1,
  "Box": 1,
  "chat_id": "97339392826",
  "branch_id": "55",
  "row_number": 5,
  "confirmed_at": "2026-05-06T10:30:00.000Z"
}
```

**Response:** HTTP 200 on success

---

### POST /webhook/voice-update
**Purpose:** Process voice recording and extract field values

**Request:**
```
POST https://n8n.srv1587311.hstgr.cloud/webhook/voice-update
Content-Type: application/json

{
  "chat_id": "97339392826",
  "branch_id": "55",
  "row_number": 5,
  "audio_base64": "UklGRi...",
  "audio_mime": "audio/webm",
  "current": {
    "product_name": "Grapes",
    "color": "Black",
    "size": "10",
    "size_unit": "kg",
    "price": "4.000",
    "price_unit": "BD",
    "Origin": "India"
  }
}
```

**Response:**
```json
{
  "product_name": "Grapes",
  "color": "Black",
  "size": "10",
  "size_unit": "kg",
  "price": "4.000",
  "price_unit": "BD",
  "Origin": "India",
  "transcript": "Black Grapes 10 kg 4 BD India"
}
```
