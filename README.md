# AI-Powered Receipt Data Extraction & Annotation Pipeline

An automated receipt-processing pipeline that uses a vision-capable Large Language Model (LLM) to extract structured receipt and product information from receipt images and write the results directly into Google Sheets.

The project was developed in Google Colab for an Upwork data extraction and annotation workflow. It combines multimodal AI, structured JSON extraction, Google Sheets automation, batch processing, error handling, and automated quality checks.

---

Project Overview

Manually extracting information from receipt images is time-consuming and difficult to scale, particularly when each receipt contains different layouts, product formats, currencies, discounts, or partially unreadable information.

This project automates much of that workflow.

The pipeline:

1. Authenticates with Google services through Google Colab.
2. Reads receipt image URLs and identifiers from Google Sheets.
3. Sends receipt images to a vision-capable LLM.
4. Extracts structured receipt-level information.
5. Extracts individual product line items.
6. Parses and validates the model's JSON output.
7. Writes the extracted information back to designated Google Sheets.
8. Processes receipts in batches while avoiding already-completed records.
9. Tracks known extraction/JSON failures.
10. Performs automated quality checks on the resulting annotation data.

The result is a structured annotation workflow that reduces manual extraction effort while providing checks for common data-quality problems.

---

Project Objectives

The main objectives were to:

- Automate receipt information extraction from images.
- Extract both receipt-level and product-level information.
- Produce consistent structured JSON output from an LLM.
- Integrate AI extraction directly with Google Sheets.
- Support batch processing of multiple receipts.
- Prevent unnecessary reprocessing of completed records.
- Handle malformed or incomplete LLM responses.
- Identify extraction records requiring manual review.
- Validate relationships between receipt and product data.

---

Pipeline Architecture

                 Google Sheets
                      │
                      │
               Receipt IDs
               + Image URLs
                      │
                      ▼
             Google Colab / Python
                      │
                      ▼
             Vision-capable LLM
                ┌─────┴─────┐
                │           │
                ▼           ▼
        Receipt Details   Product Details
                │           │
                └─────┬─────┘
                      │
                      ▼
              JSON Parsing
                      │
                      ▼
             DataFrame Processing
                      │
                      ▼
                Google Sheets
                      │
                      ▼
             Automated Quality
                  Checks
                      │
                      ▼
              Manual Review Flags

---

Key Features

1. Google Authentication and Sheets Integration

The notebook uses Google Colab authentication and "gspread" to interact with Google Sheets.

from google.colab import auth
from google.auth import default
import gspread

auth.authenticate_user()

creds, _ = default()
gc = gspread.authorize(creds)

This allows the pipeline to read source data and write extracted results directly to Google Sheets without manually downloading and uploading files.

---

2. Loading Google Sheet Data

The "get_sheet()" function retrieves a worksheet and converts its contents into a Pandas DataFrame.

def get_sheet(sheet_name, worksheet_name="Data"):
    spreadsheet = gc.open(sheet_name)
    worksheet = spreadsheet.worksheet(worksheet_name)

    data = worksheet.get_all_values()

    df_sheet = pd.DataFrame(
        data[1:],
        columns=data[0]
    )

    return df_sheet

This creates a convenient bridge between the spreadsheet-based annotation workflow and Python data processing.

---

3. Writing Extracted Data Back to Google Sheets

The pipeline includes reusable functions for writing Pandas DataFrames back into Google Sheets.

The extraction results can therefore flow directly from:

Receipt Image
     ↓
LLM
     ↓
Python DataFrame
     ↓
Google Sheet

The implementation also handles missing Pandas values before writing them to Google Sheets.

---

Multimodal LLM Extraction

The core extraction function sends both a text prompt and an image URL to a vision-capable LLM through an OpenAI-compatible API interface.

from openai import OpenAI

def extract_text(
    image_url: str,
    prompt: str,
    model: str,
    base_url: str,
    api_key: str = None,
    max_tokens: int = 2000,
    temperature: float = 0.0,
):
    ...

