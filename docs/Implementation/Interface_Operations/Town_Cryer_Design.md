# Town Cryer Deep Dive

## Overview
Town cryer NPCs at banks in faction towns and Britain announce automatic events (sieges, dungeons, tournaments) when active; clickable for gump listing running events.

## Algorithms and Logic
Event listeners trigger yells; clickable response opens event gump; no global chat, localized near banks to enhance immersion.

## Edge Cases
Visibility range filters; no spamming on inactive events.

## Implementation Details
NPC placement at banks; event hooks for speech/gump; anti-spam timers.

## Testing Plan
Event sequence triggers, player interaction testing with range checks.

## Change Log
Filled with immersion-focused yelling, bank localization, event awareness promotion.
