# Project Notes for Claude Code

## Important Guidelines

- **NEVER use emojis** in code, files, or conversation responses
- Keep responses concise and technical

## Project Overview

Daily/Weekly Planning Tracker - A time visualization and planning tool.

## Files Structure

```
DailyWeeklyPlanning_Tracker/
├── FileSource/
│   ├── daily-planner-light V0.html    # Original version (with bug fixes)
│   ├── daily-planner-light V1.html    # UX improved version (current)
│   ├── TimeSheetVF5_KB_2026 StartOfYear.xlsx  # Excel timesheet template
│   └── DailyWeeklyPlanner description.docx    # Feature documentation
├── TestData/
│   ├── sarah_developer.json           # Test user: Software Developer
│   ├── mike_freelancer_parent.json    # Test user: Freelancer with kids
│   ├── alex_student.json              # Test user: University Student
│   └── README.txt                     # Usage instructions
├── .mcp.json                          # MCP server config (playwright)
└── CLAUDE.md                          # This file
```

## Version History

### V0.html (Original)
- Fixed JavaScript bug: `state` variable accessed before initialization
- Simplified custom color picker (removed checkbox, auto-detects custom colors)

### V1.html (Current - UX Improved)
- Compact header with Daily/Weekly toggle
- Quick-add presets (Sleep, Lunch, Meeting, Exercise, Commute)
- Improved form layout (2-row design)
- Activity cards with hour badges and duration tags
- Visual feedback animations (slide-in, fade-out)
- Mobile responsive breakpoints (768px, 480px)
- **9 categories**: work, family, admin, myself, goal1, goal2, goal3, sleep, else
- **Cinnamon color scheme** with black borders
- Fixed legend not updating on add/delete activities

## Current Color Scheme (V1)

```css
--bg-primary: #faf6f1;      /* Warm beige */
--bg-secondary: #f5ede4;    /* Light cream */
--bg-tertiary: #efe5d8;     /* Soft tan */
--text-primary: #5d4037;    /* Brown */
--text-secondary: #8d6e63;  /* Light brown */
--accent: #d4a574;          /* Cinnamon */
--border: #000000;          /* Black outlines */
```

## Category Colors

| Category | Color | Hex |
|----------|-------|-----|
| Work | Red | #e74c3c |
| Family | Green | #27ae60 |
| Admin | Orange | #f39c12 |
| Myself | Teal | #16a085 |
| Goal 1 | Purple | #9b59b6 |
| Goal 2 | Pink | #e91e63 |
| Goal 3 | Blue | #3498db |
| Sleep | Muted Blue | #7c9cb5 |
| Else | Gray | #7f8c8d |

## Excel Timesheet (TimeSheetVF5_KB_2026)

French timesheet template for HUG (Geneva University Hospitals):

**Sheets:**
1. Calendrier - Main calendar with AM/PM tracking
2. Guide Utilisateur - User guide
3. Parameters - Configuration
4. Copyrights

**Tracking Categories:**
- Vacances (Vacation)
- Conge (Leave)
- Formation (Training)
- Recuperation (Comp time)
- Malade (Sick)
- Jour Travaille (Worked day)

**Sample Employee:**
- Name: Mike Marek
- Activity rate: 80%
- Canton: GE (Geneva)

## Bug Fixes Applied

1. **State initialization error** (V0 & V1):
   - Problem: `getCategoryColor()` called before `state` was defined
   - Fix: Changed `let categories = getCategories();` to `let categories = null;`

2. **Legend not updating** (V1):
   - Problem: `renderLegend()` not called after add/delete
   - Fix: Added `renderLegend()` to `addActivity()` and `deleteActivity()` functions

## Technical Notes

- HTML app uses Chart.js from CDN for weekly view charts
- Data stored in localStorage under key: `dailyPlannerStateLight`
- Overnight activities supported (e.g., sleep from 23:00 to 07:00)
- Activities sorted by start time automatically
- All sections have 2px black borders for clear visual separation

## Windows MCP Configuration

Working Playwright configuration:

**.mcp.json:**
```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"],
      "disabled": false
    }
  }
}
```

**.claude/settings.local.json** must include:
```json
{
  "enableAllProjectMcpServers": true,
  "enabledMcpjsonServers": ["playwright"]
}
```

## Testing

Use Playwright MCP to test the application:
- Import test JSON files from TestData/
- Test daily/weekly view switching
- Verify clock rendering and activity blocks
- Test add/edit/delete activities
- Check export/import functionality

---

## V2.html - Architecture & Data Model

### Version History

**V2.html (Current - Sprint 1 & 2 Complete)**
- Year view with 12-month calendar grid
- Click-to-change day type (Work, Off, Weekend, Holiday, Vacation, Sick)
- Geneva public holidays pre-loaded (2023-2027)
- Enhanced KPIs: Work Days, Public Holidays, Days Off Taken, Weekends
- Day overrides persisted to localStorage

### Current Data Model (V2 - Limited)

```javascript
state = {
  // Activities stored by DAY TYPE (not by weekday or date)
  activities: {
    'Work Day': [...],
    'Off Day': [...],
    'Holiday': [...],
  },

  // Week pattern: which day type applies to each weekday
  weekAssignments: {
    'Monday': 'Work Day',
    'Tuesday': 'Work Day',  // ALL Tuesdays use same Work Day activities
    'Wednesday': 'Work Day',
    'Thursday': 'Work Day',
    'Friday': 'Work Day',
    'Saturday': 'Off Day',
    'Sunday': 'Off Day',
  },

  // Year view: date-specific day type overrides
  dayOverrides: {
    '2026-12-25': 'Public Holiday',
    '2026-01-01': 'Public Holiday',
  }
}
```

**Limitation**: Cannot have different activities for different weekdays of the same type.
Example: Tuesday (Work Day) and Wednesday (Work Day) MUST have identical schedules.

---

## V3 Architecture - Hierarchical Activity Model

### Design Philosophy

The system uses a **3-tier hierarchical model** inspired by Outlook/calendar applications:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         TIER 1: DAY TYPE TEMPLATES                      │
│                              (Base Patterns)                            │
│                                                                         │
│   Define the foundational schedule for each category of day.            │
│   These are reusable patterns, not tied to specific dates.              │
│                                                                         │
│   Templates:                                                            │
│   ├── "Work Day"  → [Sleep 22:00-06:30, Work 08:30-17:30, ...]         │
│   ├── "Off Day"   → [Sleep 23:00-08:00, Family time, Hobbies, ...]     │
│   └── "Holiday"   → [Sleep late, Relaxation, ...]                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Inherits from
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      TIER 2: WEEKDAY TEMPLATES                          │
│                        (Per-Weekday Patterns)                           │
│                                                                         │
│   Customize activities for specific weekdays while inheriting           │
│   from the base day type. Only applies when day type is "work-like".   │
│                                                                         │
│   Weekday Schedule:                                                     │
│   ├── Monday    → uses "Work Day" template (no custom)                 │
│   ├── Tuesday   → uses "Work Day" + Piano 20:00-22:00, Sleep 23:00     │
│   ├── Wednesday → uses "Work Day" template (no custom)                 │
│   ├── Thursday  → uses "Work Day" + Gym 18:00-19:30                    │
│   ├── Friday    → uses "Work Day" + Early finish 16:00                 │
│   ├── Saturday  → uses "Off Day" template                              │
│   └── Sunday    → uses "Off Day" template                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Overridden by
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       TIER 3: YEAR VIEW OVERRIDES                       │
│                         (Date-Level Control)                            │
│                                                                         │
│   The Year view determines the EFFECTIVE day type for any date.         │
│   This can override the default weekday assignment.                     │
│                                                                         │
│   Day Type Resolution:                                                  │
│   ├── Check dayOverrides first (holidays, vacation, sick days)         │
│   ├── Fall back to weekAssignments (default pattern)                   │
│   └── Result determines which template tier to use                      │
│                                                                         │
│   Example Overrides:                                                    │
│   ├── 2026-12-25 (Friday) → "Public Holiday" (ignores Friday template) │
│   ├── 2026-08-03 (Monday) → "Vacation" (user is on leave)              │
│   └── 2026-01-13 (Tuesday) → "Work Day" (normal, uses Tuesday template)│
└─────────────────────────────────────────────────────────────────────────┘
```

### Activity Resolution Algorithm

```javascript
function getActivitiesForDate(date) {
  // Step 1: Determine effective day type from Year view
  const effectiveDayType = getEffectiveDayType(date);

  // Step 2: Classify the day type
  const isWorkLike = ['Work Day', 'Remote Day'].includes(effectiveDayType);
  const isOffLike = ['Off Day', 'Weekend', 'Public Holiday', 'Vacation', 'Sick Day'].includes(effectiveDayType);

  // Step 3: Resolve activities based on classification
  if (isWorkLike) {
    const weekday = getWeekdayName(date);  // "Tuesday"

    // Check for weekday-specific customization
    if (weekdayTemplates[weekday] && weekdayTemplates[weekday].custom) {
      return weekdayTemplates[weekday].custom;
    }

    // Fall back to base Work Day template
    return templates['Work Day'];
  }

  if (isOffLike) {
    // All off-like days use the Off Day template
    // Weekday customizations are IGNORED (you're not working!)
    return templates['Off Day'];
  }
}