The model receives:

- A structured extraction instruction
- The receipt image
- A required JSON output format

This makes the workflow suitable for extracting information from receipts without relying exclusively on traditional OCR techniques.

---

LLM Configuration

The implementation uses an OpenAI-compatible endpoint and configures the model dynamically.

Example configuration:

model = "qwen/qwen3.6-27b"
base_url = "https://api.groq.com/openai/v1"
api_key = userdata.get("GROQ_API_KEY")

The API key is retrieved through Google Colab's user data/secrets mechanism rather than being hard-coded into the notebook.

---

Receipt-Level Extraction

The first extraction prompt focuses on information describing the receipt as a whole.

The pipeline extracts fields such as:

- Receipt ID
- Receipt number
- Receipt date
- Receipt time
- Subtotal
- Tax amount
- Discount amount
- Total amount
- Currency
- Payment method
- OCR raw text
- Savings amount
- Loyalty points earned
- Loyalty points redeemed
- Loyalty card number
- Loyalty program
- Store name
- Store location
- Image URL
- Extraction model
- Processing status

The model is explicitly instructed to use "null" when information is missing or unreadable rather than guessing.

---

Product-Level Extraction

A second extraction process identifies individual purchased items on the receipt.

Each product record contains fields including:

Field| Description
"raw_text"| Product text as it appears on the receipt
"item_name"| Lightly cleaned product name
"quantity"| Extracted quantity
"unit_price"| Price per unit
"discount"| Product-level discount when visible
"total_price"| Extracted line-item total
"sequence_order"| Position of the item on the receipt
"Reciept index"| Identifier linking the product to its receipt

The prompt specifically instructs the model not to invent missing values and not to reconstruct obscured prices using arithmetic.

For example, if part of a number is unreadable, the system uses "null" rather than attempting to guess the missing digit.

---

Structured JSON Processing

Because LLM responses can sometimes contain additional formatting around JSON, the notebook includes a reusable parser.

The parser handles:

- "<think>...</think>" blocks
- Fenced JSON responses
- Generic fenced code blocks
- Plain JSON responses

Conceptually:

LLM Response
     │
     ▼
Remove reasoning blocks
     │
     ▼
Detect JSON code block
     │
     ▼
Extract JSON
     │
     ▼
Parse with json.loads()
     │
     ▼
Structured Python object

This makes the extraction workflow more resilient to variations in model output formatting.

---

Batch Processing

The pipeline supports processing multiple receipts rather than requiring each receipt to be processed manually.

A configurable batch size is used:

BATCH_SIZE = 28

The system identifies receipts that have not yet been fully processed and processes only the required records.

This is particularly useful for large annotation datasets where processing the entire dataset repeatedly would waste API calls.

---

Avoiding Duplicate Processing

Before processing a batch, the notebook checks the existing Google Sheets data.

It identifies:

- Receipt IDs already present in the Receipt Details sheet.
- Receipt IDs already associated with Product Details.

The pipeline then determines which receipts still require processing.

Conceptually:

All Receipts
     │
     ├── Receipt Details exists
     │
     ├── Product Details exists
     │
     └── Not fully processed
              │
              ▼
        Process only these

This makes the workflow incremental rather than requiring the entire dataset to be reprocessed every time the notebook runs.

---

Failure Tracking

LLM-generated JSON can occasionally be incomplete or malformed.

The pipeline therefore maintains a file:

known_json_failures.json

Receipt IDs that encounter JSON parsing failures can be recorded for later investigation or retry.

This provides a simple mechanism for tracking problematic records across notebook runs.

---

Rate-Limit Handling

The batch-processing workflow also detects API rate-limit errors.

If a rate-limit condition such as:

429

or:

rate_limit_exceeded

is detected, the current batch is stopped rather than continuing to make requests that are likely to fail.

This helps avoid unnecessary API calls and makes the batch process safer to resume later.

---

Automated Data Quality Checks

One of the important components of the project is a separate automated quality-control stage.

The quality-check section reads the existing annotation sheets without modifying them.

