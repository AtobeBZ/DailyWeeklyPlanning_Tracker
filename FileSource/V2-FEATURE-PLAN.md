# V2 Feature Plan: Work Schedule Configuration

## Current Understanding

### What the User Wants

1. **Define Work Day template** - activities for a typical work day (Daily View)
2. **Define Off Day template** - activities for a typical off day (Daily View)
3. **Customize per weekday** - in Weekly View, modify schedule for specific days (e.g., Tuesday has piano)
4. **Define work schedule** - which days of week are work days (part-time support)
5. **Year View integration** - holidays/vacation override the schedule, KPIs update

### Data Flow (3-Tier Hierarchy)

```
TIER 1: Day Type Templates (Daily View)
    |
    |  "What does a typical Work Day look like?"
    |  "What does a typical Off Day look like?"
    |
    v
TIER 2: Weekday Configuration (Weekly View)
    |
    |  "Monday uses Work Day template"
    |  "Tuesday uses Work Day template + Piano at 20:00" (custom)
    |  "Friday uses Off Day template" (80% part-time)
    |  "Saturday uses Off Day template"
    |
    v
TIER 3: Date Overrides (Year View)
    |
    |  "2026-01-01 is Public Holiday" (ignores weekday config)
    |  "2026-08-15 is Vacation" (ignores weekday config)
    |
    v
FINAL: Resolved Activities for Any Date
```

---

## Current Code Analysis

### State Structure (V3 - Already Implemented)

```javascript
state = {
    version: 3,

    // TIER 1: Day Type Templates
    templates: {
        'Work Day': [...activities...],
        'Off Day': [...activities...],
        'Holiday': [...activities...]
    },

    // TIER 2: Weekday Configuration
    weekdayTemplates: {
        'Monday':    { baseType: 'Work Day', custom: null },
        'Tuesday':   { baseType: 'Work Day', custom: null },
        'Wednesday': { baseType: 'Work Day', custom: null },
        'Thursday':  { baseType: 'Work Day', custom: null },
        'Friday':    { baseType: 'Work Day', custom: null },
        'Saturday':  { baseType: 'Off Day', custom: null },
        'Sunday':    { baseType: 'Off Day', custom: null }
    },

    // TIER 3: Date Overrides
    dayOverrides: {
        '2026-01-01': 'holiday',
        '2026-12-25': 'holiday',
        // ... more holidays ...
    },

    // UI State
    dayTypes: ['Work Day', 'Off Day', 'Holiday'],
    currentDayType: 'Work Day',
    currentMainView: 'daily',
    categoryColors: {...}
}
```

### Year View Day Types (yearDayTypes)

```javascript
const yearDayTypes = {
    'work': { label: 'Work Day', class: 'work-day', color: '#d5f5e3' },
    'off': { label: 'Off Day', class: 'off-day', color: '#fadbd8' },
    'weekend': { label: 'Weekend', class: 'weekend', color: '#e8daef' },
    'holiday': { label: 'Public Holiday', class: 'holiday', color: 'hatched' },
    'vacation': { label: 'Vacation', class: 'vacation', color: '#d6eaf8' },
    'sick': { label: 'Sick Day', class: 'sick', color: '#f5eef8' }
};
```

### Key Functions Already Implemented

| Function | Location | Purpose |
|----------|----------|---------|
| `getWeekdayBaseType(day)` | Line 2686 | Returns base day type for a weekday |
| `setWeekdayBaseType(day, type)` | Line 2695 | Sets base day type for a weekday |
| `getDayType(year, month, day)` | Line 3135 | Resolves day type (override > weekday) |
| `calculateYearStats(year)` | Line 3161 | Calculates KPIs for year |
| `renderWeeklyView()` | Line 3017 | Renders weekly grid with activities |
| `renderYearView()` | Line 3218 | Renders year calendar |
| `setWeekDayType(day, type)` | Line 3067 | Updates weekday from dropdown |

### Data Inconsistency Issue

**Problem:** Two different key systems are used:

1. `state.weekdayTemplates[day].baseType` uses: `'Work Day'`, `'Off Day'`
2. `state.dayOverrides[date]` uses: `'holiday'`, `'vacation'`, `'sick'`, `'work'`, `'weekend'`
3. `yearDayTypes` keys: `'work'`, `'off'`, `'weekend'`, `'holiday'`, `'vacation'`, `'sick'`

**Solution:** Create mapping functions to convert between systems.

---

## What Needs to Be Built

### 1. Year View: Work Schedule Configuration Panel

**Location:** Between year navigation and KPI stats