function getEffectiveDayType(date) {
  const dateString = formatDate(date);  // "2026-01-13"

  // Priority 1: Explicit date override (from Year view)
  if (dayOverrides[dateString]) {
    return dayOverrides[dateString];
  }

  // Priority 2: Default weekday assignment
  const weekday = getWeekdayName(date);
  return weekAssignments[weekday];
}
```

### Example Scenarios

#### Scenario 1: Normal Tuesday (Work Day with Piano)
```
Date: Tuesday, January 13, 2026
Year View Override: None
Effective Day Type: "Work Day" (from weekAssignments)
Is Work-Like: YES
Weekday Template: Tuesday has custom activities
Result: Tuesday custom template (Work Day + Piano 20:00-22:00, Sleep 23:00)
```

#### Scenario 2: Tuesday that is a Public Holiday
```
Date: Tuesday, December 25, 2026 (Christmas)
Year View Override: "Public Holiday" (Geneva holidays)
Effective Day Type: "Public Holiday"
Is Work-Like: NO (it's off-like)
Result: Off Day template (NO piano - you're on holiday!)
```

#### Scenario 3: Tuesday marked as Vacation
```
Date: Tuesday, August 4, 2026 (User on vacation)
Year View Override: "Vacation" (user marked it)
Effective Day Type: "Vacation"
Is Work-Like: NO (it's off-like)
Result: Off Day template (NO work activities)
```

#### Scenario 4: Regular Wednesday (No customization)
```
Date: Wednesday, January 14, 2026
Year View Override: None
Effective Day Type: "Work Day" (from weekAssignments)
Is Work-Like: YES
Weekday Template: Wednesday has NO custom (null)
Result: Base Work Day template
```

### Proposed V3 Data Model

```javascript
const state = {
  // ============================================
  // TIER 1: Day Type Templates (Base Patterns)
  // ============================================
  templates: {
    'Work Day': [
      { id: 1, name: 'Sleep', startTime: '22:00', endTime: '06:30', category: 'sleep' },
      { id: 2, name: 'Morning Routine', startTime: '06:30', endTime: '07:15', category: 'myself' },
      { id: 3, name: 'Commute', startTime: '07:15', endTime: '08:00', category: 'else' },
      { id: 4, name: 'Deep Work', startTime: '08:00', endTime: '12:00', category: 'work' },
      { id: 5, name: 'Lunch', startTime: '12:00', endTime: '13:00', category: 'myself' },
      { id: 6, name: 'Meetings', startTime: '13:00', endTime: '17:00', category: 'work' },
      { id: 7, name: 'Commute Home', startTime: '17:00', endTime: '17:45', category: 'else' },
      { id: 8, name: 'Family Time', startTime: '17:45', endTime: '20:00', category: 'family' },
      { id: 9, name: 'Personal Time', startTime: '20:00', endTime: '22:00', category: 'myself' },
    ],
    'Off Day': [
      { id: 101, name: 'Sleep In', startTime: '23:00', endTime: '08:00', category: 'sleep' },
      { id: 102, name: 'Lazy Morning', startTime: '08:00', endTime: '10:00', category: 'myself' },
      { id: 103, name: 'Family Activities', startTime: '10:00', endTime: '18:00', category: 'family' },
      { id: 104, name: 'Relaxation', startTime: '18:00', endTime: '23:00', category: 'myself' },
    ],
  },

  // ============================================
  // TIER 2: Weekday Templates (Customizations)
  // ============================================
  weekdayTemplates: {
    'Monday':    { baseType: 'Work Day', custom: null },
    'Tuesday':   {
      baseType: 'Work Day',
      custom: [
        // Full activity list for Tuesday (copied from Work Day, then modified)
        { id: 1, name: 'Sleep', startTime: '23:00', endTime: '06:30', category: 'sleep' },  // Later!
        { id: 2, name: 'Morning Routine', startTime: '06:30', endTime: '07:15', category: 'myself' },
        { id: 3, name: 'Commute', startTime: '07:15', endTime: '08:00', category: 'else' },
        { id: 4, name: 'Deep Work', startTime: '08:00', endTime: '12:00', category: 'work' },
        { id: 5, name: 'Lunch', startTime: '12:00', endTime: '13:00', category: 'myself' },
        { id: 6, name: 'Meetings', startTime: '13:00', endTime: '17:00', category: 'work' },
        { id: 7, name: 'Commute Home', startTime: '17:00', endTime: '17:45', category: 'else' },
        { id: 8, name: 'Family Time', startTime: '17:45', endTime: '20:00', category: 'family' },
        { id: 'tue-1', name: 'Piano Practice', startTime: '20:00', endTime: '22:00', category: 'goal1' },  // NEW!
        // Note: Personal Time removed, Sleep moved to 23:00
      ]
    },
    'Wednesday': { baseType: 'Work Day', custom: null },
    'Thursday':  { baseType: 'Work Day', custom: null },
    'Friday':    { baseType: 'Work Day', custom: null },
    'Saturday':  { baseType: 'Off Day', custom: null },
    'Sunday':    { baseType: 'Off Day', custom: null },
  },

  // ============================================
  // TIER 3: Year View Overrides (Date-Specific)
  // ============================================
  dayOverrides: {
    // Geneva Public Holidays (auto-loaded)
    '2026-01-01': 'Public Holiday',  // Nouvel An
    '2026-04-03': 'Public Holiday',  // Vendredi-Saint
    '2026-04-06': 'Public Holiday',  // Lundi de Paques
    '2026-05-14': 'Public Holiday',  // Ascension
    '2026-05-25': 'Public Holiday',  // Pentecote
    '2026-08-01': 'Public Holiday',  // Fete nationale
    '2026-09-10': 'Public Holiday',  // Jeune genevois
    '2026-12-25': 'Public Holiday',  // Noel
    '2026-12-31': 'Public Holiday',  // Restauration

    // User-added overrides
    '2026-08-03': 'Vacation',
    '2026-08-04': 'Vacation',
  },

  // ============================================
  // Supporting Data
  // ============================================
  categoryColors: { ...defaultCategoryColors },
  currentView: 'daily',
  currentMainView: 'daily',  // daily, weekly, year
};
```

### View Responsibilities

#### Daily View
- **Purpose**: Edit day type templates (Work Day, Off Day patterns)
- **Shows**: Template activities for selected day type
- **Edits**: `state.templates['Work Day']` or `state.templates['Off Day']`

#### Weekly View (Outlook-style)
- **Purpose**: View/edit the actual week with weekday customizations
- **Shows**: 7 columns (Mon-Sun), each showing resolved activities for that weekday
- **Interaction**:
  - Click activity → Modal: "Edit for all [Weekday]s" or "Edit base template"
  - If "Edit for all Tuesdays" → saves to `weekdayTemplates['Tuesday'].custom`
  - If "Edit base template" → saves to `templates['Work Day']`
- **Smart Display**: If a day is marked as holiday/vacation in Year view, show Off Day template

#### Year View
- **Purpose**: Manage day type assignments and see yearly statistics
- **Shows**: 12-month calendar with color-coded days
- **Interaction**: Click day → change day type (Work, Off, Holiday, Vacation, Sick)
- **Edits**: `state.dayOverrides['2026-12-25'] = 'Vacation'`
- **KPIs**: Work Days, Public Holidays, Days Off Taken, Weekends

### Migration Path (V2 → V3)

```javascript
function migrateV2toV3(oldState) {
  return {
    // Convert old activities to templates
    templates: {
      'Work Day': oldState.activities['Work Day'] || [],
      'Off Day': oldState.activities['Off Day'] || oldState.activities['Holiday'] || [],
    },

    // Initialize weekday templates (no customizations yet)
    weekdayTemplates: {
      'Monday':    { baseType: oldState.weekAssignments['Monday'], custom: null },
      'Tuesday':   { baseType: oldState.weekAssignments['Tuesday'], custom: null },
      'Wednesday': { baseType: oldState.weekAssignments['Wednesday'], custom: null },
      'Thursday':  { baseType: oldState.weekAssignments['Thursday'], custom: null },
      'Friday':    { baseType: oldState.weekAssignments['Friday'], custom: null },
      'Saturday':  { baseType: oldState.weekAssignments['Saturday'], custom: null },
      'Sunday':    { baseType: oldState.weekAssignments['Sunday'], custom: null },
    },

    // Keep day overrides as-is
    dayOverrides: oldState.dayOverrides || {},

    // Keep other settings
    categoryColors: oldState.categoryColors,
  };
}
```

### Implementation Sprints

#### Sprint 3: Data Model Refactor
- Migrate state structure from V2 to V3
- Implement `getActivitiesForDate()` resolution function
- Add migration function for existing localStorage data
- Update `saveToStorage()` and `loadFromStorage()`

#### Sprint 4: Weekly View Enhancement
- Add Outlook-style weekly grid (time slots on Y-axis, days on X-axis)
- Click activity to open edit modal
- Modal with "Edit for this weekday" vs "Edit base template" options
- Visual indicator when viewing custom vs inherited activities

#### Sprint 5: Daily View Update
- Selector to choose: "Edit Work Day template" or "Edit Tuesday specifically"
- Preview mode: "Show me what Tuesday looks like"
- Sync indicators showing which weekdays use this template

#### Sprint 6: Cross-View Synchronization
- Changes in Weekly view update weekdayTemplates
- Changes in Daily view update templates
- Year view overrides affect Weekly view display
- Real-time KPI updates across all views

### Day Type Classification

```javascript
const DAY_TYPE_CLASSIFICATION = {
  // Work-like: Use weekday templates
  'Work Day': 'work-like',
  'Remote Day': 'work-like',

  // Off-like: Always use Off Day template
  'Off Day': 'off-like',
  'Weekend': 'off-like',
  'Public Holiday': 'off-like',
  'Vacation': 'off-like',
  'Sick Day': 'off-like',
};

function isWorkLike(dayType) {
  return DAY_TYPE_CLASSIFICATION[dayType] === 'work-like';
}
```

### Geneva Public Holidays (Reference)

Pre-loaded for years 2023-2027:

| Holiday | 2026 Date | Day |
|---------|-----------|-----|
| Nouvel An | Jan 1 | Thursday |
| Vendredi-Saint | Apr 3 | Friday |
| Lundi de Paques | Apr 6 | Monday |
| Jeudi de l'Ascension | May 14 | Thursday |
| Lundi de Pentecote | May 25 | Monday |
| Fete nationale | Aug 1 | Saturday |
| Jeune genevois | Sep 10 | Thursday |
| Noel | Dec 25 | Friday |
| Restauration de la Republique | Dec 31 | Thursday |
