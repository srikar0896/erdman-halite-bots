# erdman-halite-bots

Bots from the Halite.io competition

This repository includes three different versions of my bot that may be of interest to others.

* v12 was live on the site for most of December 2016.  If memory serves, I think a close ancestor bot had already unseated @djma, and this bot submission was the response to @timfoden's unseating of my previous version.  This bot remained on the leaderboard for a couple weeks or so, until @nmalaguti developed his strength innovation and took over the leaderboard.
* v17 was my response to @nmalaguti, increasing my own strength-building and overall Halite street-fighting capabilities.  This was my bot that ran for most of January 2017 and into the first week of February.
* v26 is my final entry.  It includes two important new features and a feeble attempt at dealing with NAP.

## code notes

### `hlt.py`
Unmodified ("stock") from the alt-python3-halite-starter that I wrote, which later became the official Python starter package.  As result, all the interesting bot code is self-contained within the `MyBot.py` files (renamed `erdman_vXX.py` here).

### `erdman_v12.py`

The key idea is a single "potential field" map (called `pf_map` in the code) that indicates where every square should want to move. Strength-divided-by-production was the valuation measure I cared about; so, lower scores are better -- like water, the squares want to flow downhill.  Generated by a Dijkstra-style search over `initial_potential` (strength/production) of the the map squares.  As the lowest-potential squares are pulled off the min-priority queue, its neighbors are added to the queue with a potential that is the exponentially-weighted-average of the potential of the square just pulled and the strength/production of the neighbor square.  This causes squares on the path to the very best squares on the map to have lower (better) scores than they would have if just scored on their standalone strength/production.  While I only intended the bot to favor moving towards the best mining areas, this in fact creates the observed tunneling behavior.

Combat squares are also included on the map, with a negative weight multiplied by the number of enemies bordered, in the hopes of maximizing overkill.  Remember, lower is better, so combat squares with negative weights have lowest potential of all.

My own squares do not receive an initial_potential.  When the border to my squares is reached while moving through the priority queue items, the exponentially weighted average process stops, and the scores are decayed (in this case, increased) by a function of the squared walking distance to the border.  The coefficient of the distance^2 was tuned to balance local action vs distant action.  This turned out to be a particular strength of my bots, as it did not overcommit to combat and continued mining when sufficiently far from the combat.  It was not uncommon to hold-the-line or even lose the battle, but to win the war later in the game as a result of the increased production that the continued mining would provide.

Squares are assigned a move in descending order of strength.  `destinations` tracks the amount of strength already moved to a square, so remaining move assignments can minimize cap loss.

Lines 44-45 resulted from my realization that squares moving together side-by-side or front-to-back were unusually susceptible to overkill.  So, I did not allow squares to do that.  This resulted in the (in)famous "checkerboard" pattern.

The argparse constants are particularly important:
* `alpha` is the weighting used in the exponentially weighted average of potentials
* `potential_degradation_step` is a bit of a misnomer ... it's the coefficient of the distance^2 decay factor.
* `enemy_ROI` is the initial_potential for each bordered enemy in combat squares
* `hold_until` is the multiple of production that must be reached before a square is allowed to move.

When tuning these constants, the first three must vary together (so I learned with hard experience).  They interact with one another ... lowering `alpha` will allow the bot to "see" further for better mining, but will result in overall lower border potential values, so the distance^2 decay factor coefficient and the `enemy_ROI` must also be adjusted to keep everything in balance.

### `erdman_v17.py`

Two primary ideas were added to this version.

#### strength_hurdle
Lines 103 - 106.  @nmalaguti figured out that substantially increased strength at the cost of a little production was a big strategic weapon, and it was then evident that my bot moved too much.  I imagined that a territory of a given size (number of squares) had some optimal percentage of squares that should be moving, and I guessed that the percentage decreased with the area of the territory, in the same way that the relative length of the perimeter (border) of a circle decreases with its area.  The largest territory that I would ever have is 50x50, so the percentage of my squares to move is based on that area as a maximum.  Like all other parameters, the `int_max` and `int_min` argparse defaults were tuned empirically.  Square strength has to exceed both this percentile-based strength hurdle as well as the old-school style `hold_until` multiple-of-production hurdle.

#### dangerous_empties
Lines 42 - 53.  It dawned on me that most overkill is self-inflicted, that overkill isn't so much something you do to an opponent, rather something you should avoid letting your bot do to itself.  A `dangerous_empty` is a combat-empty (enemy adjacent) that another of my own squares has already commited to moving into or adjacent.  The square whose move is currently being assigned should avoid also moving (or stilling) adjacent to a `dangerous_empty` to avoid being subject to overkill.  This overkill "dodge" behavior allowed me to remove the code that caused "checkerboard".  This removal, in turn, seemed to improve my interior logistics of moving squares efficiently to the border.

