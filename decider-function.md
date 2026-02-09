### 1. **Expected Goals (xG) Calculator**
```q
/ Calculate xG for a shot based on multiple factors
calcXG:{[shot]
  / Factors: distance from goal, angle, body part, assist type, pressure
  distance:sqrt[(shot`x_coord - 100) xexp 2] + (shot`y_coord - 50) xexp 2;
  angle:atan[(shot`y_coord - 50) % (100 - shot`x_coord)];
  
  / Base xG from distance
  baseXG:0.95 * exp[-0.1 * distance];
  
  / Modifiers
  angleModifier:1.0 + (0.3 * abs[angle] % 90);
  bodyModifier:$[shot`body_part=`head; 0.7;
                  shot`body_part=`left_foot; 1.0;
                  shot`body_part=`right_foot; 1.0; 1.0];
  assistModifier:$[shot`assist_type=`through_ball; 1.2;
                    shot`assist_type=`cross; 0.8;
                    shot`assist_type=`corner; 0.6; 1.0];
  pressureModifier:$[shot`under_pressure; 0.8; 1.0];
  
  / Final xG
  finalXG:baseXG * angleModifier * bodyModifier * assistModifier * pressureModifier;
  finalXG
 }

/ Apply to all shots in a match
updateMatchXG:{[match_id]
  shots:select from match_events where match_id=match_id, event_type=`shot;
  update xg_value:calcXG each shots from `match_events where match_id=match_id, event_type=`shot;
 }
```

### 2. **Player Performance Rating (0-10 scale)**
```q
/ Calculate match rating like WhoScored/SofaScore
calcPlayerRating:{[stats]
  / Weight different actions
  goalWeight:3.0;
  assistWeight:2.0;
  shotWeight:0.2;
  passWeight:0.01;
  tackleWeight:0.3;
  interceptionWeight:0.2;
  duelWeight:0.1;
  
  rating:6.0;  / Base rating
  rating:rating + (stats`goals * goalWeight);
  rating:rating + (stats`assists * assistWeight);
  rating:rating + (stats`shots_on_target * shotWeight);
  rating:rating + (stats`passes_completed * passWeight);
  rating:rating + (stats`tackles * tackleWeight);
  rating:rating + (stats`interceptions * interceptionWeight);
  rating:rating + (stats`duels_won * duelWeight);
  
  / Penalties for negative actions
  rating:rating - (stats`fouls_committed * 0.1);
  rating:rating - (stats`yellow_cards * 0.5);
  rating:rating - (stats`red_cards * 2.0);
  rating:rating - ((stats`passes_attempted - stats`passes_completed) * 0.005);
  
  / Clamp between 1.0 and 10.0
  rating:max[1.0; min[10.0; rating]];
  rating
 }

/ Get top performers in a match
topPerformers:{[match_id]
  stats:select from player_match_stats where match_id=match_id, minutes_played > 0;
  stats:update rating:calcPlayerRating each stats from stats;
  stats:`rating xdesc stats;
  10#stats
 }
```

### 3. **Possession Calculation**
```q
/ Calculate possession % for each team in a match
calcPossession:{[match_id]
  events:select from match_events where match_id=match_id;
  teamPasses:select pass_count:count i by team from events where event_type=`pass;
  teamPasses:update possession_pct:100.0 * pass_count % sum pass_count from teamPasses;
  teamPasses
 }
```

### 4. **Form Analysis (Last 5 Matches)**
```q
/ Get team's recent form
getTeamForm:{[team;n]
  recentMatches:select from matches where (home_team=team) or (away_team=team), status=`finished;
  recentMatches:`match_date xdesc recentMatches;
  recentMatches:n#recentMatches;
  
  / Calculate points
  results:update result:?[(home_team=team) & (home_score > away_score); `W;
                          (away_team=team) & (away_score > home_score); `W;
                          (home_score = away_score); `D; `L] from recentMatches;
  results:update points:?[result=`W; 3; result=`D; 1; 0] from results;
  
  ([] team:team; form:results`result; total_points:sum results`points; matches:count results)
 }

/ Example: getTeamForm[`ManCity; 5]
```

### 5. **Player Heat Map Data**
```q
/ Generate heat map data for player's touches
getPlayerHeatMap:{[match_id;player_id]
  touches:select minute, x_coord, y_coord from match_events 
    where match_id=match_id, player_id=player_id, 
          event_type in `pass`shot`dribble`tackle;
  
  / Bin coordinates into grid (10x10)
  touches:update x_bin:floor[x_coord % 10], y_bin:floor[y_coord % 10] from touches;
  
  / Count touches per bin
  heatMap:select touch_count:count i by x_bin, y_bin from touches;
  heatMap
 }
