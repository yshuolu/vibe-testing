# UI Quality Check — The Obvious Stuff

Reference: [Apple Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines)

Here's a dirty secret about UI bugs: most of them are not subtle. They're spacing that's clearly off. Text you can barely read against the background. Buttons that are misaligned by 8 pixels. Stuff that any human would catch in 2 seconds of looking at the screen — but agents skip because they "confirmed the component renders."

When you take a screenshot, don't just check "did it render." Check if it looks *right*. Here's what to look for, with concrete numbers from Apple HIG so you have a baseline, not vibes.

## Spacing

Apple uses an **8pt grid**. Most well-designed apps do too. The universal rule: spacing should be consistent and intentional.

When you screenshot a page, check:

```
browser_screenshot
browser_evaluate: (() => {
  const el = document.querySelector('.your-element');
  const style = window.getComputedStyle(el);
  return {
    padding: style.padding,
    margin: style.margin,
    gap: style.gap
  };
})()
```

**What to catch:**
- **Inconsistent spacing** — if cards in a grid have 16px gap between them but one pair has 24px, that's a bug. Eyeball the screenshot first. If something looks "off," measure it.
- **Missing padding** — content jammed against container edges. Standard horizontal content padding is 16px. If text touches the edge of its container, flag it.
- **Uneven margins** — left margin is 16px but right margin is 12px. Or top/bottom padding differs on elements that should be symmetric.
- **Cramped interactive elements** — buttons, links, and inputs should have at least 8px of breathing room between them. If two buttons are touching or nearly touching, that's a bug.

```
browser_evaluate: (() => {
  const items = document.querySelectorAll('.card');
  const gaps = [];
  for (let i = 1; i < items.length; i++) {
    const prev = items[i-1].getBoundingClientRect();
    const curr = items[i].getBoundingClientRect();
    gaps.push(Math.round(curr.top - prev.bottom));
  }
  return { gaps, consistent: new Set(gaps).size === 1 };
})()
```

## Alignment

Misalignment is the #1 thing that makes UI look amateur. Your eye catches it instantly, even when you can't articulate what's wrong.

**What to catch:**
- **Left edges don't line up** — a heading, a paragraph, and a button below it should all share the same left edge. If the button is 4px to the right, that's a bug.
- **Centered content isn't centered** — off-center by even a few pixels is visible.
- **Baseline misalignment** — text next to an icon where the text is 2px higher than the icon. Text in a row of columns where the baselines don't match.
- **Grid breaks** — one card in a row that's a different height or width than its siblings.

```
browser_evaluate: (() => {
  const items = document.querySelectorAll('.nav-item');
  const lefts = Array.from(items).map(el => el.getBoundingClientRect().left);
  const tops = Array.from(items).map(el => el.getBoundingClientRect().top);
  return { lefts, tops, alignedLeft: new Set(lefts).size === 1, alignedTop: new Set(tops).size === 1 };
})()
```

## Contrast

If you can't read the text easily, neither can the user. Apple defers to WCAG here, and so should you.

**The numbers:**
- **Normal text: minimum 4.5:1 contrast ratio** (WCAG AA)
- **Large text (18px+ or 14px bold+): minimum 3:1**
- **Enhanced/AAA: 7:1** for normal, 4.5:1 for large

