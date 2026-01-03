DAILY CLOCK PLANNER - TEST USER DATA
=====================================

These JSON files contain sample planning data for testing the Daily Clock Planner.
Each file represents a different user persona with realistic daily and weekly schedules.

HOW TO USE:
-----------
1. Open the HTML file: FileSource/daily-planner-light V0.html in a browser
2. Click "Import" button in the side panel
3. Select one of the JSON files below
4. The planner will load with that user's data

Alternatively, click "Load Example Week" to load the built-in example data.

TEST USER PROFILES:
-------------------

1. sarah_developer.json
   - Persona: Software Developer (28 years old)
   - Day Types: Work Day, Remote Day, Off Day
   - Focus: Work-life balance with fitness goals, learning Rust, AWS certification
   - Schedule: Hybrid work (3 office / 2 remote days)
   - Custom colors: Blue theme for work

2. mike_freelancer_parent.json
   - Persona: Freelance Designer with 2 kids
   - Day Types: Full Workday, Half Day, Family Day, School Holiday
   - Focus: Balancing client work with active parenting
   - Schedule: Flexible around school hours
   - Custom colors: Red for work, green for family

3. alex_student.json
   - Persona: University Computer Science Student (21 years old)
   - Day Types: Class Day, Study Day, Part-Time Work, Weekend
   - Focus: Studies, part-time cafe job, fitness, coding side projects
   - Schedule: Student life with late nights
   - Custom colors: Purple for classes, red for studying

FEATURES DEMONSTRATED:
----------------------
- Multiple day types with different activity patterns
- Overnight activities (sleep spanning midnight)
- Various categories: work, family, admin, myself, goal1, goal2, goal3, else
- Custom color schemes per user
- Weekly view with day type assignments
- Time tracking across all 24 hours

EXPORT YOUR OWN DATA:
--------------------
After making changes in the planner, click "Export" to save your data as a JSON file.
This file can be imported later or shared with others.

DATA IS AUTO-SAVED:
------------------
The planner automatically saves to browser localStorage. Click "Save" to ensure
your changes are persisted, or "Clear" to reset the current day type.
