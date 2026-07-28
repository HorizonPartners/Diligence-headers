# Diligence Headers - Insert Logo & Text Box Into Section Documents

## Overview
This command inserts a branded header (logo image + "City, State" text box + decorative bar) into every Word document across all 23 diligence sections. It verifies the folder structure, collects user inputs, offers a Section 1 test run, then processes the remaining sections.

## Template Assets

The templates live in the `templates/` folder at the root of this repository (the same repo that contains this skill file):

- **Header XML template:** `templates/header2_template.xml`
- **Decorative bar image:** `templates/bar.png`

When resolving these paths, use the current working directory (the repo root). For example, if the user cloned to `C:\Users\jsmith\Diligence-headers`, the full path would be `C:\Users\jsmith\Diligence-headers\templates\bar.png`. Build the absolute path at runtime using the working directory.

The header template contains `{{CITY_STATE}}` placeholders that get replaced with the user's input.

---

## Step 1: Get the Sections File Path

Ask the user:

**"What is the full file path to the parent folder containing the numbered sections (e.g., `G:\Shared drives\...\Internal Staging (Shared)`)?"**

Once provided, verify the path exists. If it does not exist, report the error and ask the user to provide a corrected path.

---

## Step 2: Verify Folder Structure

Write and run a Python script that scans the provided path:

1. Look for folders matching the pattern `XX.)` where XX is 01 through 23
2. For each section folder found:
   - Recursively find all `.docx` files (excluding `~$` and `~WRL` temp files)
   - Count the documents
3. For each section folder NOT found, flag it

Present a **Verification Report** to the user in this format:

```
SECTION VERIFICATION
====================
[OK]  01.) General Corporate Documents — 10 documents
[OK]  02.) Corporate Org. and Info — 6 documents
...
[!!]  XX.) Section Name — NO FOLDER FOUND
[!!]  YY.) Section Name — 0 documents (folder exists but empty)
```

If any sections are flagged, ask the user: **"The above sections have issues. Would you like to proceed with the available sections, or fix the folder structure first?"**

If everything looks good, confirm: **"All X sections found with Y total documents. Ready to proceed."**

---

## Step 3: Header Content Audit

**This step is critical.** Before collecting any inputs, scan every document's header to detect what already exists. This catches leftover logos from past clients and inconsistent states across sections.

Write and run a Python script that inspects every `.docx` file found in Step 2. For each file:

