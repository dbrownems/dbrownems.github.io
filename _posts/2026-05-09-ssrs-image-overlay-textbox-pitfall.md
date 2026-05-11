---
title: Stop Trying to Overlay Textboxes on Form Images in SSRS / Report Builder
date: 2026-05-09 16:45:00 -0500
categories: [SQL Server, SSRS]
tags: [ssrs, report-builder, rdl, power-bi-report-server, rendering, ai-pairing]
---

You have a beautiful PDF invoice template from your legacy system. You drop it into
a Report Builder report as an image, drag a few textboxes on top to fill in the
customer name and totals, and hit Run. The textboxes are nowhere near where
you put them. They've slid down the page, *underneath* the image. You stare
at the screen.

If this is you, welcome — you've just discovered one of SSRS's oldest and most
consistently misunderstood quirks. This pattern looks like it should "just
work," and on a printed page it kind of does. Everywhere else, it doesn't.
This post explains why, and gives you the only two approaches that actually
work in practice.

## Why "image + overlay textboxes" is so seductive

The mental model is irresistible:

1. The designer hands you `invoice_template.pdf`.
2. You export it to PNG.
3. You drag it onto the body of a new report.
4. You drop textboxes wherever fields go: customer number, invoice number,
   amount due.
5. Ship it.

Five minutes of work. The picture stays pixel-perfect. The textboxes carry
the data. What's not to love?

## Why it doesn't work

SSRS has two fundamentally different rendering pipelines:

- **Hard-page-break renderers** — PDF, TIFF, physical print. These honor the
  absolute (Top, Left) coordinates you set in the designer. Items can overlap
  freely; the rendered page is a faithful Z-ordered composition of everything
  you laid out.
- **Soft-page-break renderers** — HTML (the Report Builder preview, the
  Power BI Report Server web viewer, SharePoint), Word, Excel. These flatten
  every container's children into an HTML `<table>` keyed on each item's
  Top/Left. **Two items whose bounding boxes overlap cannot share a row.**
  One of them gets pushed into its own row, below the other.

In other words: a textbox that visually sits on top of an image becomes a
textbox that *renders below the image* — every time, in every browser-based
viewer.

This is not a bug, it's documented behavior. Stack Overflow has been telling
people this since at least 2014:

> *Page overlapping is supported in hard page break formats, such as PDF, or
> physical print of a report. Essentially, any other format (such as HTML or
> the Report Builder tool) will move your items around the page, but if you
> export that same (non-overlapped) report to PDF, your items will be
> displayed as you intended, with overlapping items.*
>
> — RideTheWalrus, [Stack Overflow](https://stackoverflow.com/questions/26545408)

## The escape hatches that *almost* work

Once you've hit the bug, you'll go down this rabbit hole. I went down all of
these. Save yourself the time:

| Approach | Design canvas | Print Preview | HTML viewer | PDF |
| --- | --- | --- | --- | --- |
| `<Image>` + sibling `<Textbox>` (the obvious thing) | ✓ | ✗ reflows | ✗ reflows | ✓ |
| `<BackgroundImage>` on the parent Rectangle | wrong DPI | wrong DPI | ✓ | ✓ |
| `<BackgroundImage>` on the Body | wrong DPI | wrong DPI | ✓ | ✓ |
| `<BackgroundImage>` on the Page | blank | blank | ✓ | ✓ |

`BackgroundImage` is interesting because it sidesteps the flattening rule —
the image is not a sibling of your textboxes, it's a paint instruction on the
container. The runtime renderers respect it. But Report Builder's design
canvas assumes a different effective DPI for the background bitmap than the
runtime does, so what looks aligned in the designer is misaligned at runtime,
or vice versa. None of the four `BackgroundImage` placements aligns in all
four contexts.

## The two approaches that actually work

### Option 1 — "PDF only": ship the image-overlay report and never look at it in a browser

If your only delivery channel is PDF (you email it, you print it, you upload
it to a doc-management system), then the obvious approach is *fine*. Drop the
image, drop the textboxes on top, set the report's default rendering format
to PDF, and move on.

Two firm rules:

- **Never render in Report Builder's preview tab.** It will always look
  wrong. Train your team to "Run" → "Export → PDF" and inspect the PDF.
- **Never render in the web portal viewer.** If users will open the report
  in a browser, this option is off the table.

> This is honestly the right answer for a surprising number of reports —
> payroll, invoicing, statutory forms — where PDF *is* the deliverable.
{: .prompt-tip }

### Option 2 — "Web-friendly": abandon the image, rebuild the form in RDL

If you need the report to look right in the browser, in Report Builder's
preview, in Print Preview, *and* in PDF, you have one option: throw away the
image and recreate the form's layout as a tree of `<Rectangle>` and
`<Textbox>` elements.

The trick that makes this work is *containment*:

- Each "card" or "cell" of the form is its own `<Rectangle>` with a border
  and background color.
- The label textbox goes **inside** that rectangle.
- The data textbox also goes **inside** that rectangle, vertically below
  the label.

When the HTML renderer flattens children into rows, it only flattens *within
one container at a time*. Inside one rectangle, label-on-top-of-value
flattens to two stacked rows — exactly what you want. At the page level, the
rectangle is one atomic box; nothing reflows.

The minimal RDL pattern for one cell:

```xml
<Rectangle Name="Cell_ClientNumber">
  <ReportItems>
    <Textbox Name="LblClientNumber">
      <!-- label, Top=0.02in -->
    </Textbox>
    <Textbox Name="ValBillToCode">
      <!-- value, Top=0.12in, =Fields!BillToCode.Value -->
    </Textbox>
  </ReportItems>
  <Top>0.396in</Top>
  <Left>3.540in</Left>
  <Width>1.040in</Width>
  <Height>0.290in</Height>
  <Style>
    <Border><Color>#1e3c8c</Color><Width>0.75pt</Width></Border>
  </Style>
</Rectangle>
```

This is, not coincidentally, exactly how Microsoft's own canonical
`Invoice.rdl` sample is built. Open it up — there is not a single image
element. Every line, every band, every cell is a rectangle or a textbox.

## "But rebuilding the form in RDL is so tedious"

Yes. It is. Counting borders, measuring widths in inches, getting the
typography right, recreating colored bands and wordmarks — it's hours of
fiddly work, and Report Builder is not a fun design tool.

> This is exactly the kind of work where an AI coding agent earns its keep.
> Hand it your PDF/PNG of the form along with a description ("invoice
> template, six-cell header grid, navy blue 0.75pt borders, blue band for
> the lower section"), and ask it to emit the RDL. You'll get a first cut
> in seconds, iterate on the geometry in a couple of rounds of feedback,
> and end up with a layout that renders identically in the design canvas,
> Print Preview, the web viewer, and PDF. That's the workflow I used to
> produce the worked example for this post — about twenty minutes of
> conversation versus a half-day of dragging pixels in Report Builder.
{: .prompt-info }

## TL;DR

- The "drop an image, overlay textboxes" pattern only renders correctly in
  PDF and on physical print. Browsers and Report Builder's preview will
  always reflow your textboxes below the image.
- If PDF is your only output, embrace it: ship the image-overlay report,
  render to PDF, and don't open it in a browser.
- If you need any browser-based or in-tool preview to look right, **rebuild
  the form's layout in pure RDL** (rectangles and textboxes, with data
  textboxes nested *inside* their parent label rectangles). It's tedious —
  but it's the only thing that works in every renderer, and it's a perfect
  job to delegate to an AI agent.
