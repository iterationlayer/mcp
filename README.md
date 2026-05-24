The Iteration Layer API includes a built-in [Model Context Protocol (MCP)](https://modelcontextprotocol.io) server. MCP lets AI agents like Claude Desktop, Cursor, and Windsurf call the API directly as tools — no custom integration code required.

## Server Endpoint

The MCP server is available via Streamable HTTP transport:

```
https://api.iterationlayer.com/mcp
```

## Authentication

The MCP endpoint uses **OAuth 2.1** — not API keys. Your MCP client handles the authentication flow automatically. When you first use an Iteration Layer tool, a browser window will open for you to log in and authorize access.

## Project Scoping

MCP tool usage is associated with your organization through OAuth. It is not scoped to a project or project-specific API key.

For project-level usage tracking, run the workflow through the REST API or n8n with a project-scoped API key. The dashboard's API and n8n onboarding paths create the project and scoped key together.

## First Tool Call

After OAuth completes, ask your MCP client:

```text
Use Iteration Layer to convert https://iterationlayer.com/code-samples/accounts-payable-invoice.pdf to markdown and summarize the result.
```

This hosted sample lets you verify the full OAuth-to-tool-call loop before you bring your own document.

## Available Tools

The MCP server exposes one tool per Iteration Layer API:

### convert_document_to_markdown

Convert a document or image to clean markdown. Supports PDF, DOCX, XLSX, images (PNG, JPG, GIF, WebP), HTML, Markdown, CSV, JSON, and plain text.

**Parameters:**
- `file` (required) — File input with `type`, `name`, and either `base64` or `url`

See [Document to Markdown](https://iterationlayer.com/docs/document-to-markdown) for the full API reference.

### extract_document

Extract structured data from documents using AI. Supports PDF, DOCX, images (PNG, JPG, GIF, WebP), and text files (MD, TXT, CSV, JSON).

**Parameters:**
- `files` (required) — Array of file inputs (each with `type`, `name`, and either `base64` or `url`)
- `schema` (required) — Extraction schema with a `fields` array defining what to extract

See [Document Extraction](https://iterationlayer.com/docs/document-extraction) for the full schema reference.

### extract_website

Extract structured data from one public website page using the same extraction schema system as Document Extraction.

**Parameters:**
- `file` (required) — Website URL input with `type: "url"`, `url`, optional `name`, and optional `fetch_options`
- `schema` (required) — Extraction schema with a `fields` array defining what to extract

```json
{
  "file": {
    "type": "url",
    "url": "https://example.com/pricing"
  },
  "schema": {
    "fields": [
      {
        "name": "plan_name",
        "type": "TEXT",
        "description": "The name of the pricing plan"
      }
    ]
  }
}
```

See [Website Extraction](https://iterationlayer.com/docs/website-extraction) for the full schema reference.

### transform_image

Transform and manipulate images. Supported operations: resize, crop, extend, trim, rotate, flip, flop, blur, sharpen, modulate, tint, grayscale, invertColors, autoContrast, gamma, removeTransparency, threshold, denoise, opacity, convert, upscale, smartCrop, compressToSize, removeBackground.

**Parameters:**
- `file` (required) — File input with `type`, `name`, and either `base64` or `url`
- `operations` (required) — Array of transformation operations to apply in order

See [Image Transformation](https://iterationlayer.com/docs/image-transformation) for the full operations reference.

### generate_image

Generate images from layer compositions. Layer types: solid-color, gradient, text (with Markdown bold/italic), image, qr-code, barcode, and layout (nested flex-like container with direction, gap, alignment, and padding). Layers are composited by index order with per-layer opacity and rotation.

See [Image Generation](https://iterationlayer.com/docs/image-generation) for the full composition schema.

### generate_document

Generate PDF, DOCX, PPTX, or EPUB documents from structured definitions. Documents are composed of metadata, page settings, styles, and content blocks (paragraph, headline, image, table, grid, list, table-of-contents, page-break, separator, QR code, barcode).

See [Document Generation](https://iterationlayer.com/docs/document-generation) for the full document schema.

### generate_sheet

Generate XLSX, CSV, or Markdown spreadsheets from structured tabular data. Define columns and positional rows of cells with optional formatting (currency, percentage, number, decimal, date, datetime, time, custom), cell styles, merged cell spans, and Excel formulas. Supports multiple sheets, ISO 4217 currency codes, and custom fonts.

See [Sheet Generation](https://iterationlayer.com/docs/sheet-generation) for the full schema reference.

## Client Setup

### Claude Desktop

Add to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "iterationlayer": {
      "type": "http",
      "url": "https://api.iterationlayer.com/mcp"
    }
  }
}
```

### Cursor

Add to your `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "iterationlayer": {
      "type": "http",
      "url": "https://api.iterationlayer.com/mcp"
    }
  }
}
```

### Windsurf

Add to your Windsurf MCP configuration:

```json
{
  "mcpServers": {
    "iterationlayer": {
      "type": "http",
      "url": "https://api.iterationlayer.com/mcp"
    }
  }
}
```

## How Input and Output Formats Work

Every tool follows the same file input convention:

```json
{
  "type": "url",
  "name": "document.pdf",
  "url": "https://example.com/document.pdf"
}
```

Or for inline content:

```json
{
  "type": "base64",
  "name": "document.pdf",
  "base64": "JVBERi0xLjQK..."
}
```

Every tool that produces binary output (images, documents, spreadsheets) returns the same structure:

```json
{
  "buffer": "base64-encoded-content",
  "mime_type": "application/pdf"
}
```

This consistency matters for composability. The output format of `generate_image` matches the input format of `generate_document`'s image blocks. The agent does not need to convert between formats or understand format-specific encoding.

## Composability Patterns

All tools share a single endpoint, the same authentication, the same credit pool, and compatible I/O formats. This makes multi-step agent workflows natural.

For production workflows, keep confidence, review, and audit decisions explicit in your application logic. An agent can call `extract_document`, but your workflow should still gate low-confidence or high-impact fields before generating artifacts, updating records, or sending customer-facing output. See [Workflow Design Best Practices](https://iterationlayer.com/docs/workflow-best-practices) for confidence gates, review branches, audit records, approved values, retention boundaries, and data-flow behavior.

### Extract and Report

Extract structured data from a document, then generate a report from the extracted data.

**Flow:** `extract_document` → application logic → `generate_document`

The extraction returns structured JSON. The document generation accepts structured JSON. No format conversion, no intermediate files. The agent reads the extraction result, builds a document definition from it, and generates the output.

### Convert, Extract, and Generate

When the document needs to be understood as a whole, convert it to markdown first, then extract specific data, then generate output.

**Flow:** `convert_document_to_markdown` → `extract_document` → `generate_document` or `generate_sheet`

The markdown conversion gives the agent the full document text for context. The extraction pulls specific structured fields. This pattern works well when the agent needs to both summarize content (from the markdown) and pull precise values (from the extraction).

### Extract, Transform, and Compose

Combine document data with processed images to create visual output.

**Flow:** `extract_document` → `transform_image` → `generate_image`

The extraction provides the data (product name, price, description). The image transformation prepares visual assets (resize, background removal, format conversion). The image generation composites everything into a final visual.

### Batch Extract to Spreadsheet

Process multiple documents and aggregate results into a single spreadsheet.

**Flow:** `extract_document` (multiple calls) → `generate_sheet`

Each extraction returns structured data for one document. The agent collects the results and generates a spreadsheet with one row per document. Confidence scores can be included as a column for filtering.

## Agent Use Cases

### Invoice Processing Pipeline

The agent extracts vendor name, invoice number, line items, totals, and payment terms from supplier invoices. It evaluates confidence scores, generates a PDF summary for approved invoices and a review report for flagged ones, then exports the batch to an XLSX spreadsheet for accounting.

**Tools used:** `extract_document` → confidence routing → `generate_document` (summary or review) → `generate_sheet` (batch export)

### Content Repurposing

The agent converts a long-form PDF whitepaper to markdown, extracts the title, author, key takeaways, and statistics, generates a social card image with the title and a branded gradient, and generates a one-page PDF executive summary.

**Tools used:** `convert_document_to_markdown` → `extract_document` → `generate_image` (social card) → `generate_document` (summary)

### Product Catalog Processing

The agent extracts product names, descriptions, prices, and SKUs from a supplier catalog PDF. It takes each product photo, removes the background, resizes to marketplace-required dimensions, and converts to WebP. It then generates product card images with standardized layouts.

**Tools used:** `extract_document` → `transform_image` (per product) → `generate_image` (per product)

### Contract Review Preparation

The agent converts a contract to markdown for full-text review, extracts structured data (parties, effective date, termination clauses, payment terms, governing law), and generates a review checklist as a DOCX with the extracted terms pre-filled and blank fields for reviewer notes.

**Tools used:** `convert_document_to_markdown` → `extract_document` → `generate_document`

## How MCP Tools Map to SDK Methods

MCP is ideal for interactive agent workflows — Claude Code sessions, Cursor conversations, ad-hoc processing tasks. For production pipelines, the same tools are available through the TypeScript, Python, and Go SDKs.

| MCP Tool | TypeScript | Python | Go |
|----------|-----------|--------|-----|
| `convert_document_to_markdown` | `client.convertDocumentToMarkdown()` | `client.convert_document_to_markdown()` | `client.ConvertDocumentToMarkdown()` |
| `extract_document` | `client.extractDocument()` | `client.extractDocument()` | `client.ExtractDocument()` |
| `extract_website` | `client.extractWebsite()` | `client.extract_website()` | `client.ExtractWebsite()` |
| `transform_image` | `client.transformImage()` | `client.transformImage()` | `client.TransformImage()` |
| `generate_image` | `client.generateImage()` | `client.generate_image()` | `client.GenerateImage()` |
| `generate_document` | `client.generateDocument()` | `client.generate_document()` | `client.GenerateDocument()` |
| `generate_sheet` | `client.generateSheet()` | `client.generate_sheet()` | `client.GenerateSheet()` |

Every method also has an async variant that accepts a `webhook_url` for non-blocking processing:

| TypeScript | Python | Go |
|-----------|--------|-----|
| `client.extractDocumentAsync()` | `client.extract_document_async()` | `client.ExtractDocumentAsync()` |
| `client.extractWebsiteAsync()` | `client.extract_website_async()` | `client.ExtractWebsiteAsync()` |
| `client.transformImageAsync()` | `client.transform_image_async()` | `client.TransformImageAsync()` |
| `client.generateImageAsync()` | `client.generate_image_async()` | `client.GenerateImageAsync()` |
| `client.generateDocumentAsync()` | `client.generate_document_async()` | `client.GenerateDocumentAsync()` |
| `client.generateSheetAsync()` | `client.generate_sheet_async()` | `client.GenerateSheetAsync()` |

The parameters, request shapes, and response shapes are identical between MCP and SDK. A workflow prototyped in Claude Code via MCP translates directly to production code via the SDK.

## Listings

Find the Iteration Layer MCP server on these directories:

- [Official MCP Registry](https://registry.modelcontextprotocol.io) — `com.iterationlayer/mcp`
- [Glama.ai](https://glama.ai/mcp/connectors/com.iterationlayer/mcp) — score `A4.2/5.0`
- [Smithery.ai](https://smithery.ai/servers/iterationlayer/iterationlayer)

## Billing

MCP tool calls consume credits the same way as direct API calls. See [Credits & Pricing](https://iterationlayer.com/docs/credits-and-pricing) for details.