It evaluates both:

- Receipt Details
- Product Details

The checks are designed to identify records that deserve manual review.

---

Quality Check 1 — Dataset Structure

The pipeline verifies that the expected receipt and product columns exist.

This helps identify structural problems before performing deeper validation.

---

Quality Check 2 — Receipt ID Integrity

The system checks for:

- Duplicate receipt IDs
- Missing receipt IDs
- Product records linked to nonexistent receipts

This helps maintain the relationship between receipt-level and product-level datasets.

---

Quality Check 3 — Required Field Completeness

The system calculates missing-value counts and percentages for important receipt fields.

Examples include:

- "receipt_number"
- "receipt_date"
- "receipt_time"
- "total_amount"
- "currency"
- "payment_method"
- "image_url"
- "ocr_raw_text"

The same type of check is performed for product fields.

---

Quality Check 4 — Numeric Field Validation

The pipeline verifies that fields expected to contain numeric values can actually be interpreted as numbers.

Receipt-level numeric fields include:

subtotal
tax_amount
discount_amount
total_amount
savings_amount
loyalty_points_earned
loyalty_points_redeemed

Product-level numeric fields include:

quantity
unit_price
discount
total_price

Non-numeric values are flagged for review.

---

Quality Check 5 — Product Price Consistency

Where quantity, unit price, and total price are all available, the system calculates:

quantity × unit_price

and compares the result with the extracted "total_price".

Differences greater than the defined tolerance are flagged for manual review.

This helps detect potential extraction errors in product-level pricing.

---

Quality Check 6 — Product Sequence

Each product is expected to have a sequential order:

1
2
3
4
...

The system groups products by receipt and checks whether the sequence is complete and correctly ordered.

For example:

Expected:
1, 2, 3, 4

Potential issue:
1, 2, 4, 5

Such records are flagged for review.

---

Quality Check 7 — Receipt/Product Coverage

The pipeline checks whether every receipt has associated product records.

It also calculates the distribution of the number of products per receipt.

This helps identify receipts that may have been processed at the receipt level but failed during product extraction.

---

Quality Check 8 — Suspicious Product Records

The system identifies product records where both:

- "item_name"
- "raw_text"

are missing.

These records are considered suspicious because they contain no useful identifying product information.

---

Quality Check 9 — Receipt Total vs Product Total

The pipeline compares:

Receipt Total

against:

Sum of Extracted Product Totals

The absolute difference is calculated for each receipt.

Receipts with a difference above the defined tolerance are flagged for manual review.

This provides an additional cross-level consistency check between the two annotation tables.

---

Example Quality-Control Output

The demonstrated run processed:

Receipt records: 124
Product records: 367
Unique receipts: 124

The receipt ID integrity checks showed:

Duplicate receipt IDs: 0
Missing receipt IDs: 0
Products linked to non-existent receipts: 0

The completeness checks also provided field-level missing-value statistics.

For example:

receipt_id        0 missing
receipt_number    0 missing
total_amount      0 missing
image_url         0 missing
ocr_raw_text      0 missing

The product dataset showed complete values for several core fields, while "discount" was frequently missing because discounts were not present or extractable for many line items.

These results demonstrate how the quality-control layer can distinguish between expected missing information and potential structural or extraction problems.

---

Technologies Used

- Python
- Google Colab
- Pandas
- NumPy
- gspread
- Google Authentication
- OpenAI Python SDK
- Vision-capable LLM
- Groq-compatible API
- JSON
- Regular Expressions
- Google Sheets

---

Project Structure

The project is implemented primarily as a Google Colab notebook.

A recommended repository structure is:

receipt-data-extraction/
│
├── OCR_Image_Data_Annotation.ipynb
├── known_json_failures.json
├── README.md
└── requirements.txt

---

Workflow Summary

The complete workflow can be summarized as:

1. Authenticate Google Account
            ↓
2. Connect to Google Sheets
            ↓
3. Load Receipt IDs + Image URLs
            ↓
