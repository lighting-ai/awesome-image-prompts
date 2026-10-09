# Auto-locate Pipeline

How to resolve the shooting location automatically instead of asking the user to
fill in the `【填写城市、景点或 GPS 坐标】` placeholder by hand.

## Why the placeholder exists

Measured facts behind the graded design:

- Social platforms strip GPS. The cover image downloaded from the Xiaohongshu CDN
  is a transcoded WebP whose only EXIF payload is `ColorSpace`,
  `PixelXDimension` and `PixelYDimension`. There is no `GPS IFD` (`0x8825`),
  no `Make` / `Model` / `DateTime`.
- Vision models cannot read EXIF at all — metadata has to be parsed outside the
  model and injected as text.
- Vision models can recognize only famous city-level landmarks. The note author
  confirms this in the comments: recognizable for the Eiffel Tower, not for an
  ordinary street or hillside.
- A model asked to both "recognize" and "render a pretty poster" at once will
  invent plausible-looking coordinates to make a nicer image.

Therefore: parse metadata outside the model, ask for recognition as a separate
text-only step, and gate the render prompt on a confidence level.

## Three stages

1. **Metadata** — read GPS from the user's original upload (camera files still
   carry it), then reverse-geocode `lat/lng` into a city name and a country.
2. **Recognition** — a text-only call: "Where was this shot? Answer with place,
   confidence (high/medium/low), and the visual evidence." Never let this step
   also generate the image.
3. **Render** — inject the resolved place or coordinates into the render prompt,
   and pick the branch by confidence.

## Stage 1 — EXIF + reverse geocoding

```ts
type ResolvedPlace = {
  source: 'exif' | 'user' | 'model' | 'none';
  label?: string;        // "Barcelona, Spain"
  lat?: number;
  lng?: number;
  confidence: 'high' | 'medium' | 'low';
};

async function resolveFromExif(file: ArrayBuffer): Promise<ResolvedPlace> {
  const exif = readExif(file);            // exifr / exif-reader
  const lat = exif?.latitude;
  const lng = exif?.longitude;
  if (lat == null || lng == null) return { source: 'none', confidence: 'low' };

  // Reverse geocoding: any provider works, cache by rounded coordinate.
  const geo = await reverseGeocode(lat, lng);
  return {
    source: 'exif',
    label: geo.city ? `${geo.city}, ${geo.country}` : geo.displayName,
    lat,
    lng,
    confidence: 'high',
  };
}
```

Round coordinates to ~4 decimals before caching; two shots 100 m apart should
reuse the same lookup.

## Stage 2 — recognition call (text only)

Send the photo alone, no poster instructions:

```text
这是一张旅行摄影作品。请判断拍摄地点，并按以下格式回答：
地点：城市 + 国家／地区（若不确定到城市，写"不确定"）
置信度：高／中／低
依据：画面中你实际看到的地标、地形或文字线索

只有在画面存在明确可辨认的城市级地标时才给出具体城市。
不要为了给出答案而猜测，不确定就写"不确定"。
```

Map the answer onto the render branch:

| Recognition result | Render branch |
| --- | --- |
| Named city + high confidence | precise branch — real geography, English place labels, red dot |
| Named region only / medium | approximate branch — terrain and coastline only, no street names |
| Unsure / low confidence | nameless branch — contour lines, graticule, compass, no toponym at all |

## Stage 3 — prompt injection

Replace the placeholder line in the prompt with one of:

```text
拍摄地点：Barcelona, Spain（41.4036, 2.1744）— 使用真实街道网格与海岸线走向
拍摄地点：意大利北部，多洛米蒂山区 — 只使用地形与等高线，不出现城市街道名
拍摄地点：无法确认 — 执行无地名地理视觉方案，不标注任何地名、坐标与比例尺
```

The third form is the important one: it is what keeps the model from stamping a
confident, wrong city onto an anonymous hillside.

## Verifying a result

- Ask the model to restate the place and confidence in the same reply, and treat
  a mismatch with stage 2 as a failed render.
- Spot-check the rendered map against a real basemap: coastline bearing, the
  relative position of labeled landmarks, and the scale bar.
- Treat every generated map as an illustration unless real GeoJSON was supplied;
  models draw map-shaped artwork, not surveyed geometry.

## When accuracy actually matters

If the product needs a geographically correct overlay, do not rely on the image
model at all: fetch real vector data (OSM/GeoJSON), render the overlay
server-side, and use the model only to composite the photo and match the paper
texture. That is the "map API" route the note author mentions.
