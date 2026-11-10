## Frontend requirements
- Mobile-first Android experience, feels like a modern run-tracking app (similar in spirit to Nike Run Club, but with my own visual identity; do not copy branding)
- Primary interaction is tap-based, not typed chat. Use cards, sliders, toggles, and large buttons.
- Screens:
  1. Home: today's recommended plan card, large Start button, quick-adjust chips (shorter, easier, no incline)
  2. Set up plan: step-by-step selectors for time, conditions (menstrual cycle, injury, low sleep), and workout style cards
  3. Plan preview: speed and incline over time as a simple chart, with a Start button
  4. Run mode: very large speed and incline numbers, time remaining, countdown to the next change, large pause and stop buttons
  5. Summary: planned vs actual workout on one chart
  6. History and profile
- Chat is secondary: a bottom sheet for special requests with free text, opened from a button
- Work offline during a run once the plan is loaded
- Use a single aggregated endpoint for the home screen to reduce mobile network calls