Additionally, my argparse parameter defaults were tuned, which materially improved my bot's performance.  Alpha was lowered (allowing it to "see" much farther), and the others were adjusted to maintain balance, as described above.

### `erdman_v26.py`

After the back-and-forth that was experienced atop the leaderboard in December and January, it became evident that good ideas were easy to replicate among the top bots.  I thought I still had some good ideas left to experiment with, and I implemented them locally towards the end of January and found a couple in particular to be very effective.  However, this time, I didn't want to show my hand until the last minute in order to hold an advantage heading into the finals.  I'd keep surprise on my side ... little did I realize that other sneaky Haliters had similar plans of their own!

If you watched much of my bots prior to the finals, you didn't see these two things; they are only in the final release:

#### red_green trees
Lines 140 - 173.  You know red-black trees?  These aren't them.  These trees have redlights and greenlights on them.  From each terminal border square (ie the border squares that have low enough potential that at least one interior square wants to move there), a tree is built of all squares that would want to flow to this border square if allowed.  The tree is walked to count strengths and productions to find which squares need to STILL (red_light) or move (green_light) to capture the border square in the fewest possible turns.  Lot of other bots had this or something similar, but I didn't, and yet my exploration had been pretty good anyway.  The red_green trees make my exploration and mining that much faster.

#### strategic stilling
Lines 55 - 60.  If a square can safely STILL and has more than one adjacent combat empty and more than one unique enemy adjacent to those combat empties, it STILLS.  Many bots walk into overkill with great frequency, and this exploits that bias.  I saw that @mzotkiew and @shummie also developed this strategy for the finals.


## non-aggression pact (nap)
I played along with the development of this strategy in the final week.  At first, this was to the benefit of my bot, because I already had a good exploration strategy, and with non-aggression, my bot was free to gobble up territory.  With the red_green trees in my back-pocket, I was optimistic about my chances in the finals.  

The NAP created a disadvantage in multi-player matches for those that did not cooperate, as they would frequently be in combat with all neighbors, while the NAP bots would only be fighting with their non-NAP neighbors.  Non-NAP's would be quickly eliminated due to resource exhaustion from fighting on multiple fronts, allowing the NAP bots to claim the top finishes for themselves.

However, soon enough, most of the top 10 bots had implemented a form of NAP, and the final 24 (12?) hours of the open competition saw an explosion of innovation to take advantage of a situation where a game full of NAP bots do not attack each other at all ... until the final few moves of the game.  The "zit popping" strategy (strength is built up within the confines of the controlled space, and then bursts outwards in the last few moves, to claim a bit more territory away from the other NAP'ers) in particular has proven effective.

I did not have the time nor tools really to deal with this in the final hours.  I hoped that I could participate in NAP until I've captured all available space, but I knew I'd be at a real disadvantage to the zit-poppers if the game were allowed to continue to the move cap.  My intention was to resume combat after I'd taken all available space, but in the short time remaining, I botched the implementation.  Specifically, line 120 is bugged and should be `seen_enemies.update(neighbor.owner for empty in hero_empties for neighbor in game_map.neighbors(empty)`; also, tuning the completely-out-of-thin-air magic number in line 97 could have helped raise my propensity to jail-break when available mining was depleted.  As a result of these bugs, my bot starts a fight through just a single square, which has no effect at all given the great strengths on either sides of the wall.

Finally, in the last 30 minutes prior to the close of submissions, I debated whether to disable NAP entirely when down to final two players.  I decided yes, but determining the number of players remaining required another pass through the entire game_map, which after some frantic testing, threatened to timeout my bot on the largest maps, which wasn't worth the risk, so I abandoned the plan.  In that mild panic, I missed the easy opportunity to disable NAP for heads-up matches (starter bot code already counts number of starting players in the initialization period), which has quite likely cost me some heads-up matches in the finals.  I would take my bot against all comers in an old-fashioned Halite street fight!

The meta-meta game that developed in those last hours was really amazing, and I didn't see it coming.  Kudos to @timfoden and @mzotkiew that had the brilliance to develop and implement really great NAP strategies.  It isn't necessarily always pretty :), but it looks like a winner.  Congrats!

## thank you's

@truell20, @Sydriax, and @TwoSigma for the terrific competition.  It was great fun.  I've waited five years for a worthy successor to Ants, and Halite was it.

@janzert for helping keep sand out of the gears of the great machinery that runs the competition.

@nmalaguti for showing this newb how to suck a little less at git.  Saved my bacon that you took the time to do that.  It served me throughout, and I couldn't have been near as productive or competitive in Halite without those git basics.

And thanks to all that participated for raising the level of competition in Halite.  I had a blast!
