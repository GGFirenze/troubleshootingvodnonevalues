# Canal+ "App Users" Data Quality Analysis
## Comparison of 3 Raw Requests

---

## 🔴 ISSUE IDENTIFIED: Missing User Properties Across Multiple Platforms

---

## Key Finding

**"App Users" is a USER PROPERTY, not an Event Property**

Multiple data sources are missing the root-level `user_properties` object, which is why 18,124 users show **(none)** in your Amplitude chart.

## Impact Scale (Last 30 Days)

| Platform | Affected Users | % of Total | Status |
|----------|----------------|------------|--------|
| **Web** | **17,243** | **95.2%** | ❌ Primary Issue |
| Samsung TV | 508 | 2.8% | ❌ Missing |
| lg tv | 236 | 1.3% | ❌ Missing |
| hisense tv | 118 | 0.7% | ❌ Missing |
| titan | 19 | 0.1% | ❌ Missing |
| **TOTAL (none)** | **18,124** | **100%** | ❌ |
| **STB (myCANAL)** | **Working** | **N/A** | ✅ |

**Business Impact**: 95% of the problem is Web platform via Vacuum ETL pipeline.

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

## Root Cause Analysis

This is a **systematic user property mapping problem** affecting multiple data sources:

### Primary Issue: Vacuum ETL Pipeline (95.2% of problem)
Your **Vacuum ETL pipeline** (`vacuum/generic_converter_2020_03`) is:

1. ❌ **NOT mapping data to the root-level `user_properties` object**
2. ❌ **NOT setting the "app users" user property**
3. ❌ **NOT extracting app_version**
4. ❌ **NOT properly parsing OS from User Agent strings**
5. ❌ **NOT setting device_model correctly** (defaulting to "Unknown")
6. ⚠️ **Adding leading spaces** to some values like subscriber status

### Secondary Issues: TV Platform SDKs (4.8% of problem)
The following platforms also have missing user properties:
- **Samsung TV SDK** (508 users) - Not setting user properties on initialization
- **LG TV SDK** (236 users) - Missing user_properties object
- **Hisense TV SDK** (118 users) - User properties not configured
- **Titan platform** (19 users) - User property mapping incomplete

All these sources are **only populating event properties, not user properties**.

---

## Data Sources Comparison

### Source 1: Vacuum Pipeline (Web Users) - 95.2% OF PROBLEM
- **Path**: `/batch`
- **Ingestion**: Vacuum ETL (`vacuum/generic_converter_2020_03`)
- **Platform**: `web`
- **Affected Users**: 17,243 users
- **Result**: **(none)** for "app users"
- **User Type**: Web users watching VOD on desktop browsers
- **Priority**: 🔴 **HIGHEST - Fix this first**

### Source 2: Smart TV Platforms - 4.8% OF PROBLEM
- **Platforms**: Samsung TV, LG TV, Hisense TV, Titan
- **Affected Users**: 881 users (508 + 236 + 118 + 19)
- **Result**: **(none)** for "app users"
- **User Type**: Smart TV app users
- **Priority**: 🟡 **MEDIUM - Fix after Vacuum pipeline**

### Source 3: Direct SDK (STB Users) - ✅ WORKING
- **Path**: `/2/httpapi`
- **Ingestion**: Direct from SDK
- **Platform**: `Canal+ STB`
- **Result**: ✅ **"myCANAL"**
- **User Type**: Set-Top Box users
- **Status**: ✅ **No action needed - this is the reference implementation**

---

## Impact on Your Chart

When you group by **"App Users"** (a user property):

- **STB users** → Show as "myCANAL" ✅
- **Web users (via Vacuum)** → 17,243 show as **(none)** ❌
- **Smart TV users** → 881 show as **(none)** ❌
- **Total affected**: 18,124 users with no "app users" value

---

## Recommended Solutions

### 🔴 PRIORITY 1: Fix Vacuum Pipeline (Solves 95% of Problem)

**Impact**: Will fix 17,243 of 18,124 affected users

Update your `vacuum/generic_converter_2020_03` ETL to map event properties to user properties:

```javascript
// Add user_properties mapping
user_properties: {
  "app users": event_properties["app name"],  // Map "CANAL VOD" or "myCANAL"
  "subscriber status": event_properties["subscriber status"].trim(),  // Remove leading spaces
  "subscription type": event_properties["subscription type"],
  "device category": event_properties["device category"],
  "login status": event_properties["login status"],
  "platform type": "web"
}
```

**Timeline**: Implement this week - highest ROI

---

### 🟡 PRIORITY 2: Fix Smart TV SDKs (Solves remaining 5%)

