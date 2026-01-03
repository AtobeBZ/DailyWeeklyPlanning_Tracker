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
