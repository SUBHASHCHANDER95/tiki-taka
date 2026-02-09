# tiki-taka
brief of player and game analysis
**Key Focus Areas:**
- xG (Expected Goals) modeling and analysis
- Player performance tracking & heat maps
- Tactical formation analysis
- Team momentum detection
- Match outcome predictions
- Transfer market valuation



### Core Tables Schema

#### 1. **Matches Table**
```q
matches:([]
  match_id:`symbol$();
  match_date:`date$();
  match_time:`time$();
  competition:`symbol$();        / EPL, UCL, LaLiga, etc.
  season:`symbol$();             / 2025-2026
  home_team:`symbol$();
  away_team:`symbol$();
  home_score:`int$();
  away_score:`int$();
  home_xg:`float$();             / Expected goals
  away_xg:`float$();
  status:`symbol$();             / scheduled, live, finished
  attendance:`int$();
  venue:`symbol$();
  referee:`symbol$();
  weather:`symbol$()             / sunny, rainy, cold, etc.
 )
```

#### 2. **Match Events Table** (High-Frequency Data)
```q
match_events:([]
  event_id:`symbol$();
  match_id:`symbol$();
  timestamp:`timestamp$();        / Precise event timing
  minute:`int$();
  second:`int$();
  period:`symbol$();              / 1H, 2H, ET, PK
  team:`symbol$();
  player_id:`symbol$();
  event_type:`symbol$();          / pass, shot, tackle, foul, card, sub, goal
  event_outcome:`symbol$();       / successful, unsuccessful, blocked
  x_coord:`float$();              / 0-100 (pitch coordinates)
  y_coord:`float$();              / 0-100
  end_x:`float$();                / For passes/shots
  end_y:`float$();
  body_part:`symbol$();           / left_foot, right_foot, head
  assist_type:`symbol$();         / through_ball, cross, corner, etc.
  xg_value:`float$();             / For shots only
  under_pressure:`boolean$()      / Was player under pressure?
 )
```

#### 3. **Player Stats Table** (Per Match)
```q
player_match_stats:([]
  match_id:`symbol$();
  player_id:`symbol$();
  team:`symbol$();
  position:`symbol$();            / GK, CB, LB, RB, CDM, CM, CAM, LW, RW, ST
  minutes_played:`int$();
  goals:`int$();
  assists:`int$();
  xg:`float$();                   / Expected goals
  xa:`float$();                   / Expected assists
  shots:`int$();
  shots_on_target:`int$();
  key_passes:`int$();
  passes_attempted:`int$();
  passes_completed:`int$();
  pass_accuracy:`float$();
  progressive_passes:`int$();     / Passes that move ball forward significantly
  crosses:`int$();
  dribbles_attempted:`int$();
  dribbles_successful:`int$();
  touches:`int$();
  touches_box:`int$();            / Touches in opponent box
  tackles:`int$();
  interceptions:`int$();
  clearances:`int$();
  blocks:`int$();
  duels_won:`int$();
  duels_lost:`int$();
  aerials_won:`int$();
  fouls_committed:`int$();
  fouls_won:`int$();
  yellow_cards:`int$();
  red_cards:`int$();
  offsides:`int$();
  distance_covered:`float$();     / in km
  sprints:`int$();
  top_speed:`float$()             / km/h
 )
```

#### 4. **Players Metadata**
```q
players:([]
  player_id:`symbol$();
  full_name:`symbol$();
  short_name:`symbol$();
  nationality:`symbol$();
  date_of_birth:`date$();
  age:`int$();
  height:`float$();               / in cm
  weight:`float$();               / in kg
  preferred_foot:`symbol$();      / left, right, both
  current_team:`symbol$();
  position:`symbol$();
  shirt_number:`int$();
  market_value:`float$();         / in millions
  contract_expiry:`date$();
  injury_status:`symbol$()        / fit, doubtful, injured, suspended
 )
```

#### 5. **Teams Metadata**
```q
teams:([]
  team_id:`symbol$();
  team_name:`symbol$();
  short_name:`symbol$();
  country:`symbol$();
  league:`symbol$();
  founded:`int$();
  stadium:`symbol$();
  manager:`symbol$();
  formation:`symbol$();           / 4-3-3, 4-4-2, 3-5-2, etc.
  avg_possession:`float$();
  avg_goals_scored:`float$();
  avg_goals_conceded:`float$();
  avg_xg_for:`float$();
  avg_xg_against:`float$()
 )
```

#### 6. **Team Season Stats**
```q
team_season_stats:([]
  team:`symbol$();
  season:`symbol$();
  competition:`symbol$();
  matches_played:`int$();
  wins:`int$();
  draws:`int$();
  losses:`int$();
  goals_for:`int$();
  goals_against:`int$();
  goal_difference:`int$();
  points:`int$();
  xg_for:`float$();
  xg_against:`float$();
  clean_sheets:`int$();
  possession_avg:`float$();
  pass_accuracy:`float$();
  shots_per_game:`float$();
  tackles_per_game:`float$();
  ppda:`float$()                  / Passes Allowed Per Defensive Action (pressing metric)
 )
```

#### 7. **Passing Networks Table** (Advanced)
```q
passing_networks:([]
  match_id:`symbol$();
  team:`symbol$();
  passer_id:`symbol$();
  receiver_id:`symbol$();
  pass_count:`int$();
  successful_passes:`int$();
  avg_pass_length:`float$();
  progressive_passes:`int$()
 )
```
