# Phase 1 Implementation Prompt

Use this prompt to start a new chat for implementing Phase 1 of the Smart Farmer Search plugin fixes.

---

## PROMPT START

```
# Task: Implement Phase 1 - Critical Config Fixes for Smart Farmer Search Plugin

## Context

I'm working on a Joget DX8/DX9 plugin called "Smart Farmer Search" (joget-smart-search). A code review identified 23 issues, and we're fixing them in phases. This is **Phase 1: Critical Fixes (P1)**.

### Project Location
```
/Users/aarelaponin/IdeaProjects/gs-plugins/joget-smart-search/
```

### Background Documents (READ THESE FIRST)
- `docs/CODE_REVIEW_REPORT.md` - Full issue list with details
- `docs/TESTING_STRATEGY.md` - Testing approach
- `docs/IMPLEMENTATION_PLAN.md` - Phased plan (we're doing Phase 1)
- `CLAUDE.md` - Project architecture guide

### Plugin Architecture Summary
The plugin has a configuration flow:
1. **JSON** (`SmartSearchElement.json`) - Property definitions with defaults
2. **Java** (`SmartSearchElement.java`) - Reads properties, passes to FTL
3. **FTL** (`SmartSearchElement.ftl`) - Template passes config to JavaScript
4. **JS** (`smart-search.js`) - Uses config values

The P1 issues are all **broken config propagation** - properties defined in JSON that never reach JavaScript.

---

## Phase 1 Issues to Fix

### Issue #1: `showRecentFarmers` Not Propagated
- **JSON** (line 65-73): Property defined with checkbox, default ""
- **Java**: NOT read - missing `getPropertyString("showRecentFarmers")`
- **FTL**: NOT passed to JS config object
- **JS**: Recent farmers panel always shows regardless of setting

### Issue #2: `maxRecentFarmers` Not Propagated
- **JSON** (line 75-80): Property defined, default "5"
- **Java**: NOT read
- **FTL**: NOT passed
- **JS** (line 280): Hardcoded `new RecentFarmers(10)` instead of using config
- **recent-farmers.js** (line 17): `DEFAULT_MAX_ITEMS = 10` (inconsistent with JSON default of 5)

### Issue #3: `nationalIdPattern` Not Passed to JS
- **JSON** (line 115-120): Property defined, default `^\d{9,13}$`
- **Java** (line 219): IS read and passed to dataModel
- **FTL**: NOT in JS config object (missing!)
- **JS** (line 2319): Falls back to hardcoded pattern

### Issue #4: `phonePattern` Not Passed to JS
- **JSON** (line 129-134): Property defined, default `^\+?\d{8,}$`
- **Java** (line 221): IS read and passed to dataModel
- **FTL**: NOT in JS config object (missing!)
- **JS** (line 2320): Falls back to hardcoded pattern

---

## Files to Modify

| File | Changes Needed |
|------|----------------|
| `src/main/java/.../element/SmartSearchElement.java` | Add reading of `showRecentFarmers`, `maxRecentFarmers` |
| `src/main/resources/templates/SmartSearchElement.ftl` | Add all 4 properties to JS config object |
| `src/main/resources/static/smart-search.js` | Use `config.maxRecentFarmers` at line ~280, conditionally render recent farmers panel based on `config.showRecentFarmers` |
| `src/main/resources/static/recent-farmers.js` | Change `DEFAULT_MAX_ITEMS` from 10 to 5 |

---

## Implementation Steps

### Step 1: Fix Java Property Reading

In `SmartSearchElement.java`, find the `renderTemplate()` method and add:

```java
// Recent farmers config (Issue #1, #2)
String showRecentFarmersStr = getPropertyString("showRecentFarmers");
boolean showRecentFarmers = "true".equalsIgnoreCase(showRecentFarmersStr);
dataModel.put("showRecentFarmers", showRecentFarmers);