**Impact**: Will fix 881 remaining users across 4 platforms

Review and update SDK configurations for:
1. **Samsung TV** (508 users) - Add user_properties to initialization
2. **LG TV** (236 users) - Configure user properties on app launch
3. **Hisense TV** (118 users) - Set user properties in SDK setup
4. **Titan** (19 users) - Enable user property tracking

**Timeline**: Next sprint after Vacuum fix

---

### ⚡ QUICK WIN: Use Event Property Instead

**Impact**: Immediate visibility into all platforms

Change your Amplitude chart to use:
- ❌ Remove: `app users` (user property)
- ✅ Add: `app name` (event property)

This works RIGHT NOW for all platforms because event properties are already populated.

**Timeline**: 5 minutes - do this today while waiting for engineering fixes

---

### 🔧 ALTERNATIVE: Create Computed User Property

In Amplitude, create a computed user property:
- **Property Name**: `app users (computed)`
- **Definition**: `LAST(event_properties["app name"])`

This auto-generates a user property from existing event data.

**Timeline**: 10 minutes in Amplitude UI

---

### 🔄 OPTIONAL: Backfill Historical Data

Use Amplitude's Identify API to set user properties for affected users based on their event properties.

**Timeline**: After pipeline fix, for historical data correction

---

## Data Quality Issues to Fix

1. **Leading spaces**: `" prospect"` → `"prospect"`
2. **OS parsing**: All showing `"none"` despite valid User Agent strings
3. **Device model**: Defaulting to `"Unknown"` instead of parsing from User Agent
4. **Missing user properties**: Critical for user-level analysis
5. **Inconsistent app naming**: "CANAL VOD" vs "myCANAL" vs "myCANAL"

---

## Quick Validation Query

Run this in Amplitude to validate the fix:

**Event Segmentation Chart:**
- Event: `pay vod` or `view page`
- Where: `platform` = `web` (to isolate Vacuum pipeline)
- Group by: `app users` (user property)
- Also add breakdown by: `app name` (event property)

**Expected Results:**
- **Before fix**: 17,243 showing "(none)" for `app users`
- **After fix**: All web users should show proper app name in `app users`

**Additional validation:**
- Check Samsung TV, LG TV, Hisense TV, Titan separately
- Monitor "(none)" percentage trending down over time

---

## Next Steps - Action Plan

### 🚀 TODAY (Quick Win)
1. **Change chart to use `app name` event property** - Get immediate visibility
2. **Share this analysis** with Data Engineering team
3. **Create Jira/ticket** for Vacuum pipeline fix

### 📅 THIS WEEK (Priority 1)
1. **Fix Vacuum pipeline** to populate user_properties
   - Will resolve 17,243 users (95% of problem)
   - Highest ROI action
2. **Test pipeline changes** with sample events
3. **Deploy to production**

### 📅 NEXT SPRINT (Priority 2)
1. **Audit Smart TV SDK implementations**
   - Samsung TV (508 users)
   - LG TV (236 users)
   - Hisense TV (118 users)
   - Titan (19 users)
2. **Update SDK configurations** to include user_properties
3. **Deploy updated TV apps**

### 📊 ONGOING
1. **Monitor data quality** - Track "(none)" percentage weekly
2. **Create alerts** if "(none)" values increase
3. **Review all Vacuum mappings** for other missing properties (OS, device model, etc.)
4. **Consider backfill** for historical data once pipeline is fixed

---

## Executive Summary

### The Problem
18,124 users show **(none)** for the "app users" user property, preventing proper user segmentation and analysis.

### The Root Cause
Multiple data sources are **only setting event properties, not user properties**:
- **Vacuum ETL pipeline** (Web) - 17,243 users (95.2%)
- **Smart TV SDKs** (Samsung/LG/Hisense/Titan) - 881 users (4.8%)

### The Solution
1. **Quick Win (Today)**: Change chart to use `app name` event property
2. **Priority 1 (This Week)**: Fix Vacuum pipeline mapping - solves 95% of problem
3. **Priority 2 (Next Sprint)**: Update TV SDK configurations

### Business Impact
- **Current**: Unable to segment users by app/platform reliably
- **After Fix**: Full visibility into user behavior across all platforms
- **ROI**: Fixing Vacuum pipeline alone resolves 17,243 users

---

**Analysis Date**: November 27, 2025  
**Analyst**: Amplitude Solutions Architecture  
**Chart Reference**: "Device ID on None Values - TO INVESTIGATE"  
**Data Period**: Last 30 days (Oct 28 - Nov 27, 2025)
