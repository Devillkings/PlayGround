# The Long Road Home — Campaign & Technical Notes

## Prototype scope

This repository contains a browser-playable vertical slice for Chapter 1, set near La Madeleine, Normandy, on 7 June 1944. The canvas prototype is a presentation and gameplay mock-up; it is not a Unity build. The dossier in the game summarizes the larger campaign and business model.

## Campaign outline

A ten-mission, three-act story follows a platoon from the scattered aftermath of the Normandy landings through the Falaise pocket and toward a quiet farm. The central themes are the cost of survival, the bonds between veterans and replacements, and how ordinary people carry loss home. Radio exchanges, field-journal panels, mission events, and letters between chapters provide the narrative rather than long interruptions to play.

| # | Mission | Act / story beat |
|---|---|---|
| 1 | First light at La Madeleine | I — Regroup, hold the crossroads, and open a route for medics. *(Playable prototype.)* |
| 2 | The drowned lane | I — Find a missing element through flooded fields. |
| 3 | Letters from Sainte-Mère | I — Escort medics through a besieged village. |
| 4 | The orchard ambush | II — Survive a close-range ambush and identify its source. |
| 5 | A bridge in the dark | II — Reconnoiter and demolish a bridge before dawn. |
| 6 | Steel in the bocage | II — Coordinate infantry and armor in a costly duel. |
| 7 | The long retreat | II — Withdraw while protecting wounded soldiers and replacements. |
| 8 | Falaise road | III — Break through a congested, contested route. |
| 9 | No one left behind | III — Choose a rescue plan under severe time pressure. |
| 10 | The quiet farm | III — Reach safety, account for survivors, and read the names of the missing. |

Each mission targets 10–15 minutes, with a clear primary objective, optional tactical opportunities, a mid-mission checkpoint, and a short narrative debrief. The campaign concept carries surviving squad members, injuries, veterancy, and equipment forward. Fallen soldiers are remembered in the roster and story; they are not monetized or used as a retry penalty.

## Unity implementation direction

For a production mobile/desktop game, use Unity with URP and a fixed-step simulation. Separate simulation state from rendering and UI so that tactical pause, save/reload, and deterministic replays do not depend on frame rate.

- **Mission layer:** authored objectives, triggers, reinforcement schedules, checkpoint snapshots, and narrative/radio events.
- **Squad layer:** finite-state machine (`Idle`, `Move`, `TakeCover`, `Engage`, `Suppress`, `Pinned`, `Retreat`, `Disabled`) with orders queued while paused.
- **Navigation:** NavMesh for traversable space, with a cover-point graph layered on top. Score candidate cover by protection, distance, approach cost, flanking exposure, and squad formation constraints.
- **Combat:** fixed-tick weapon resolution with range, line of sight, weapon accuracy, cover, suppression, and morale modifiers. Use a seeded random stream per mission for repeatable results.
- **Presentation:** URP lighting and restrained effects; UI Toolkit or uGUI for scalable HUD; input abstraction that maps touch, mouse, and controller to the same selection/order commands.
- **Persistence:** versioned campaign save containing mission checkpoint, squad roster, injuries, veterancy, gear, and completed objectives. Write atomically and retain a previous valid save.
- **Mobile performance:** pool effects/projectiles, cap simultaneous audio voices and particles, use LODs and baked lighting where practical, and profile on target devices before increasing battlefield density.

### Cover and suppression pseudocode (C#-style)

```csharp
void IssueMove(Vector3 destination)
{
    var candidates = coverGraph.Within(destination, searchRadius)
        .Where(point => navMesh.IsReachable(squad.Position, point.Position));

    var chosen = candidates
        .Select(point => new {
            Point = point,
            Score = point.Protection * protectionWeight
                  - PathCost(squad.Position, point.Position) * distanceWeight
                  - EnemyFlankExposure(point, knownEnemies) * flankWeight
                  - FormationPenalty(point, squad.Members) * formationWeight
        })
        .OrderByDescending(candidate => candidate.Score)
        .FirstOrDefault();

    squad.Order = chosen != null && chosen.Score >= minimumCoverScore
        ? SquadOrder.MoveToCover(chosen.Point)
        : SquadOrder.MoveTo(destination);
}

void TickSuppression(float fixedDeltaTime)
{
    foreach (var member in squad.Members.Where(member => member.IsAlive))
    {
        float incoming = combatEvents.SuppressionFor(member);
        float coverFactor = member.CurrentCover?.SuppressionFactor ?? 1f;
        member.Suppression = Mathf.Clamp01(
            member.Suppression + incoming * coverFactor
            - recoveryPerSecond * fixedDeltaTime);
    }

    if (squad.AverageSuppression >= pinnedThreshold)
    {
        squad.State = SquadState.Pinned;
        squad.CancelUnsafeAdvance();
        squad.AccuracyMultiplier = pinnedAccuracyMultiplier;
        squad.TryFindSaferCover();
    }
    else if (squad.State == SquadState.Pinned &&
             squad.AverageSuppression < recoverThreshold)
    {
        squad.State = squad.HasActiveOrder ? SquadState.Engage : SquadState.Idle;
    }
}
```

In production, evaluate exposure from threat direction and line-of-sight rather than treating cover as a binary radius. Use hysteresis between pinned/recovered thresholds to prevent rapid state flicker, and communicate state changes through clear animation, audio, and HUD feedback.

## Controls and fair monetization

The prototype supports tap/click selection, Move, Attack, Suppress, tactical pause (Space), speed controls, restart, and a campaign dossier. A full mobile release should provide generous touch targets, optional drag selection, and a command queue while paused.

Recommended model: a free first chapter followed by a one-time full-campaign unlock. Optional cosmetics may be offered if they do not obscure battlefield readability. No paid combat power, random paid rewards, energy timers, or purchasable retries.

## Prototype assumptions and limitations

- Normandy in June 1944 is the assumed setting; characters and events are fictional and the story is not a representation of a specific real company.
- Only Mission 1 is playable. The rest of the missions, squad persistence, and checkpoint persistence are design targets, not implemented campaign systems.
- The current web simulation is intentionally lightweight: it uses simplified cover, combat, and suppression rules and does not contain Unity code or a real navigation mesh.
