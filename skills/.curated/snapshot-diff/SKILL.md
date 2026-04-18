---
name: snapshot-diff
description: Compare before/after screenshots or other raster images with `rasterdiff` to generate a diff image and use the exit status to tell whether pixels changed. Use when Codex needs to verify UI snapshot changes, pixel regressions, or whether two images are visually identical.
---

# Snapshot Diff

Run `rasterdiff` through `npx` to compare two raster images and write a diff image.

## Workflow

1. Confirm the input paths and the desired output path.
2. Check that both inputs are comparable before running the diff.
3. Run `npx rasterdiff before.png after.png output.png`.
4. Read the exit status first, then inspect `output.png` when a diff is found.
5. Treat status `2` as a real failure and keep the original stderr in the report.

## Check Before Running

- Prefer PNG for deterministic snapshot comparisons.
- Make sure both images have the same dimensions and represent the same crop or viewport.
- Do not overwrite either input file with the diff output.

## Run The Diff

Use the standard CLI shape:

```bash
npx rasterdiff before.png after.png output.png
```

If the package is not available locally, allow `npx` to fetch it:

```bash
npx --yes rasterdiff before.png after.png output.png
```

## Interpret Exit Status

- `0`: no diff
- `1`: diff found
- `2`: usage or runtime error

## Decision Rules

- Report "no diff" only on exit status `0`.
- Report "diff found" on exit status `1` and point to the generated diff image.
- Report a failure on exit status `2`; include stderr so the root cause is visible in logs or chat.
- If images differ in size or framing, fix the capture or normalization first instead of treating the result as a product regression.
