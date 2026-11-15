# Feishu Document Normalization

Apply the main skill's width, spacing, and blue heading-number rules. Preserve the wording, content order, and other formatting.

## Identify and Read the Target

- Use the installed `lark-doc` skill and read its references for fetching, updating, and XML at the corresponding stage. Run `lark-cli skills read lark-doc` to open the skill entry point; consult local command help for exact arguments.
- Support `/docx/` and `/wiki/` URLs or an explicit document token, including doubao.com links with these paths. Route by path. `/page/` identifies an application page, not a cloud document.
- When the user asks to normalize or edit a specified document, write back directly within the requested scope. For checks, previews, or suggestions, only read the document and return a diff. Do not ask for confirmation for each update.
- Respect any specified section, selection, or block anchor. For a partial edit, read the document outline to compute heading numbers for the affected scope. For normalization of the entire document, read the complete body; process long documents section by section and track coverage.
- Fetch complete XML and style information for editing. Retain `document.content`, `reference_map`, `revision_id`, and `tips` from the same response:

  ```bash
  lark-cli docs +fetch --doc 'document URL or token' --as user --doc-format xml --detail full
  ```

## Conversion and Targeted Updates

- Process editable body text, including section headings, lists, quotes, and native document table cells. Preserve heading levels, paragraph order, list numbering, table structure, bold formatting, colors, and other styles.
- For width and spacing changes, convert only visible natural language text nodes. Preserve tags, attributes, IDs, URLs, email addresses, code, formulas, @mentions, and comments. Leave images, attachments, whiteboards, embedded spreadsheets, multidimensional tables, and synced references unchanged; do not modify the source documents of references.
- Determine adjacent characters across rich text spans within the same paragraph. Apply replacements and inserted spaces to the original spans rather than flattening them into plain text. For example:

  ```xml
  <!-- Before -->
  <p>Use <b>ＧＰＴ５</b> to write ３ articles titled《Report》。</p>
  <!-- After -->
  <p>Use <b>GPT5</b> to write 3 articles titled&lt; Report&gt; .</p>
  ```

- In each editable `<h1>`–`<h9>` heading, put the number in a blue text span at the start of the heading. For example, `<h2><span text-color="blue">1.1.</span> <b>Section heading</b></h2>`. Replace an existing section-number prefix and remove any native `seq` attribute from the updated heading so it displays a single number. Use `block_replace` to apply the change with the existing title text and inline formatting.
- Compare the text and heading number styles before and after conversion, and write only changed content. Do not convert characters globally in the serialized XML. Escape `<`, `>`, and `&` in new text according to XML rules.
- For simple inline changes, use `docs +update --command str_replace`, which matches throughout the document. First confirm that every match is within the authorized scope and does not affect protected content; do not rely on it to implicitly enforce a local scope.
- For multiple changes in one block, edits across rich text spans, or updates requiring precise scope, use `block_replace` as described in the `lark-doc` update reference. Preserve the block's structure, styles, and resource references. Include the original `reference_map` when replaying references, and use the fetched version as the write baseline.
- Do not use a full-document `overwrite` for formatting normalization. If a block cannot be updated while preserving its structure and formatting, leave it unchanged and report the unprocessed scope.

## Verify and Report

- After each round of writes, fetch the affected scope again to verify text conversion and spacing, and check structure, styles, and resource references. Confirm each affected heading displays a single correctly ordered blue number. Use the latest block IDs and version for subsequent changes. On a version conflict, read again and recompute the diff.
- Check the processed text against the main skill's rules again; no further changes should be needed. Do not write when the original text already meets the rules.
- Inspect `result`, `warnings`, and `tips` in the update response. Do not report full completion for partial success or degraded content. For authentication or permission errors, follow `lark-doc` to the `lark-shared` skill.
- On completion, briefly return the original document link and processed scope. Clearly identify any unprocessed blocks or failed verification. In check-only mode, return a diff summary without claiming the document was modified.