**Common offenders:**
- Light gray text on white background (#999 on #fff = 2.85:1 — fails)
- Placeholder text that's too faint to read
- Text over images without overlay/shadow
- Disabled states that are *so* faded they're invisible
- Links that look identical to surrounding text (no color or underline distinction)

Check it programmatically:

```
browser_evaluate: (() => {
  function luminance(r, g, b) {
    const [rs, gs, bs] = [r, g, b].map(c => {
      c = c / 255;
      return c <= 0.03928 ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4);
    });
    return 0.2126 * rs + 0.7152 * gs + 0.0722 * bs;
  }
  function contrastRatio(l1, l2) {
    const lighter = Math.max(l1, l2), darker = Math.min(l1, l2);
    return (lighter + 0.05) / (darker + 0.05);
  }
  function parseColor(str) {
    const m = str.match(/rgba?\((\d+),\s*(\d+),\s*(\d+)/);
    return m ? [+m[1], +m[2], +m[3]] : null;
  }

  const issues = [];
  document.querySelectorAll('p, span, a, h1, h2, h3, h4, h5, h6, label, button, li, td, th').forEach(el => {
    const style = window.getComputedStyle(el);
    const fg = parseColor(style.color);
    const bg = parseColor(style.backgroundColor);
    if (fg && bg && bg.some(c => c > 0)) {
      const ratio = contrastRatio(luminance(...fg), luminance(...bg));
      const fontSize = parseFloat(style.fontSize);
      const isBold = parseInt(style.fontWeight) >= 700;
      const isLarge = fontSize >= 18 || (fontSize >= 14 && isBold);
      const minRatio = isLarge ? 3 : 4.5;
      if (ratio < minRatio) {
        issues.push({
          text: el.textContent.slice(0, 40),
          ratio: Math.round(ratio * 100) / 100,
          required: minRatio,
          tag: el.tagName
        });
      }
    }
  });
  return issues.length ? { pass: false, issues } : { pass: true };
})()
```

**Note:** This script checks direct background colors. Text on images or gradients needs visual inspection from the screenshot — there's no shortcut. Look at the screenshot.

## Touch Targets / Click Targets

Apple's rule: **minimum 44x44 points for any interactive element.** On the web this translates to roughly 44x44 CSS pixels.

Tiny buttons and links are a real usability problem. Check it:

```
browser_evaluate: (() => {
  const issues = [];
  document.querySelectorAll('a, button, input, select, [role="button"], [onclick]').forEach(el => {
    const rect = el.getBoundingClientRect();
    if (rect.width > 0 && rect.height > 0 && (rect.width < 44 || rect.height < 44)) {
      issues.push({
        tag: el.tagName,
        text: (el.textContent || el.getAttribute('aria-label') || '').slice(0, 30),
        width: Math.round(rect.width),
        height: Math.round(rect.height)
      });
    }
  });
  return issues.length ? { pass: false, issues } : { pass: true };
})()
```

## Typography

**Quick checks:**
- **Body text should be at least 16-17px.** Anything smaller is straining.
- **Clear hierarchy** — headings are noticeably larger/bolder than body text. If you squint at the screenshot and can't tell headings from paragraphs, the hierarchy is broken.
- **Line length** — comfortable reading is 45-80 characters per line. If text stretches full-width on a 1440px monitor, it's too wide.
- **Line height** — body text should have at least 1.4-1.6x line-height. If lines of text are cramped together, flag it.

```
browser_evaluate: (() => {
  const body = document.querySelector('p, .content, main p, article p');
  if (!body) return { note: 'no body text found' };
  const style = window.getComputedStyle(body);
  const fontSize = parseFloat(style.fontSize);
  const lineHeight = parseFloat(style.lineHeight);
  const ratio = lineHeight / fontSize;
  const width = body.getBoundingClientRect().width;
  const charsPerLine = Math.round(width / (fontSize * 0.5));
  return {
    fontSize: fontSize,
    lineHeightRatio: Math.round(ratio * 100) / 100,
    approxCharsPerLine: charsPerLine,
    issues: [
      fontSize < 16 ? 'body text too small (< 16px)' : null,
      ratio < 1.4 ? 'line height too tight (< 1.4x)' : null,
      charsPerLine > 80 ? 'lines too wide (> 80 chars)' : null
    ].filter(Boolean)
  };
})()
```

## The Eyeball Test

All the scripts above are helpers, but the most important tool is **looking at the screenshot carefully.** When you take a screenshot, spend a moment and ask:

1. Does everything line up? Follow the left edge of the content — do all elements share it?
2. Is the spacing even? Same gaps between same kinds of elements?
3. Can you read all the text easily? Nothing faint, nothing tiny?
4. Are interactive things obviously interactive? Buttons look like buttons?
5. Is there visual breathing room, or does it feel cramped?

If something looks wrong to you, it looks wrong to the user. Trust your eyes, then verify with the measurement scripts.