**UI Design:**
```
                        2026                    [<] [>]

+------------------------------------------------------------------+
| Work Schedule                                                      |
|                                                                    |
| [Mon] [Tue] [Wed] [Thu] [Fri] [Sat] [Sun]                         |
|  [x]   [x]   [x]   [x]   [x]   [ ]   [ ]                          |
|                                                                    |
| Presets: [Full-time (Mon-Fri)] [80% (Mon-Thu)] [Custom]           |
+------------------------------------------------------------------+

+--------+--------+--------+--------+--------+
|  253   |   9    |   0    |  103   |  365   |
| Work   | Public | Days   | Week-  | Total  |
| Days   | Hols   | Off    | ends   | Days   |
+--------+--------+--------+--------+--------+
```

**Functionality:**
- Toggle buttons for each weekday (work = green/checked, off = gray/unchecked)
- Preset buttons for common patterns
- Changing toggles immediately updates:
  - `state.weekdayTemplates[day].baseType`
  - KPIs recalculate
  - Calendar colors update
  - Month stats update

### 2. Key Mapping Functions

```javascript
// Map yearDayTypes key to template name
function dayTypeKeyToTemplate(key) {
    const mapping = {
        'work': 'Work Day',
        'off': 'Off Day',
        'weekend': 'Off Day',
        'holiday': 'Off Day',  // Holidays use Off Day template
        'vacation': 'Off Day',
        'sick': 'Off Day'
    };
    return mapping[key] || 'Work Day';
}

// Map template name to yearDayTypes key
function templateToDayTypeKey(template) {
    const mapping = {
        'Work Day': 'work',
        'Off Day': 'off',
        'Holiday': 'holiday'
    };
    return mapping[template] || 'work';
}

// Check if a day type key is "work-like"
function isWorkLikeDay(dayTypeKey) {
    return dayTypeKey === 'work';
}
```

### 3. Activity Resolution Function

```javascript
function getActivitiesForDate(year, month, day) {
    // Step 1: Get effective day type from year view (override > weekday default)
    const effectiveDayType = getDayType(year, month, day); // Returns key like 'work', 'holiday'

    // Step 2: Determine if this is a work-like day
    const isWorkLike = isWorkLikeDay(effectiveDayType);

    // Step 3: Resolve activities
    if (isWorkLike) {
        const weekday = getWeekdayName(year, month, day); // 'Monday', 'Tuesday', etc.

        // Check for weekday-specific customization
        if (state.weekdayTemplates[weekday]?.custom) {
            return state.weekdayTemplates[weekday].custom;
        }

        // Fall back to base Work Day template
        return state.templates['Work Day'] || [];
    }

    // Off-like days always use Off Day template
    return state.templates['Off Day'] || [];
}
```

### 4. Update calculateYearStats()

Current logic counts based on `getDayType()` result. Need to ensure:
- When a weekday is toggled OFF in work schedule, it counts as weekend/off
- Holidays still count as holidays
- Vacation/sick still count separately

### 5. Weekly View Connection

The weekly view already has:
- Day type dropdown per column
- `setWeekDayType(day, type)` function

Need to ensure:
- Changes in weekly view update year view KPIs
- Visual consistency between views

---

## Implementation Order

### Phase 1: Work Schedule Config UI (Year View)
1. Add HTML for work schedule panel
2. Add CSS styles
3. Add toggle button functionality
4. Connect to `setWeekdayBaseType()`
5. Test: Toggle Friday off, verify KPIs change

### Phase 2: Preset Buttons
1. Add preset buttons HTML
2. Implement `applyWorkSchedulePreset(preset)` function
3. Presets: 'full-time', '80-mon-thu', '80-tue-fri', 'custom'
4. Test: Click preset, verify all toggles update

### Phase 3: KPI Recalculation
1. Review `calculateYearStats()` for accuracy
2. Ensure weekday base type changes affect KPI counts
3. Ensure month stats ("21 work / 10 off") update correctly
4. Test: Toggle Friday, verify January stats change

### Phase 4: Visual Calendar Update
1. Ensure calendar days reflect weekday config
2. Work days show as green, off days as purple
3. Holidays remain hatched (override)
4. Test: Toggle Friday, verify all Fridays turn purple

### Phase 5: Cross-View Sync
1. Weekly view changes sync to year view
2. Year view work schedule syncs to weekly view
3. Test: Change day type in weekly, verify year KPIs update

---

## Files to Modify

| File | Changes |
|------|---------|
| `daily-planner-light V2.html` | Add work schedule UI, mapping functions, update year view |

---

## Testing Checklist

- [ ] Year view shows work schedule config panel
- [ ] Toggle weekday updates KPIs immediately
- [ ] Toggle weekday updates calendar colors
- [ ] Toggle weekday updates month stats
- [ ] Preset buttons work correctly
- [ ] Holidays still show as hatched (not affected by toggles)
- [ ] Weekly view day type dropdown syncs with year view
- [ ] Data persists after page reload
- [ ] 80% schedule (Mon-Thu work) calculates correctly
- [ ] Vacation days still count separately from off days
