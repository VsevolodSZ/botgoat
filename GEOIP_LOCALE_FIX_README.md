# GeoIP Locale Fix for Chromium

This patch fixes locale detection for GeoIP-based fingerprinting protection in Chromium.

## What it fixes

**Problem**: CreepJS and similar fingerprinting tools detect inconsistencies when:
- `navigator.language` returns "ja-JP"
- `Intl.DateTimeFormat().resolvedOptions().locale` returns "ja" (without country suffix)
- `navigator.languages` array doesn't include the short form

**Solution**:
- `GetGeoIPLocale()` now returns both forms: "ja-JP,ja"
- Browser UI uses the first (full) locale: "ja-JP"
- `navigator.languages` includes both: ["ja-JP", "ja"]
- All Workers (Dedicated, Shared, Service) use the same locale list
- No more red flags in CreepJS tests!

## Changed Files

### Core GeoIP Changes
- `content/browser/geoip/geoip_initializer.cc` - Returns "locale-COUNTRY,locale" format
- `content/browser/geoip/geoip_initializer.h` - Function declarations
- `content/browser/geoip/BUILD.gn` - Build configuration (removed content/common dependency)
- `content/browser/geoip/*.h` - Removed CONTENT_EXPORT macros

### Locale Integration
- `chrome/app/chrome_main_delegate.cc` - Initialize GeoIP BEFORE loading locale
- `chrome/browser/chrome_resource_bundle_helper.cc` - Use GeoIP locale for browser UI (first from list)
- `chrome/browser/renderer_preferences_util.cc` - Use GeoIP locale for accept_languages (full list)
- `content/browser/renderer_host/render_process_host_impl.cc` - Use GeoIP locale for --lang flag (first from list)

### Anti-Detection Fixes
- `components/variations/service/variations_service.cc` - Skip locale consistency check when --geoip flag is active
- `content/browser/browser_main_loop.cc` - Removed duplicate GeoIP initialization

### Build System
- `chrome/browser/BUILD.gn` - Added direct dependency on geoip
- `content/browser/BUILD.gn` - Added geoip to deps (not public_deps)

## How to Apply

```bash
cd /path/to/chromium/src
git apply geoip_locale_fix.patch
```

## Testing

1. Start Chromium with GeoIP:
```bash
chrome.exe --geoip --proxy-server="socks5://127.0.0.1:12000"
```

2. Open CreepJS test page
3. Check that:
   - ✅ Timezone shows correct location (e.g., "Asia/Tokyo")
   - ✅ Intl shows correct locale (e.g., "ja")
   - ✅ Worker shows correct language (e.g., "ja-JP")
   - ✅ No red flags for locale mismatches
   - ✅ `navigator.languages` includes both "ja-JP" and "ja"

## Technical Details

### Why Both Locale Forms?

Chromium's ResourceBundle normalizes locales:
- Input: "ja-JP" → Output: "ja" (when country matches language region)
- Input: "en-US" → Output: "en-US" (kept as-is for compatibility)

To avoid fingerprinting detection, we provide both forms:
- Full form ("ja-JP") for `navigator.language` and browser UI
- Short form ("ja") automatically included in `navigator.languages`
- Intl APIs use the normalized form ("ja")

This ensures CreepJS check passes:
```javascript
const localeIntlEntropyIsTrusty =
  new Set(navigator.languages).has(Intl.DateTimeFormat().resolvedOptions().locale);
// ["ja-JP", "ja"].includes("ja") === true ✅
```

### Initialization Order

Critical: GeoIP **must** initialize BEFORE locale loading:

1. `chrome_main_delegate.cc` → `InitializeGeoIPIfNeeded()`
2. `chrome_main_delegate.cc` → `LoadLocalState()` → `InitResourceBundleAndDetermineLocale()`
3. ResourceBundle loads with GeoIP locale
4. Browser UI displays in correct language

## Compatibility

- Tested on Chromium M143 (Windows)
- Compatible with `--proxy-server` flag
- Works with SOCKS5 proxies
- No performance impact when `--geoip` flag is not used

## Author

Created for anti-fingerprinting purposes.
