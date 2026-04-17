# Add "Click date pill → Google Calendar event" to cityhenge

A reproducible recipe for re-applying the clickable-pill / Google Calendar
feature to a fresh copy of `index.html`. Three small surgical edits to the
existing file — no new dependencies, no build step.

## What it does

Each upcoming date pill on a city card becomes a link. Clicking opens a
Google Calendar event template (new tab) prefilled with:

- **Title:** `{HengeName}: catch the sun lining up with {City}`
- **Time:** starts X minutes before sunset, ends at sunset (city's timezone)
- **X is latitude-aware:** `~20 / cos(lat)` minutes — the time for the sun to
  drop the last ~5° to the horizon, since `dh/dt ≈ 15°/hr · cos(lat)` near
  sunset. Miami → 22 min, Manhattan → 26, Montreal → 29.
- **Location:** `{City}, {Country}`
- **Description:** local sunset time, lead-time explainer, grid offset, target
  azimuth, the existing city note, link back to the cityhenge page.

Past dates stay as plain divs (no link). The `+ add to cal` affordance fades
in only on hover, so the resting design is unchanged.

## Edit 1 — CSS (in the `<style>` block)

Find the `.date-pill` rule and replace the small block of pill styles with:

```css
.date-pill { padding: 8px 12px; border: 1px solid #e0e0e0; text-align: center; background: #fafafa; position: relative; }
.date-pill.upcoming { border-color: #111; background: #fff; }
a.date-pill { text-decoration: none; color: inherit; display: block; cursor: pointer; transition: background 0.12s ease, border-color 0.12s ease, transform 0.12s ease, box-shadow 0.12s ease; }
a.date-pill:hover { border-color: var(--orange); background: #fff8ee; transform: translateY(-1px); box-shadow: 0 2px 0 0 var(--orange); }
a.date-pill:hover .pill-cta { color: var(--orange); }
a.date-pill:focus-visible { outline: 2px solid var(--orange); outline-offset: 2px; }
.pill-date { font-size: 14px; font-weight: 600; font-family: var(--mono); }
.pill-days { font-size: 11px; color: #888; margin-top: 2px; font-family: var(--mono); }
.date-pill.upcoming .pill-days { color: var(--orange); }
.pill-cta { font-size: 8px; color: #bbb; margin-top: 3px; font-family: var(--mono); text-transform: uppercase; letter-spacing: 0.06em; max-height: 0; opacity: 0; overflow: hidden; transition: max-height 0.15s ease, opacity 0.15s ease, margin-top 0.15s ease; }
a.date-pill:hover .pill-cta, a.date-pill:focus-visible .pill-cta { max-height: 14px; opacity: 1; }
```

## Edit 2 — JS helpers (in the `<script>` block)

Insert this block immediately after `function fmtLong(...)` (before `drawGrid`):

```js
// NOAA solar algorithm — UTC instant of sunset for given lat/lng on a calendar day.
function sunsetUTC(lat,lng,date){
  const year=date.getFullYear(),month=date.getMonth(),day=date.getDate();
  const startUTC=Date.UTC(year,0,1);
  const dayOfYear=Math.floor((Date.UTC(year,month,day)-startUTC)/86400000)+1;
  const gamma=2*Math.PI/365*(dayOfYear-1);
  const decl=0.006918-0.399912*Math.cos(gamma)+0.070257*Math.sin(gamma)
            -0.006758*Math.cos(2*gamma)+0.000907*Math.sin(2*gamma)
            -0.002697*Math.cos(3*gamma)+0.00148*Math.sin(3*gamma);
  const eqtime=229.18*(0.000075+0.001868*Math.cos(gamma)-0.032077*Math.sin(gamma)
                      -0.014615*Math.cos(2*gamma)-0.040849*Math.sin(2*gamma));
  const latRad=lat*Math.PI/180;
  const haCos=(Math.cos(90.833*Math.PI/180)-Math.sin(latRad)*Math.sin(decl))/(Math.cos(latRad)*Math.cos(decl));
  if(Math.abs(haCos)>1)return null;
  const haDeg=Math.acos(haCos)*180/Math.PI;
  const sunsetMin=720-4*lng-eqtime+4*haDeg;
  return new Date(Date.UTC(year,month,day)+sunsetMin*60000);
}

// Latitude-aware viewing window: ~5° / (15·cos(lat)) hours = 20/cos(lat) minutes.
function leadMinutes(lat){return Math.round(20/Math.cos(Math.abs(lat)*Math.PI/180));}

function gcalStamp(d){
  const p=n=>String(n).padStart(2,'0');
  return `${d.getUTCFullYear()}${p(d.getUTCMonth()+1)}${p(d.getUTCDate())}T${p(d.getUTCHours())}${p(d.getUTCMinutes())}00Z`;
}
function fmtLocalTime(utcDate,tz){
  return utcDate.toLocaleTimeString('en-US',{timeZone:tz,hour:'numeric',minute:'2-digit'});
}
function gcalUrl(city,hengeDate){
  const sunset=sunsetUTC(city.lat,city.lng,hengeDate);
  if(!sunset)return null;
  const lead=leadMinutes(city.lat);
  const start=new Date(sunset.getTime()-lead*60000);
  const sunsetLocal=fmtLocalTime(sunset,city.tz);
  const tgtCompass=(city.hem==='N'?(city.offsetDir==='CW'?270+city.offset:270-city.offset):(city.offsetDir==='CW'?270-city.offset:270+city.offset)).toFixed(1);
  const offsetSign=city.offset===0?'':(city.hem==='S'?(city.offsetDir==='CW'?'-':'+'):(city.offsetDir==='CW'?'+':'-'));
  const title=`${city.henge}: catch the sun lining up with ${city.name}`;
  const details=[
    `Sunset in ${city.name}: ${sunsetLocal}`,
    `Recommended arrival: ${lead} minutes before sunset`,
    ``,
    `This event spans the latitude-aware viewing window — from when the sun drops into the "frame" of the grid down to the horizon. At ${Math.abs(city.lat).toFixed(1)}° latitude, that's about ${lead} minutes.`,
    ``,
    `Grid offset: ${offsetSign}${city.offset}° from cardinal`,
    `Target sun azimuth: ${tgtCompass}°`,
    ``,
    city.note + '.',
    ``,
    `More cityhenges → https://mikecardarelli.github.io/city-henge/`
  ].join('\n');
  const params=new URLSearchParams({
    action:'TEMPLATE',
    text:title,
    dates:`${gcalStamp(start)}/${gcalStamp(sunset)}`,
    details,
    location:`${city.name}, ${city.country}`,
    ctz:city.tz
  });
  return `https://calendar.google.com/calendar/render?${params.toString()}`;
}
```

## Edit 3 — pill rendering (inside `renderCards()`)

Replace the existing `pillsHtml = city.datesThis.map(...)` block with:

```js
const pillsHtml=city.datesThis.map(d=>{
  const days=daysUntil(d);
  const up=days>=0;
  const dayStr=days===0?'today':days===1?'tomorrow':`in ${days} days`;
  if(up){
    const url=gcalUrl(city,d);
    const sunset=sunsetUTC(city.lat,city.lng,d);
    const sunsetLocal=sunset?fmtLocalTime(sunset,city.tz):'';
    const tip=sunset?`Add to Google Calendar — sunset ${sunsetLocal} (${leadMinutes(city.lat)} min before)`:'Add to Google Calendar';
    return `<a href="${url}" target="_blank" rel="noopener" class="date-pill upcoming" title="${tip}" aria-label="${tip}">
      <div class="pill-date">${fmtLong(d)}</div>
      <div class="pill-days">${dayStr}</div>
      <div class="pill-cta">+ Add to cal</div>
    </a>`;
  }
  return `<div class="date-pill">
    <div class="pill-date">${fmtLong(d)}</div>
    <div class="pill-days">passed</div>
  </div>`;
}).join('');
```

## Verification

Spot-check the math against a published source (e.g.
[timeanddate.com sunrise/sunset](https://www.timeanddate.com/sun/)):

| City      | Date       | Expected sunset | NOAA-computed |
|-----------|------------|-----------------|---------------|
| Manhattan | 2026-05-29 | 8:18 PM EDT     | 8:17 PM       |
| Savannah  | 2026-03-19 | ~7:34 PM EDT    | 7:34 PM       |
| Melbourne | 2026-10-03 | ~6:25 PM AEST   | 6:24 PM       |

Within ~1 minute, which is well inside atmospheric-refraction tolerance.

Local preview:

```bash
python3 -m http.server 8765
open http://localhost:8765/index.html
```

Hover a date pill → "+ ADD TO CAL" fades in. Click → Google Calendar opens
with the prefilled event in a new tab. Past dates remain plain (no link).

## Why these specific choices

- **Google Calendar TEMPLATE URL** (vs. .ics download or full API): no auth,
  no API key, opens in a new tab where the user clicks Save. Works for
  everyone with a Google account in one click.
- **Latitude-aware lead time** (vs. fixed 30 min): physically meaningful — at
  higher latitudes the sun moves more slowly along the horizon (`dh/dt ∝
  cos(lat)`), so the "framed" window is genuinely longer. The event runs from
  ~5° altitude down to the horizon.
- **NOAA solar algorithm** (vs. the existing simplified azimuth math used for
  the henge-date fallback): sunset *time* needs to be accurate to the minute
  for a calendar event. The simplified formula is fine for picking a *date*
  but not a *time*.
- **Hover-only `+ add to cal`** (vs. always-visible): keeps the resting layout
  identical to Mike's original design — no extra line, no taller pills.