```

### 6. **Progressive Passing Analysis**
```q
/ Identify progressive passes (moves ball significantly forward)
isProgressivePass:{[pass]
  / Progressive if moves ball 10+ yards toward goal and closer to goal
  forward_distance:(pass`end_x) - (pass`x_coord);
  distance_to_goal_before:100 - pass`x_coord;
  distance_to_goal_after:100 - pass`end_x;
  
  (forward_distance >= 10.0) and (distance_to_goal_after < distance_to_goal_before)
 }

/ Get progressive passing leaders
getProgressivePassers:{[match_id]
  passes:select from match_events where match_id=match_id, event_type=`pass, event_outcome=`successful;
  passes:update is_progressive:isProgressivePass each passes from passes;
  
  leaders:select progressive_count:sum is_progressive by player_id, team from passes;
  leaders:`progressive_count xdesc leaders;
  leaders
 }
```

### 7. **Head-to-Head Analysis**
```q
/ Compare two teams' historical record
getH2H:{[team1;team2;n]
  h2h:select from matches where 
    ((home_team=team1) and (away_team=team2)) or 
    ((home_team=team2) and (away_team=team1)),
    status=`finished;
  
  h2h:`match_date xdesc h2h;
  h2h:n#h2h;
  
  / Calculate wins for each team
  h2h:update winner:?[(home_score > away_score) and (home_team=team1); team1;
                      (away_score > home_score) and (away_team=team1); team1;
                      (home_score > away_score) and (home_team=team2); team2;
                      (away_score > home_score) and (away_team=team2); team2;
                      `Draw] from h2h;
  
  summary:([] team1_wins:sum h2h[`winner]=team1;
              team2_wins:sum h2h[`winner]=team2;
              draws:sum h2h[`winner]=`Draw;
              avg_goals_team1:avg ?[h2h[`home_team]=team1; h2h`home_score; h2h`away_score];
              avg_goals_team2:avg ?[h2h[`home_team]=team2; h2h`home_score; h2h`away_score]);
  
  (summary; h2h)
 }
```

### 8. **Match Momentum Detector**
```q
/ Detect momentum swings in a match (dangerous periods)
detectMomentum:{[match_id]
  events:select from match_events where match_id=match_id;
  
  / Group events into 5-minute windows
  events:update time_bin:5 * floor[minute % 5] from events;
  
  / Count dangerous events (shots, key passes) per team per window
  danger:select shot_count:sum event_type=`shot,
                key_pass_count:sum event_type=`key_pass
         by time_bin, team from events;
  
  / Calculate momentum score
  danger:update momentum_score:(shot_count * 2.0) + key_pass_count from danger;
  
  danger
 }
```

### 9. **Pressing Intensity (PPDA)**
```q
/ Calculate Passes Allowed Per Defensive Action (lower = more pressing)
calcPPDA:{[team;match_id]
  / Get opponent passes in attacking third
  opponent_team:first exec ?[home_team=team; away_team; home_team] from matches where match_id=match_id;
  
  opponent_passes:select from match_events where match_id=match_id, team=opponent_team, 
                   event_type=`pass, x_coord < 66.67;  / Attacking third
  
  / Get defensive actions by team
  def_actions:select from match_events where match_id=match_id, team=team,
               event_type in `tackle`interception`foul;
  
  ppda:(count opponent_passes) % (count def_actions);
  ppda
 }
```

### 10. **Win Probability Model**
```q
/ Predict match outcome based on pre-match stats
predictMatchOutcome:{[home_team;away_team]
  / Get recent form
  home_form:getTeamForm[home_team; 5];
  away_form:getTeamForm[away_team; 5];
  
  / Get season stats
  home_stats:select from team_season_stats where team=home_team, season=`2025_2026;
  away_stats:select from team_season_stats where team=away_team, season=`2025_2026;
  
  / Simple model factors
  home_advantage:1.3;  / Home teams win ~45% of matches
  
  / Calculate strength based on xG difference
  home_strength:(first home_stats`xg_for) - (first home_stats`xg_against);
  away_strength:(first away_stats`xg_for) - (first away_stats`xg_against);
  
  / Form impact
  form_impact:(first home_form`total_points) - (first away_form`total_points);
  
  / Combined score
  home_score:(home_strength * home_advantage) + (0.1 * form_impact);
  away_score:away_strength - (0.1 * form_impact);
  
  / Convert to probabilities (softmax)
  draw_score:0.5 * (home_score + away_score);  / Draw probability
  
  total:exp[home_score] + exp[away_score] + exp[draw_score];
  home_prob:100.0 * exp[home_score] % total;
  away_prob:100.0 * exp[away_score] % total;
  draw_prob:100.0 * exp[draw_score] % total;
  
  ([] outcome:(`home_win; `draw; `away_win); 
      probability:(home_prob; draw_prob; away_prob))
 }
```