1. **Copy locally first** (Google Drive workaround — Python's `zipfile` cannot read directly from G: drive)
2. Open as a ZIP and read `word/header2.xml` and `word/_rels/header2.xml.rels`
3. **Detect header state** by classifying into one of these categories:

| Category | How to detect |
|----------|---------------|
| **FULL HEADER** | `header2.xml` size >= 10000 bytes (has text box + logo + bar) |
| **BAR ONLY** | `header2.xml` size is 3500-6000 bytes AND `header2.xml.rels` has exactly 1 image relationship |
| **CLEAN** | `header2.xml` size < 3500 bytes OR no `header2.xml.rels` OR rels has 0 image relationships |
| **UNKNOWN** | Anything else — flag for manual review |

4. **If FULL HEADER**, extract existing content:
   - **City/State text:** Parse `header2.xml` with regex to find text inside the text box. The text box lives inside `<wps:txbx><w:txbxContent>` and the city/state is in `<w:t>` tags within that block. Use this regex pattern on the XML to extract it: find the `<wps:txbx>` block, then extract all `<w:t[^>]*>([^<]+)</w:t>` matches within it. Join the matches — that is the existing city/state text.
   - **Logo image:** Check `header2.xml.rels` for image relationships. If there are 2 relationships, one points to the bar (13,390 bytes) and the other is the logo. Record the logo filename and its size in bytes.

5. **Collect results** into a summary grouped by state.

### Present the Audit Report

Display the results to the user in this format:

```
DOCUMENT HEADER AUDIT
=====================

FULL HEADER (logo + text box + bar): X documents
  City/State found: "New York City, NY"
  Logo: image4.png (21,445 bytes)
  Files:
    01.01_Organizational Company Documents.docx
    01.03_List of Business Jurisdictions.docx
    ... (show all, or first 10 + "and N more" if > 15)

BAR ONLY (no logo, no text box): Y documents
  Files:
    05.01_Bank Lines of Credit Agreements.docx
    ... (show all, or first 10 + "and N more")

CLEAN (no header content): Z documents
  Files:
    ...

UNKNOWN (needs manual review): W documents
  Files:
    ...
```

**If multiple different city/state texts are found** (e.g., some docs say "New York City, NY" and others say "Austin, TX"), break out the FULL HEADER group by city/state:

```
FULL HEADER — "New York City, NY": 45 documents
FULL HEADER — "Austin, TX": 12 documents
FULL HEADER — (text could not be extracted): 3 documents
```

**If multiple different logo sizes are found**, note that too — it may indicate different client logos mixed in:

```
  Logo sizes found: 21,445 bytes (230 docs), 18,200 bytes (5 docs)
  WARNING: Multiple logo sizes detected — some documents may have a different client's logo
```

### Ask the User How to Proceed

After presenting the audit, ask:

**"Based on the audit above, how would you like to proceed?"**

Provide options:
- **"Replace all headers"** — overwrite everything (FULL HEADER, BAR ONLY, and CLEAN docs all get the new header with the new logo and city/state)
- **"Only update documents without a full header"** — skip docs that already have FULL HEADER, only update BAR ONLY and CLEAN docs
- **"Strip existing headers only"** — remove the logo and text box from all FULL HEADER documents, leaving them in BAR ONLY state (no new logo or city/state inserted). This is a cleanup-only mode.
- **"Let me review first"** — user wants to investigate before proceeding

Store the user's choice to control behavior in later steps.

**If the user chose "Strip existing headers only":** Skip Steps 4 and 5 (no need to collect a new logo or city/state). Jump directly to Step 6, then Step 7/8 in strip mode.

### Strip Mode Logic

When the user chose "Strip existing headers only", the processing script in Step 8 works differently for FULL HEADER documents:

1. Instead of inserting a new header, **restore the BAR ONLY header state**:
   - Replace `word/header2.xml` with a minimal header that only references the bar image (rId1). Use the BAR ONLY header XML from any existing BAR ONLY document as the template — read it from the first BAR ONLY doc found during the audit. If no BAR ONLY docs exist, use a clean header that contains only the bar image reference.
   - Replace `word/_rels/header2.xml.rels` with a single-relationship rels file: `rId1 -> media/image2.png` (the bar)
   - **Remove the old logo image file** from the ZIP (the `word/media/imageN.png` that was the logo — identified during the audit by checking which image in header2.xml.rels is NOT the bar). Do NOT remove the bar image or any other media files.
2. BAR ONLY and CLEAN documents are left untouched in strip mode.
3. The final report should say "Documents stripped" instead of "Documents updated".

---

## Step 4: Get the Logo

**Skip this step if the user chose "Strip existing headers only" in Step 3.**

Ask the user:

**"What is the file path to the logo image? (PNG, JPG, or EMF)"**

Validate:
- The file exists
- The file extension is a supported image format (`.png`, `.jpg`, `.jpeg`, `.emf`, `.bmp`)
- The file is readable and not empty

If validation fails, report the specific issue and ask the user to provide a corrected path.

Read the logo image file into memory for later use.

---

## Step 5: Get City, State

**Skip this step if the user chose "Strip existing headers only" in Step 3.**

Ask the user:

**"Where is the company headquartered? (City, State format — e.g., 'New York City, NY')"**

Validate:
- The input is not empty
- The input contains a comma (City, State format)

If the format looks off (e.g., no comma, just a city name, or something unexpected), ask the user to clarify: **"The expected format is 'City, State' (e.g., 'Austin, TX'). Did you mean: [their input]?"**

---

## Step 6: Show Configuration Summary

Display a summary before proceeding:

```
DILIGENCE HEADER CONFIGURATION
===============================
Sections Path: [path]
Total Sections: [X]
Total Documents: [Y]
Mode: [Replace all / Only update without full header / Strip existing]
Logo: [filename] ([size] KB)          <-- omit in strip mode
Headquarters: [City, State]           <-- omit in strip mode
Documents to process: [N] (out of [Y] total)
Documents to skip: [M]
```

---

## Step 7: Section 1 Test Run

Ask the user:

**"Would you like to run Section 1 first to verify the header looks correct before processing all sections?"**

If yes:
1. Process ONLY the documents in Section 01
2. Tell the user: **"Section 1 complete. Please open any document from Section 01 in Word and verify the header shows the logo on the left, '[City, State]' text on the right, and the decorative bar below. Does it look correct?"**
3. If the user says it looks correct, proceed to Step 7
4. If the user reports issues, troubleshoot and ask what needs to change

If no, proceed directly to Step 8.

---

## Step 8: Process All Remaining Sections

Write and run a Python script that processes every document across all sections (skipping Section 01 if already done in Step 7).

### Script Logic

The script must follow these rules exactly:

1. **Read template assets:**
   - Read `header2_template.xml` from the template assets path
   - Replace `{{CITY_STATE}}` with the user's city/state input to produce `final_header2_xml`
   - Read `bar.png` from the template assets path
   - Read the user's logo image file

2. **Google Drive workaround:** Python's `zipfile` cannot read directly from Google Drive File Stream (G: drive). For every file:
   - Copy the target `.docx` to a local temp directory first
   - Perform all ZIP operations on the local copy
   - Copy the modified file back to the original path

3. **For each .docx file:**
   - Back up the original to a backups directory (organized by section)
   - Copy locally, read all ZIP entries into memory
   - Check if `word/header2.xml` already has the full header (size >= 10000 bytes). **If the user chose "Only update documents without a full header" in Step 3, skip these files.** If the user chose "Replace all headers", process them anyway (overwrite the existing header with the new one).
   - Determine if the file already has the decorative bar image: check if `word/media/image2.png` exists with size 13,390 bytes
   - Find the next available image number (look at existing `word/media/imageN.*` files, take max N + 1)
   - **If bar exists (standard template ~450 KB):**
     - Add logo as `word/media/image[N].png`
     - Build rels: rId1 -> logo image, rId2 -> existing `media/image2.png` (bar)
   - **If bar does NOT exist (different template ~75 KB):**
     - Add logo as `word/media/image[N].png`
     - Add bar as `word/media/image[N+1].png`
     - Build rels: rId1 -> logo image, rId2 -> new bar image
   - Replace `word/header2.xml` with `final_header2_xml` (encoded to UTF-8)
   - Replace `word/_rels/header2.xml.rels` with the constructed rels XML
   - Write the new ZIP preserving original compression types (ZIP_DEFLATED for XML, ZIP_STORED for images)
   - Validate: confirm the new image exists in the written ZIP
   - Copy modified file back to the Google Drive path

4. **Print progress per section** and a final summary:
   ```
   FINAL SUMMARY
     Total files: X
     Succeeded: Y
     Failed: Z
     Skipped: W (already had full header)
   ```

5. **If any failures**, list them clearly so the user can investigate.

### Important Implementation Notes

- **Never use bash heredocs for Python scripts** — always use the Write tool to create .py files
- **ASCII-only in print() statements** — no Unicode arrows, em-dashes, or special characters (Windows cp1252 will crash)
- **Always back up before modifying** — save originals to a scratchpad backups directory
- **Use `zipfile.ZipInfo` with correct `compress_type`** when writing entries:
  - `ZIP_DEFLATED` for `.xml` and `.rels` files
  - `ZIP_STORED` for image files (`.png`, `.jpg`, `.jpeg`, `.emf`)
- **Add a 0.5s sleep between sections** to avoid overwhelming Google Drive sync
- **Rels XML format:**
  ```xml
  <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
  <Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">
    <Relationship Id="rId2" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/image" Target="media/[bar_filename]"/>
    <Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/image" Target="media/[logo_filename]"/>
  </Relationships>
  ```

---

## Error Handling

- **File path not found:** Report the exact path that failed and ask the user to correct it
- **Logo file unreadable:** "Error: Cannot read the logo file at [path]. Please check the file exists and is not locked."
- **ZIP read failure:** If a document fails to read as a ZIP, copy it locally first (Google Drive workaround). If it still fails, skip it and report.
- **Excel/Word lock files:** Skip any files starting with `~$` or `~WRL` — these are temporary lock files
- **Per-file error isolation:** Wrap each file's processing in a try/except so one failure does not stop the batch

---

## Step 9: Final Report

After all sections are processed, display:

```
HEADER INSERTION COMPLETE
=========================
Sections processed: [X]
Documents updated: [Y]
Documents skipped: [Z] (already had header)
Documents failed: [W]
Backups location: [path]
```

If any documents failed, list them. If any sections were missing or empty, remind the user.

Tell the user: **"Please spot-check a few documents across different sections in Word to confirm the headers look correct."**
