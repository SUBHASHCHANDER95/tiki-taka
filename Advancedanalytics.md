###  1: Top Scorers with xG Overperformance
```q
/ Find players scoring more than their xG suggests (clutch finishers)
topScorers:{[]
  stats:select sum goals, sum xg by player_id from player_match_stats;
  stats:update xg_diff:goals - xg from stats;
  stats:update overperformance_pct:100.0 * xg_diff % xg from stats where xg > 0;
  stats:`overperformance_pct xdesc stats;
  10#stats
 }
```

###  2: Most Creative Passers
```q
/ Find players with highest expected assists
topCreators:{[]
  stats:select sum xa, sum key_passes, sum assists by player_id from player_match_stats;
  stats:update xa_per_key_pass:xa % key_passes from stats where key_passes > 0;
  stats:`xa xdesc stats;
  10#stats
 }
```

###  3: Best Defensive Partnerships
```q
/ Find CB partnerships that concede fewest goals
bestDefensivePairs:{[min_matches]
  / Get all matches with CB pairs
  cbs:select from player_match_stats where position in `CB`LCB`RCB, minutes_played >= 60;
  
  / Group by match and team to get pairs
  pairs:select player_list:player_id by match_id, team from cbs;
  pairs:select from pairs where 2 = count each player_list;
  
  / Join with match results
  pairs_with_results:pairs lj `match_id xkey select match_id, home_team, away_team, home_score, away_score from matches;
  
  / Calculate goals conceded
  pairs_with_results:update goals_conceded:?[team=home_team; away_score; home_score] from pairs_with_results;
  
  / Aggregate by partnership
  partnership_stats:select matches_played:count i, 
                           avg_goals_conceded:avg goals_conceded,
                           clean_sheets:sum goals_conceded=0
                    by player_list from pairs_with_results;
  
  partnership_stats:select from partnership_stats where matches_played >= min_matches;
  partnership_stats:`avg_goals_conceded xasc partnership_stats;
  partnership_stats
 }
```

###  4: Formation Effectiveness
```q
/ Analyze which formations work best against each other
formationMatchups:{[]
  / Get formation for each team per match (from lineup or team data)
  matchups:select match_id, home_team, away_team, home_score, away_score,
                  home_formation:first each formation,
                  away_formation:last each formation
           from matches where status=`finished;
  
  / Calculate results
  matchups:update result:?[home_score > away_score; `home_win;
                          away_score > home_score; `away_win; `draw] from matchups;
  
  / Aggregate by formation matchup
  stats:select matches:count i,
               home_wins:sum result=`home_win,
               away_wins:sum result=`away_win,
               draws:sum result=`draw,
               avg_goals:avg home_score + away_score
        by home_formation, away_formation from matchups;
  
  stats:update home_win_pct:100.0 * home_wins % matches from stats;
  stats
 }
```

###  5: Player Fatigue Analysis
```q
/ Analyze if players perform worse when fatigued (3+ games in 7 days)
fatigueImpact:{[player_id]
  / Get all matches for player
  player_matches:select from player_match_stats where player_id=player_id;
  player_matches:player_matches lj `match_id xkey select match_id, match_date from matches;
  player_matches:`match_date xasc player_matches;
  
  / Flag congested fixtures
  player_matches:update days_since_last:?[i=0; 999; match_date - prev match_date] from player_matches;
  player_matches:update is_congested:(days_since_last <= 3) and (prev days_since_last <= 3) from player_matches;
  
  / Compare performance metrics
  comparison:select avg_rating:avg rating,
                    avg_distance:avg distance_covered,
                    avg_sprints:avg sprints
             by is_congested from player_matches;
  
  comparison
 }
```
