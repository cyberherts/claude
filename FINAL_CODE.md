# FINAL FIXED CODE - Copy Everything Below This Line

```javascript
// === Code.gs — FIXED: 10-digit searches (with or without spaces) ===
//
// What it does
// - 10-digit input (even with spaces/dashes): exact commodity (/api/v2/commodities/{code})
//   + THREE-LAYER FALLBACK:
//     1) Search API for exact match
//     2) Heading structure (catches non-declarable codes that 8-digit search finds)
// - Numeric prefix (2/4/6/8 digits): use chapters/headings → children; fallback to search+filter.
// - Odd-length numeric (3/5/7/9): search+filter to 10-digit starting with the prefix.
// - Text input: search (v2 then v1), deduped and sorted (10-digit first).
// - Returns: { payload: [{ code, description }] }.
//
// Logging: writes [email, timestamp, keyword] to the first sheet.

const LOG_SPREADSHEET_ID = '1k9qfmzykhCT6tIi0brl-floJiDeXsWi5wabxhT28imw';
const LOG_TIMEZONE = 'Europe/London';

// HMRC API endpoints
const HMRC_V2_BASE = 'https://www.trade-tariff.service.gov.uk/api/v2';
const HMRC_SEARCH_V2 = HMRC_V2_BASE + '/search?q=';
const HMRC_COMMODITY_V2 = HMRC_V2_BASE + '/commodities/';
const HMRC_CHAPTER_V2 = HMRC_V2_BASE + '/chapters/';   // e.g. /chapters/22
const HMRC_HEADING_V2 = HMRC_V2_BASE + '/headings/';   // e.g. /headings/2204

// v1 search as a safety net (it's simpler to parse on some inputs)
const HMRC_SEARCH_V1 = 'https://www.trade-tariff.service.gov.uk/api/v1/search?q=';

function doGet() {
  return HtmlService.createHtmlOutputFromFile('Index')
    .setTitle('HMRC HS Code Search');
}

/**
 * Main entry from client.
 * @param {string} p_search - user input (digits or keywords)
 * @returns {{payload: Array<{code:string, description:string}>}}
 */
function searchHS(p_search) {
  if (typeof p_search !== 'string' || !p_search.trim()) {
    throw new Error('Missing search term.');
  }

  var term = p_search.trim();
  var digits = term.replace(/\D/g, ''); // <- critical: use this for logic
  var out = [];

  try {
    if (digits.length >= 10) {
      // Treat anything with 10+ digits as a 10-digit code (first 10)
      var code10 = digits.slice(0, 10);
      out = fetchCommodityExact_(code10);

      // Fallback 1: if commodity endpoint doesn't return, try search and exact-match filter
      if (out.length === 0) {
        var raw10 = fetchSearchRawAll_(code10);
        out = filterExact10_(raw10, code10);
      }

      // Fallback 2: if search doesn't find it, try heading structure (like 8-digit search)
      // This catches non-declarable codes that exist in the hierarchy but not in commodity endpoint
      if (out.length === 0) {
        var prefix8 = code10.slice(0, 8);
        var rawHeading = fetchByNumericPrefix_(prefix8);
        out = filterExact10_(rawHeading, code10);
      }

    } else if ([2, 4, 6, 8].includes(digits.length)) {
      // Numeric prefix (2/4/6/8)
      out = fetchByNumericPrefix_(digits);
      if (out.length === 0) {
        var rawP = fetchSearchRawAll_(digits);
        out = filterPrefixTo10_(rawP, digits);
      }

    } else if (digits.length > 0) {
      // Odd-length numeric (3/5/7/9) → search+filter to 10-digit children starting with prefix
      var rawOdd = fetchSearchRawAll_(digits);
      out = filterPrefixTo10_(rawOdd, digits);

    } else {
      // Keyword mode
      var rawText = fetchSearchRawAll_(term);
      out = mapToPayload_(rawText);
    }

    // Log attempt (best-effort)
    try { logSearch_(term); } catch (e) { Logger.log('Logging failed: ' + e); }

    return { payload: out };
  } catch (err) {
    throw new Error('Execution error: ' + err.message);
  }
}

// ---------------- Numeric prefix strategy ----------------

/**
 * For 2/4/6/8-digit numeric prefixes, try the most structured sources first.
 * - 2 digits → /chapters/{cc} then filter to 10-digit children starting with {cc}
 * - 4 digits → /headings/{hhhh} then filter to 10-digit children starting with {hhhh}
 * - 6/8 digits → fetch heading (first 4) and filter to 10-digit children starting with the full prefix
 * If nothing is found, returns [] (caller will fallback to search).
 */
function fetchByNumericPrefix_(digits) {
  if (!digits) return [];

  var len = digits.length;
  if (![2,4,6,8].includes(len)) {
    return [];
  }

  // Helper: take "included" records and extract commodity children
  function fromIncluded_(included, prefix) {
    if (!Array.isArray(included)) return [];
    var rows = [];
    for (var i = 0; i < included.length; i++) {
      var it = included[i] || {};
      var attrs = it.attributes || {};
      var code = String(attrs.goods_nomenclature_item_id || attrs.goods_nomenclature_sid || '').trim();
      if (!code || code.length !== 10) continue;
      if (prefix && code.indexOf(prefix) !== 0) continue;
      var desc = String(attrs.formatted_description || attrs.description_plain || attrs.description || '').trim();
      rows.push({ code: code, description: desc });
    }
    return rows;
  }

  try {
    if (len === 2) {
      // CHAPTER
      var urlC = HMRC_CHAPTER_V2 + encodeURIComponent(digits);
      var resC = UrlFetchApp.fetch(urlC, { headers: { 'Accept': 'application/json' }, muteHttpExceptions: true });
      if (resC.getResponseCode() === 200) {
        var dataC = JSON.parse(resC.getContentText() || '{}');
        var listC = fromIncluded_(dataC && dataC.included, digits);
        if (listC.length) return dedupeAndSort_(listC, true).map(minify_);
      }
    }

    // HEADINGS (for 4 digits directly, and fallback for 6/8 by first 4)
    var heading4 = digits.slice(0, 4);
    var urlH = HMRC_HEADING_V2 + encodeURIComponent(heading4);
    var resH = UrlFetchApp.fetch(urlH, { headers: { 'Accept': 'application/json' }, muteHttpExceptions: true });
    if (resH.getResponseCode() === 200) {
      var dataH = JSON.parse(resH.getContentText() || '{}');
      var listH = fromIncluded_(dataH && dataH.included, digits);
      if (listH.length) return dedupeAndSort_(listH, true).map(minify_);
    }
  } catch (e) {
    Logger.log('fetchByNumericPrefix_ error: ' + e);
  }

  return [];
}

// ---------------- HMRC fetch helpers ----------------

/**
 * Exact commodity lookup for 10-digit code via /api/v2/commodities/{code}
 * Returns [{code, description}] or [].
 */
function fetchCommodityExact_(code10) {
  var url = HMRC_COMMODITY_V2 + encodeURIComponent(code10);
  var res = UrlFetchApp.fetch(url, {
    headers: { 'Accept': 'application/json' },
    muteHttpExceptions: true
  });

  var status = res.getResponseCode();
  if (status !== 200) {
    Logger.log('Commodity API (v2) status ' + status + ' for ' + code10);
    return [];
  }

  var txt = res.getContentText();
  if (!txt) return [];

  var data = JSON.parse(txt);
  var attrs = data && data.data && data.data.attributes;
  if (!attrs) return [];

  var desc = String(
    attrs.formatted_description ||
    attrs.description_plain ||
    attrs.description ||
    'No description'
  );

  var code = String(attrs.goods_nomenclature_item_id || code10);
  return [{ code: code, description: desc }];
}

/**
 * Try v2 search, then v1 search, and merge.
 * Returns an array of raw items with: { code, description } (no declarable filtering).
 */
function fetchSearchRawAll_(term) {
  var v2 = [];
  try { v2 = fetchSearchV2_(term); } catch (e) { Logger.log('v2 search failed: ' + e); }
  var v1 = [];
  try { v1 = fetchSearchV1_(term); } catch (e) { Logger.log('v1 search failed: ' + e); }

  var all = v2.concat(v1);
  // sanitize
  return all.filter(function (r) { return r && r.code; });
}

/**
 * Search endpoint v2.
 * Attempts to pull commodities from both goods_nomenclature_match and reference_match.
 */
function fetchSearchV2_(term) {
  var url = HMRC_SEARCH_V2 + encodeURIComponent(term);
  var res = UrlFetchApp.fetch(url, {
    headers: { 'Accept': 'application/json' },
    muteHttpExceptions: true
  });

  if (res.getResponseCode() !== 200) {
    throw new Error('HMRC v2 search failed with status ' + res.getResponseCode());
  }

  var data = JSON.parse(res.getContentText() || '{}');

  // Known structure path A (current docs)
  var gn = getIn_(data, ['data', 'attributes', 'goods_nomenclature_match', 'commodities']) || [];
  var rf = getIn_(data, ['data', 'attributes', 'reference_match', 'commodities']) || [];

  // Some deployments return arrays under data[].attributes...
  if (!gn.length && Array.isArray(data && data.data)) {
    try {
      for (var i = 0; i < data.data.length; i++) {
        var a = getIn_(data.data[i], ['attributes', 'goods_nomenclature_match', 'commodities']);
        if (Array.isArray(a)) gn = gn.concat(a);
        var b = getIn_(data.data[i], ['attributes', 'reference_match', 'commodities']);
        if (Array.isArray(b)) rf = rf.concat(b);
      }
    } catch (e) {}
  }

  var list = [].concat(gn || [], rf || []);

  return list.map(function (hit) {
    var src = (hit && hit._source) || {};
    var code = String(src.goods_nomenclature_item_id || '').trim();
    var desc = String(src.description || '').trim();
    return { code: code, description: desc };
  });
}

/**
 * Search endpoint v1 (fallback).
 * v1 returns flatter arrays (commodities/headings/etc). We only map commodities.
 */
function fetchSearchV1_(term) {
  var url = HMRC_SEARCH_V1 + encodeURIComponent(term);
  var res = UrlFetchApp.fetch(url, {
    headers: { 'Accept': 'application/json' },
    muteHttpExceptions: true
  });

  if (res.getResponseCode() !== 200) {
    throw new Error('HMRC v1 search failed with status ' + res.getResponseCode());
  }

  var data = JSON.parse(res.getContentText() || '{}');
  var commodities = (data && data.commodities) || [];

  // Some v1 variants put results under results.commodities
  if (!commodities.length) {
    commodities = getIn_(data, ['results', 'commodities']) || [];
  }

  return commodities.map(function (c) {
    var code = String(c.goods_nomenclature_item_id || c.code || '').trim();
    var desc = String(c.description || c.title || '').trim();
    return { code: code, description: desc };
  });
}

// ---------------- Transform & filter helpers ----------------

/**
 * Keep exactly one item that equals the 10-digit code, if present.
 */
function filterExact10_(rawList, code10) {
  var out = (rawList || []).filter(function (r) {
    return r && r.code && String(r.code).trim() === String(code10);
  });
  return dedupeAndSort_(out, /*put10First=*/true).map(minify_);
}

/**
 * Given raw items [{code, description}], keep:
 * - codes that START WITH the numeric prefix
 * - exactly 10 digits
 * De-dupes and sorts.
 */
function filterPrefixTo10_(rawList, prefix) {
  var filtered = (rawList || []).filter(function (r) {
    if (!r || !r.code) return false;
    var code = String(r.code).trim();
    if (prefix && code.indexOf(prefix) !== 0) return false;
    return code.length === 10;
  });

  return dedupeAndSort_(filtered, /*put10First=*/true).map(minify_);
}

/**
 * Keyword-style payload from raw search (deduped, 10-digit first).
 */
function mapToPayload_(rawList) {
  var cleaned = (rawList || []).filter(function (r) {
    return r && r.code;
  });

  var deduped = dedupeAndSort_(cleaned, /*put10First=*/true);

  return deduped.map(minify_);
}

function minify_(r) {
  return {
    code: String(r.code || ''),
    description: String(r.description || '')
  };
}

/**
 * De-duplicate by code and sort by code (optionally putting 10-digit first).
 */
function dedupeAndSort_(list, put10First) {
  var seen = Object.create(null);
  var out = [];
  for (var i = 0; i < list.length; i++) {
    var r = list[i];
    if (!r || !r.code) continue;
    var code = String(r.code);
    if (seen[code]) continue;
    seen[code] = true;
    out.push({ code: code, description: String(r.description || '') });
  }

  out.sort(function (a, b) {
    if (put10First) {
      var a10 = a.code.length === 10 ? 0 : 1;
      var b10 = b.code.length === 10 ? 0 : 1;
      if (a10 !== b10) return a10 - b10;
    }
    // Lexicographic on code (covers numeric order well for fixed-length codes)
    if (a.code < b.code) return -1;
    if (a.code > b.code) return 1;
    return 0;
  });

  return out;
}

// Safe getter
function getIn_(obj, pathArr) {
  var cur = obj;
  for (var i = 0; i < pathArr.length; i++) {
    if (cur == null) return undefined;
    cur = cur[pathArr[i]];
  }
  return cur;
}

// ---------------- Utilities & optional POST hook ----------------

function doPost(e) {
  try {
    var body = JSON.parse((e && e.postData && e.postData.contents) || '{}');
    var query = body.p_search;
    var result = searchHS(query);
    return ContentService.createTextOutput(JSON.stringify(result))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({ error: error.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// ---------------- Logging (best-effort) ----------------

function logSearch_(keyword) {
  if (typeof keyword !== 'string') return;

  var ss = SpreadsheetApp.openById(LOG_SPREADSHEET_ID);
  var sheet = ss.getSheets()[0];
  ensureHeaderIfEmpty_(sheet);

  var email = 'Unknown';
  try {
    email = Session.getActiveUser().getEmail() || 'Unknown';
  } catch (e) { email = 'Unknown'; }

  var ts = Utilities.formatDate(new Date(), LOG_TIMEZONE, 'yyyy-MM-dd HH:mm:ss');
  sheet.appendRow([email, ts, keyword]);
}

function ensureHeaderIfEmpty_(sheet) {
  var lastRow = sheet.getLastRow();
  if (lastRow === 0) {
    sheet.getRange(1, 1, 1, 3).setValues([['Name', 'Time', 'Keyword']]);
  }
}

// Quick self-test for logging only
function SELF_TEST__LOGGING_ONLY() {
  var testKeyword = 'SELF_TEST_' + Date.now();
  logSearch_(testKeyword);

  var ss = SpreadsheetApp.openById(LOG_SPREADSHEET_ID);
  var sheet = ss.getSheets()[0];
  var lastRow = sheet.getLastRow();
  var values = sheet.getRange(lastRow, 1, 1, 3).getValues()[0];

  return { status: 'VERIFICATION_LOG_OK', lastRow: lastRow, values: values };
}
```

## Key Fixes in This Version:

1. **Handles formatted 10-digit input**: `"2204 1013 00"` works now
2. **Triple-layer fallback** for 10-digit codes:
   - Layer 1: `/commodities/{code}` endpoint
   - Layer 2: Search API with exact match
   - Layer 3: Heading structure (catches non-declarable codes)
3. **Supports odd-length prefixes**: 3/5/7/9 digits
4. **Routes by digit count**, not string format

## Instructions:

1. Copy everything inside the code block (starting from `// === Code.gs` to the last `}`)
2. Paste it into your Google Apps Script `Code.gs` file
3. Replace ALL existing code
4. Save and test
