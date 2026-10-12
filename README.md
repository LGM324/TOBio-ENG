https://tobio.pages.dev/

# TOBio (ENG)

A website that recommends soil microbes suited to a farm's soil composition and climate. Built for the Korean Agricultural and Rural Startup Competition (농업농촌창업경진대회).

> English translation of [LGM324/TOBio](https://github.com/LGM324/TOBio). Only the documentation is translated; the code and the app's screens are the same as the original and remain in Korean. Values shown in quotes in Korean (e.g. `"실측값"`) are literal strings used in the code and API responses.

## What is implemented

Given a road-name or lot-number address and a crop, the site first checks whether the address is a registered farmland parcel. It then fetches that parcel's soil and weather data from public APIs in real time. Based on that data, it searches 1,764 A-grade research papers for relevant evidence, and an LLM chooses the microbes to recommend. The result shows a plain-language explanation that farmers can understand right away, plus real product and seller information. The academic rationale with paper citations is kept separately behind a "더보기" (Show more) button.

## Overall flow

```
[Browser: a_recommend.html]
  1. The user enters an address (road-name or lot-number) and a crop
  2. Kakao address search API (v2/local/search/address.json) converts
     address → latitude/longitude + legal-dong code (b_code)
  3. The lat/lon is matched to the nearest of 215 national agricultural
     weather stations (js/agriStations.js)
  4. Lat/lon + legal-dong code + station code + date/time are sent to the
     backend (Render)
        ↓
[Backend: backend/server.js, Render — /api/getMergedData]
  5. Agricultural weather API (10-minute detailed observations) returns air
     temperature, precipitation, solar radiation, soil temperature, and soil
     moisture
  6. Soil data is looked up in 3 steps, in order of precision:
     a) Farm-map soil test API by coordinates → measured values for that
        parcel (used if available)
     b) Otherwise, from the legal-dong area statistics by grade, find the
        grade with the largest field (crop-growing) area in that
        neighborhood and use its representative (median) value as an estimate
     c) If that is also missing, use fixed national averages
  7. Separately, the farm-map lookup service (coordinate-based farm-map
     detail) checks whether a farmland parcel registered from satellite/aerial
     imagery exists at those coordinates (isFarmland: true/false/null).
     Unlike the soil test, this checks "whether the farmland itself exists",
     not "whether a test record exists", so it distinguishes farmland with no
     test record from land that is not farmland
  8. The results are merged and returned as JSON (the soil data source is
     shown in the soilDataSource field as "실측값" (measured) /
     "지역 추정값" (regional estimate) / "전국 평균값" (national average))
        ↓
[Browser]
  9. If isFarmland === false, no recommendation is made and a "not farmland"
     error is shown. If null (the farm-map lookup itself failed), it is not
     blocked and continues
  10. The response JSON is saved in localStorage and the page moves to
      a-1_recommend-result.html
        ↓
[Backend: backend/server.js, Render — /api/recommendMicrobe]
  11. Soil/weather values are converted, using grade ranges, into a
      qualitative English query such as "acidic/neutral", "low/medium/high"
      (LLMs and embedding models understand such descriptions better than
      raw numbers)
  12. Voyage AI (voyage-multilingual-2) embeds the query, and cosine
      similarity retrieves the 8 most relevant chunks from a pre-embedded
      index of 49,003 paper chunks (1,764 A-grade papers)
  13. Gemini (gemini-3.1-flash-lite, free tier) receives the retrieved paper
      excerpts + the environmental data (raw numbers + English qualitative
      descriptions) and produces the scientific names of the recommended
      microbes along with two separate explanations:
      a) explanation — a plain-language description of this field's
         condition and the benefits of the recommended microbes, without
         jargon or paper citations, so farmers can understand it right away
      b) scientificEvidence — the academic rationale for the same
         recommendation, written technically with paper citations ([1], [2],
         etc.) (collapsed behind "더보기" (Show more) on screen)
  14. The recommended scientific names are matched against
      backend/microbe_disclosure.csv (the original Ministry of Agriculture,
      Food and Rural Affairs disclosure list of microbial products: brand
      name, company, price, contact, factory address) to attach a list of
      sellers where the product can actually be bought. To match even when
      names are written differently (common typos, "A, B" multi-species
      labels, strain designations, reclassified names such as
      Lactobacillus→Lactiplantibacillus and Bacillus megaterium→Priestia
      megaterium, or genus-level recommendations like "Bacillus spp."), it
      applies, step by step: normalization + a typo/synonym table + matching
      on the species epithet alone + merged matching of products in the same
      genus
  15. When the free API quota is exceeded (HTTP 429), instead of an error it
      shows only the list of retrieved papers and includes a
      quotaExceeded: true flag in the response
        ↓
[Browser: a-1_recommend-result.html]
  16. By default, renders the recommended microbe cards (price range / number
      of products / pesticide or fertilizer registration / actual seller
      list: company, product, price, contact) + explanation (plain language).
      scientificEvidence + the reference paper list are collapsed inside
      "📄 더보기: 논문 근거로 살펴보기" (Show more: see the paper evidence) and
      only expand when clicked
```

## Address handling

When a road-name or lot-number address is typed into the input box, the Kakao address search API returns both coordinates and the legal-dong code. This API also matches farmland lot numbers that have no road-name address.

## Public APIs used

| Data | API | Lookup key | Notes |
|---|---|---|---|
| Air temperature, precipitation, solar radiation, soil temperature, soil moisture | Rural Development Administration, National Institute of Agricultural Sciences: agricultural weather 10-minute detailed observations by date (`getWeatherTenMinList4`) | Station code (obsr_Spot_Cd) | XML responses only. Same-day data may be slightly delayed |
| Soil pH, organic matter, available phosphate, available silicate, electrical conductivity (measured) | Korea Agency of Education, Promotion and Information Service in Food, Agriculture, Forestry and Fisheries: farm-map-based soil test lookup service (`getCoordinateBasedSoilAnalsInfo`) | Coordinates (EPSG:5179) | Returns values only for coordinates inside a registered farm-map parcel |
| Soil pH, organic matter, available phosphate, potassium, calcium, magnesium, available silicate (legal-dong estimate) | Rural Development Administration, National Institute of Agricultural Sciences: farmland chemical property statistics V2 (`getFarmExamPhInfo` and 6 other operations) | Legal-dong code (10 digits) | Area (ha) statistics by grade. Used only when the measured API above returns NODATA |
| Whether the land is registered farmland (farmland/non-farmland), land-use label (field, paddy, etc.) | Korea Agency of Education, Promotion and Information Service in Food, Agriculture, Forestry and Fisheries: farm-map lookup service, coordinate-based farm-map detail (`getCoordinateBasedFarmmapInfo`) | Coordinates (EPSG:5179) | Based on satellite/aerial image interpretation, so it checks whether the farmland itself exists regardless of soil test records. Requires a separate application and approval for this API on data.go.kr |

## Microbe recommendation (RAG + LLM) index

- The paper chunk/vector index (`chunks.jsonl` 167 MB, `vectors.f32` 200 MB) is hosted as a GitHub Release (`paper-index-a-grade-v1`) and downloaded when the backend first starts (`backend/data/` is not committed to git).
- Memory use is minimized so it runs on Render's free instance (512 MB limit):
  - The chunk text (167 MB) is not loaded fully into memory. Only per-line byte offsets (two Int32Arrays) are kept, and only the top few search results are read from the file on demand
  - The vectors (200 MB, Float32Array) must be in memory for a full cosine-similarity scan, but only one copy is allocated and no extra copies are made
  - A search does not build and sort an array of `{idx,score}` objects for all 49,003 chunks. It keeps only the top k (8) via insertion, so extra memory per request is close to zero
  - Measured RSS at startup is about 240 – 260 MB (well within the 512 MB limit)
- To rebuild the index from scratch, run chunking + Voyage embedding with `backend/scripts/buildPaperIndex.js`.

## Folder structure

```
.
├── a_recommend.html         # Address + crop input screen
├── a-1_recommend-result.html  # Recommendation result screen (live API)
├── b_spray.html
├── b-1_spray-result.html
├── index.html
├── css/style.css
├── js/
│   ├── app.js                # Renders result screens (a-1 uses the live API, b/b-1 use placeholder data)
│   ├── regions.js
│   └── agriStations.js       # Auto-generated list of agricultural weather station coordinates
└── backend/                  # Express server deployed to Render
    ├── server.js
    ├── microbe_disclosure.csv  # Original microbial product disclosure list (real sellers/prices/contacts)
    ├── package.json
    ├── render.yaml
    ├── .env.example
    └── scripts/
        ├── buildAgriStations.js  # Regenerates agriStations.js
        └── buildPaperIndex.js    # Paper chunking + Voyage embedding index builder
```

## Deployment

- Frontend: GitHub Pages (repository root, static files)
- Backend: Render (Web Service, Root Directory `backend`, Build `npm install`, Start `npm start`)
- Backend environment variables:
  - `PUBLIC_DATA_API_KEY`: Public Data Portal service key
  - `ALLOWED_ORIGINS`: domains allowed by CORS
  - `VOYAGE_API_KEY`: Voyage AI embeddings
  - `GEMINI_API_KEY`: Gemini recommendation explanations, free tier

## Remaining work

- When the soil data comes back as "전국 평균값" (national average), there is neither measured nor statistical data for that area. Users need to be told this.
- The b/b-1 (spray timing) screens still run only on placeholder (fake) data.

## Related docs

- [SUPABASE_SETUP.md](SUPABASE_SETUP.md): login and farming-record setup for the static frontend
- [microbe-ai-landing/AUTH_SETUP.md](microbe-ai-landing/AUTH_SETUP.md): login and records setup for the React app
