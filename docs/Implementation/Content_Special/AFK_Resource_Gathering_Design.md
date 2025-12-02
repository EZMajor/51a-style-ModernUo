# AFK Resource Gathering Deep Dive

## Overview
AFK gold farming illegal/monitored; AFK crafting legal/unchecked; AFK resource gathering (mining/lumber/etc.) allowed with hourly CAPTCHA (30-45s). Missed CAPTCHA: 65% rate decrease for 1 hour; answered: normal rate. Favors active players with better gains.

## Algorithms and Logic
Hourly CAPTCHA prompts (probabilistic/periodic); rate multipliers (normal: 1x, decreased: 0.35x); crafting bonus for active completions.

## Edge Cases
CAPTCHAs not prompted if offline; rate resets on login; exploit detection logs.

## Implementation Details
Hook into harvest system (harvest restart on CAPTCHA expire); gump/web CAPTCHA; persistent rate state. Gold farming monitoring via anomalies.

## Testing Plan
CAPTCHA triggers, rate transitions, active vs. AFK comparative gains.

## Change Log
Filled with CAPTCHA-based anti-AFK, favoring presence for benefits.