4. Identify Unprocessed Receipts
            ↓
5. Send Receipt Image to Vision LLM
            ↓
6. Extract Receipt-Level Data
            ↓
7. Extract Product-Level Data
            ↓
8. Parse and Validate JSON
            ↓
9. Write Results to Google Sheets
            ↓
10. Track Failed Records
            ↓
11. Run Automated Quality Checks
            ↓
12. Flag Records for Manual Review

---

Important Design Decisions

Do Not Guess Missing Information

The extraction prompts explicitly instruct the model to use "null" when information is unavailable or unreadable.

This is important for annotation work because an incorrect value can be more damaging than a missing value.

Preserve Raw Receipt Text

The pipeline retains an "ocr_raw_text" field so the original extracted text can be inspected alongside the structured fields.

Separate Receipt and Product Data

Receipt-level information and product-level information are written to separate Google Sheet worksheets.

This preserves a clear one-to-many relationship:

Receipt
   │
   ├── Product 1
   ├── Product 2
   ├── Product 3
   └── ...

Separate Extraction from Quality Control

The automated quality checks operate on the existing annotation data rather than modifying it.

This makes the validation stage safer and easier to audit.

---

Limitations

This is an AI-assisted extraction and annotation pipeline, not a guarantee of perfect OCR or data accuracy.

Potential issues include:

- Poor image quality
- Unusual receipt layouts
- Handwritten or crossed-out text
- Partially obscured numbers
- Ambiguous product names
- Model-generated formatting errors
- Missing information on the original receipt
- LLM extraction errors

The automated quality checks therefore identify records requiring review rather than claiming that every flagged record is incorrect.

Manual comparison against the original receipt image remains important for high-accuracy annotation.

---

Potential Improvements

Future versions could improve the workflow through:

Better Structured Outputs

Use strict schema validation or structured-output capabilities to reduce JSON parsing failures.

Retry Strategy

Automatically retry failed extractions using controlled retry limits and alternative prompts.

Confidence Scores

Store model confidence or field-level confidence indicators where supported.

Image Preprocessing

Add image enhancement, cropping, rotation correction, or resolution improvement before sending receipts to the vision model.

Human-in-the-Loop Review

Create a review workflow that presents flagged records alongside the original receipt image.

Automated Correction

Allow manually verified corrections to be fed back into the annotation workflow.

Monitoring Dashboard

Track:

- Number of processed receipts
- Successful extractions
- Failed extractions
- JSON failures
- Missing fields
- Product inconsistencies
- Receipt/product mismatches
- Manual-review volume

---

Outcome

The project produced an automated workflow capable of processing receipt images into two structured datasets:

Receipt Details

Contains receipt-level information such as:

receipt_id
receipt_number
receipt_date
receipt_time
subtotal
tax_amount
discount_amount
total_amount
currency
payment_method
store name
store location
...

Product Details

Contains individual purchased products such as:

raw_text
item_name
quantity
unit_price
discount
total_price
sequence_order
Reciept index

The demonstrated quality-control run contained 124 receipt records and 367 product records, with automated checks confirming no duplicate receipt IDs, no missing receipt IDs, and no orphan product records in that run.

---

What This Project Demonstrates

This project demonstrates practical experience with:

- Multimodal Generative AI
- LLM-based information extraction
- Prompt engineering
- Structured JSON generation
- Computer vision / image understanding through LLMs
- Google Sheets automation
- Python data processing
- Batch data pipelines
- API integration
- Error handling
- Rate-limit handling
- Data validation
- Automated quality assurance
- Human-in-the-loop review workflows

More importantly, the project demonstrates how an LLM can be integrated into an actual data-processing workflow, rather than being used only as a conversational chatbot.

---

Author

Ibrahim Yisau

AI / Machine Learning | Generative AI | Data & Automation

GitHub: Mrtobiss

---

Project Context

This project was completed as an Upwork data extraction and annotation task, involving the development and refinement of an AI-assisted workflow for extracting structured information from receipt images and validating the resulting annotation data.
