# Canal+ "App Users" Data Quality Analysis
## Comparison of 3 Raw Requests

---

## 🔴 ISSUE IDENTIFIED: Missing User Properties in Vacuum Pipeline

---

## Key Finding

**"App Users" is a USER PROPERTY, not an Event Property**

Users coming through the **Vacuum ETL pipeline** are missing the root-level `user_properties` object, which is why they show **(none)** in your Amplitude chart.

---

## Comparison Table

| Aspect | Request 1 (none) | Request 2 (none) | Request 3 ✅ HAS VALUE |
|--------|------------------|------------------|------------------------|
| **API Path** | `/batch` | `/batch` | `/2/httpapi` |
| **Library/Source** | `vacuum/generic_converter_2020_03` | `vacuum/generic_converter_2020_03` | Direct SDK |
| **Platform** | `web` | `web` | `Canal+ STB` |
| **Device Category** | `desktop` | `desktop` | `STB` |
| **Server** | `vod.canalplus.com` | `www.canalplus.com` | `webapp.local` |
| **User-Agent Header** | `Apache-HttpClient/4.5.14` | `Apache-HttpClient/4.5.14` | `Apache-HttpAsyncClient/4.1.5` |
| **Root user_properties** | ❌ **MISSING** | ❌ **MISSING** | ✅ **EXISTS** |
| **"app users" property** | ❌ Not present | ❌ Not present | ✅ `"myCANAL"` |
| **app_version** | ❌ Not present | ❌ Not present | ✅ `"5.8.1-1-PROD"` |
| **device_model** | `"Unknown"` | `"Unknown"` | ✅ `"G9 SAT"` |
| **os_name** | `"none"` | `"none"` | `"none"` |
| **os_version** | `"none"` | `"none"` | `"none"` |
| **subscription type** | `"Unknown"` | `"Unknown"` | ✅ `"OAE,CNL,ESS,..."` |
| **subscriber status** | `" prospect"` (leading space) | `" prospect"` (leading space) | ✅ `"Subscriber"` |

---

## Critical Differences

### Request 3 (Working - Has "app users" value)

```json
{
  "device_id": "1737601568647-adb78c6a63ae",
  "app_version": "5.8.1-1-PROD",
  "device_model": "G9 SAT",
  "user_properties": {                           // ← ROOT LEVEL USER PROPERTIES
    "app users": "myCANAL",                      // ← THIS IS THE KEY PROPERTY
    "app zone group": "fr",
    "login status": "Logged-in",
    "subscriber status": "Subscriber",
    "subscription type": "OAE,CNL,ESS,O_ABO,NO,CP,TV,PANO,PARP,PDE,UEFA,ATV",
    "device category": "STB",
    "platform type": "STB",
    "platform tvod": "Canal+ STB"
  },
  "event_type": "view page",
  "platform": "Canal+ STB"
}
```

### Requests 1 & 2 (Not Working - Missing user_properties)

```json
{
  "device_id": "c9bd27d9f3c92c502ed1827160d2cc70...",
  // NO app_version
  // NO user_properties object at root level
  "event_type": "pay vod",
  "library": "vacuum/generic_converter_2020_03",
  "platform": "web",
  "event_properties": {
    "app name": "CANAL VOD",                     // ← Only in event properties, not user properties
    "device model": "Unknown",
    "subscriber status": " prospect",            // ← Data quality issue: leading space
    "subscription type": "Unknown",
    "os name": "none",
    "os version": "none"
  }
}
```

---

## Root Cause

Your **Vacuum ETL pipeline** (`vacuum/generic_converter_2020_03`) is:

1. ❌ **NOT mapping data to the root-level `user_properties` object**
2. ❌ **NOT setting the "app users" user property**
3. ❌ **NOT extracting app_version**
4. ❌ **NOT properly parsing OS from User Agent strings**
5. ❌ **NOT setting device_model correctly** (defaulting to "Unknown")
6. ⚠️ **Adding leading spaces** to some values like subscriber status

---

## Data Sources Comparison

### Source 1: Vacuum Pipeline (Web Users)
- **Path**: `/batch`
- **Ingestion**: Vacuum ETL (`vacuum/generic_converter_2020_03`)
- **Platform**: `web`
- **Result**: **(none)** for "app users"
- **Affected Users**: Web users watching VOD on desktop

### Source 2: Direct SDK (STB Users)
- **Path**: `/2/httpapi`
- **Ingestion**: Direct from SDK
- **Platform**: `Canal+ STB`
- **Result**: ✅ **"myCANAL"**
- **Affected Users**: Set-Top Box users

---

## Impact on Your Chart

When you group by **"App Users"** (a user property):

- **STB users** → Show as "myCANAL" ✅
- **Web users (via Vacuum)** → Show as **(none)** ❌

---

## Recommended Solutions

### Option 1: Fix Vacuum Pipeline (Best Solution)
Update your `vacuum/generic_converter_2020_03` ETL to:

```javascript
// Add user_properties mapping
user_properties: {
  "app users": event_properties["app name"],  // Map "CANAL VOD" or "myCANAL"
  "subscriber status": event_properties["subscriber status"].trim(),  // Remove leading spaces
  "subscription type": event_properties["subscription type"],
  "device category": event_properties["device category"],
  "login status": event_properties["login status"]
}
```

### Option 2: Use Event Property Instead
Change your Amplitude chart to use:
- **Event Property**: `app name` (already exists)
- Instead of **User Property**: `app users`

### Option 3: Create a Computed User Property
In Amplitude, create a computed user property:
- **Property Name**: `app users (computed)`
- **Definition**: `LAST(event_properties["app name"])`

### Option 4: Backfill via Identify API
Use Amplitude's Identify API to set user properties for affected users based on their event properties.

---

## Data Quality Issues to Fix

1. **Leading spaces**: `" prospect"` → `"prospect"`
2. **OS parsing**: All showing `"none"` despite valid User Agent strings
3. **Device model**: Defaulting to `"Unknown"` instead of parsing from User Agent
4. **Missing user properties**: Critical for user-level analysis
5. **Inconsistent app naming**: "CANAL VOD" vs "myCANAL" vs "myCANAL"

---

## Quick Validation Query

Run this in Amplitude to see the split:

**Event Segmentation Chart:**
- Event: `pay vod` or `view page`
- Group by: `app users` (user property)
- Also add: `app name` (event property)

Compare the two to see the gap.

---

## Next Steps

1. **Short-term**: Switch chart to use `app name` event property
2. **Long-term**: Fix Vacuum pipeline to populate user_properties
3. **Data Quality**: Review all Vacuum mappings for completeness
4. **Validation**: Test pipeline changes with sample events

---

**Analysis Date**: November 27, 2025  
**Analyst**: Amplitude Support Investigation