String maxRecentFarmersStr = getPropertyString("maxRecentFarmers");
int maxRecentFarmers = 5; // Match JSON default
try {
    if (maxRecentFarmersStr != null && !maxRecentFarmersStr.isEmpty()) {
        maxRecentFarmers = Integer.parseInt(maxRecentFarmersStr);
    }
} catch (NumberFormatException e) {
    // Use default
}
dataModel.put("maxRecentFarmers", maxRecentFarmers);
```

### Step 2: Fix FTL Config Object

In `SmartSearchElement.ftl`, find the JavaScript config object (around line 203) and add:

```javascript
var config = {
    // ... existing properties ...

    // Issue #1, #2: Recent farmers config
    showRecentFarmers: ${showRecentFarmers?c},
    maxRecentFarmers: ${maxRecentFarmers!5},

    // Issue #3, #4: Patterns (already in dataModel, just not passed to JS)
    nationalIdPattern: '${nationalIdPattern?js_string}',
    phonePattern: '${phonePattern?js_string}',

    // ... rest of existing properties ...
};
```

### Step 3: Fix smart-search.js

Find line ~280 where RecentFarmers is initialized:
```javascript
// BEFORE:
this.recentFarmersManager = new RecentFarmers(10);

// AFTER:
this.recentFarmersManager = new RecentFarmers(this.config.maxRecentFarmers || 5);
```

Find where recent farmers panel is rendered and add condition:
```javascript
// Conditionally render based on config.showRecentFarmers
if (this.config.showRecentFarmers) {
    this.renderRecentFarmersPanel();
}
```

### Step 4: Fix recent-farmers.js

Change line 17:
```javascript
// BEFORE:
var DEFAULT_MAX_ITEMS = 10;

// AFTER:
var DEFAULT_MAX_ITEMS = 5;  // Match JSON default
```

---

## Testing Requirements

### Tests to Write (if test infrastructure exists)

```
SmartSearchElementTest.java:
- renderTemplate_shouldReadShowRecentFarmers()
- renderTemplate_shouldReadMaxRecentFarmers()

config-flow.test.js:
- showRecentFarmers should be in JS config
- maxRecentFarmers should be in JS config
- nationalIdPattern should be in JS config
- phonePattern should be in JS config
```

### Manual Verification

1. Build the plugin: `mvn clean package`
2. Deploy JAR to Joget
3. Open Form Builder with Smart Search element
4. **Test showRecentFarmers:**
   - Uncheck "Show Recent Farmers"
   - Save form
   - Open form in runtime
   - Verify recent farmers panel is NOT shown
5. **Test maxRecentFarmers:**
   - Set to "3"
   - Select 5 different farmers
   - Verify only last 3 are shown in recent list
6. **Test patterns:**
   - Change nationalIdPattern to `^\d{4}$`
   - Verify 4-digit input is detected as ID (not name)

---

## Regression Checklist

After making changes, verify these still work:

- [ ] Search by National ID returns exact match
- [ ] Search by Phone returns exact match
- [ ] Search by Name + District returns results
- [ ] Inline quick entry detects input types correctly
- [ ] Recent farmers saves selections to localStorage
- [ ] Recent farmers loads on dialog open
- [ ] Selecting a recent farmer works
- [ ] Auto-select for high-confidence single result works

---

## Constraints

1. **Backward Compatibility**: Default behavior should match current behavior
   - showRecentFarmers defaults to false (checkbox unchecked = panel hidden by default)
   - Actually, looking at current code, panel always shows. Discuss: should default be true to maintain current behavior?

2. **FreeMarker Syntax**: Use `?c` for booleans, `?js_string` for strings that might have special chars

3. **No Breaking Changes**: Ensure forms using defaults continue to work

---

## Expected Deliverables

1. Modified files with fixes
2. Build verification (`mvn clean package` succeeds)
3. Manual test results documented
4. Commit message: `fix(config): propagate showRecentFarmers, maxRecentFarmers, patterns to JS`

---

## Questions to Clarify Before Starting

1. Should `showRecentFarmers` default to `true` (maintain current behavior where panel shows) or `false` (checkbox semantics where unchecked = hidden)?

2. Should I also set up the test infrastructure (Phase 0) first, or just do the fixes with manual testing?
```

## PROMPT END

---

## Notes for Using This Prompt

1. **Start a new chat** and paste the content between "PROMPT START" and "PROMPT END"

2. **The prompt includes:**
   - Full context about the project
   - Specific issues to fix with line numbers
   - Step-by-step implementation guide
   - Testing requirements
   - Regression checklist
   - Clarifying questions

3. **Expected outcome:**
   - All 4 P1 issues fixed
   - Plugin builds successfully
   - Manual verification passes
   - Ready for commit

4. **Time estimate:** ~2 hours for implementation + testing
