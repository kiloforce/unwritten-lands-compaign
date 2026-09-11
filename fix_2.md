Fixing both. One correction to my earlier report first: `ta()` is used by the Legacy page too, so it's **three** pages affected, not two. 

Replacing 5 lines with 5 lines in 


```

```
Now verifying all three pages save, and that the export picks up People/Places.  

Ran Playwright code
Completed with input: const errs = [];
page.on('pageerror', e => errs.push(e.message));
await page.evaluate(() => localStorage.removeItem('unwritten_lands_character_v2'));
await page.reload();

// 05 Who I Am (ta -> traits)
await page.click('nav button[data-p="traits"]');
await page.fill('main textarea >> nth=0', 'Vengeful');
await page.fill('main textarea >> nth=1', 'Lost her village');

// 02 Background (ta2 -> events)
await page.click('nav button[data-p="bg"]');
await page.click('main button.primary');
await page.fill('main textarea >> nth=0', 'The Burning');
await page.fill('main textarea >> nth=1', 'Rolled 47');
await page.fill('main textarea >> nth=7', 'Who set the fire');

// 10 Legacy (ta -> legacy)
await page.click('nav button[data-p="legacy"]');
await page.fill('main textarea >> nth=0', '12');
await page.fill('main textarea >> nth=2', 'Held the bridge');

// 03 People (generic keys vs export)
await page.click('nav button[data-p="people"]');
await page.click('main button.primary');
await page.fill('main textarea >> nth=0', 'Mara');
await page.fill('main textarea >> nth=2', 'Revenge');

// navigate away and back to prove it round-trips from storage, not just the DOM
await page.click('nav button[data-p="id"]');
await page.click('nav button[data-p="bg"]');
const titleBox = await page.inputValue('main textarea >> nth=0');

const s = await page.evaluate(() => JSON.parse(localStorage.getItem('unwritten_lands_character_v2')));
return {
  pageErrors: errs,
  traits: s.traits,
  event0: s.events[0],
  legacy: s.legacy,
  people0: s.people[0],
  titleAfterRoundTrip: titleBox
};

Both pages round-trip cleanly with zero errors. Last check — capturing the actual workbook XML to confirm People/Places now export. 

Ran Playwright code
Completed with input: const xml = await page.evaluate(async () => {
  let captured = null;
  const realCreate = URL.createObjectURL, realRevoke = URL.revokeObjectURL;
  URL.createObjectURL = b => { captured = b; return 'blob:stub'; };
  URL.revokeObjectURL = () => {};
  HTMLAnchorElement.prototype.click = function(){};   // don't actually download
  downloadSpreadsheet();
  const text = await captured.text();
  URL.createObjectURL = realCreate; URL.revokeObjectURL = realRevoke;
  return text;
});
const grab = name => (xml.match(new RegExp(`<Worksheet ss:Name="${name}">.*?</Worksheet>`))||[''])[0];
return {
  peopleSheet: grab('People'),
  placesSheet: grab('Places'),
  traitsHasVengeful: xml.includes('Vengeful'),
  eventsHasBurning: xml.includes('The Burning'),
  legacyHasBridge: xml.includes('Held the bridge')
};

Ran Playwright code
Completed with input: await page.evaluate(() => localStorage.removeItem('unwritten_lands_character_v2'));
await page.reload();   // discard the click() stub my test installed
return await page.evaluate(() => ({ storage: localStorage.getItem('unwritten_lands_character_v2'), clickIsNative: HTMLAnchorElement.prototype.click.toString().includes('native code') }));

Both fixed and verified in the browser — all three affected pages now save, and People/Places data reaches the spreadsheet.

Three edits to `index.html`:

- `index.html:66` and `index.html:119` — quoted the interpolated key so the generated handler is `d.traits['dark']=…` and `d.events[0]['result']=…` instead of bare identifiers. This matches the `['${k}']` form already used in `generic()` at `index.html:123`. One fix covers pages 02, 05 and 10, since `ta()` serves both Who I Am and Legacy.
- `index.html:171-172` — the export now reads `whattheywant`, `whatiwant`, `whatiremember`, `peopleconnected`, which is what `generic()` actually derives from the column labels. I fixed the reader rather than the writer so existing journals keep their data.

Verified against a live page: zero `pageerror`s while typing, values round-trip through localStorage after navigating away and back, no more junk `""` key on events, and the captured workbook XML puts `Revenge` under the "What They Want" column. I cleared my test journal afterward.

One consequence worth knowing: anything typed into those three pages before this fix was never written to storage, so there's nothing to recover. Old background events may also carry an inert `""` key from the `title` misfire — harmless, nothing reads or exports it.

Still open from the review, if you want them: the unvalidated import that can brick a journal at `index.html:193`, the two unescaped interpolations at `index.html:111` and `index.html:131`, the immediate `revokeObjectURL`, and the duplicate-ID-after-delete numbering.

Made changes.
