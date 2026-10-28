# Trials desktop

This solution references `App2d.Core` from an existing App2d checkout. It does not contain a copy of Core.

Requires Windows and the .NET 10 SDK. By default, place this repository beside the `App2d` checkout:

```text
source/
  App2d/
    App2d.Core/
  trials/
    Trials.slnx
```

```powershell
dotnet build Trials.slnx
dotnet test App2d.Trials.Tests
dotnet run --project App2d.Trials
```

For another checkout location, add `-p:App2dRoot=C:/path/to/App2d` to these commands. Core's NuGet packages and build analyzers are restored from that checkout.

`App2d.Trials` is a separate, minimal trials-game prototype. Run it with
`dotnet run --project App2d.Trials`. It uses the shared `GameHost`, `Game2D` camera,
and physics world to simulate a double-triangle bike on a cubic B-spline terrain course.
Use **A/D** or **Left/Right** to drive, **W/S** to lean, **Space** to brake,
**R** to reset, and **Esc** to close. The camera follows the terrain height; surface marks show movement.
Three repeating depth layers provide clouds, distant mountains, and nearby hills;
both games use the shared [parallax layer API](https://github.com/raymondBporter/App2d/blob/main/docs/rendering-passes.md#parallax-layers).
No external assets are needed. `TrialsBike2D` holds the editable tube geometry, fork,
wheel radius, motor and suspension tuning. The front triangle and rear swingarm are
separate rigid bodies. A front wheel joint allows 28 units of travel along the fork;
a rear pivot and wheel bearing keep the rear triangle connected while its shock
compresses up to 14 units. Wheels rotate freely and rear-wheel torque drives through
ground friction. The fork telescopes and the rear shock graphic follows its mounts.

`TrialsCourse2D` holds the course `Width`, `FrequencyAmplitudes` and additive `Ramps`.
Change `Width` in that file to grow the default map, or pass a different configuration to
`TrialsGame2D(course)` / `TrialsTerrain2D(world, course)`. Width is measured from `XMin`
to `XMax = XMin + Width`, including the area left of the spawn. Frequencies are cycles
per world unit (`1 / wavelength`), amplitudes are heights in world units, and ramps use
world-space X/Y points. Increasing width adds more waves without stretching their scale
or moving authored ramps. The final wave fade follows the new right endpoint.

```csharp
var course = TrialsCourse2D.Default with
{
    Width = 40_000f,
    FrequencyAmplitudes = [(1f / 2400f, 48f), (1f / 700f, 20f)],
    Ramps = [PolylineCurve2D.Builder.Create([new(2000, 0), new(2200, 120), new(2400, 0)])]
};
using var game = new TrialsGame2D(course);
```

`ControlPointSpacing`, `YMin`, `WaveStartX` and wave fade lengths are also configurable.
The control-point grid remains anchored when width changes, with an exact final endpoint
even when the width is not a multiple of the spacing. Empty wave/ramp collections are valid.
The default map keeps the three waves and two hand-shaped ramp profiles.
The clamped cubic `BSpline2D` has continuous position, first and second
derivatives at its simple interior knots; its control points influence the curve rather
than being interpolation targets. `Curve2D.ToPolyline` samples each knot span adaptively,
with a 12-unit chord limit and 0.05-unit sampled-deviation target. Visible and collidable
terrain use the same sampled boundary. `SimplePolygon2D.Builder.BelowPolyline` closes
the region down to a bottom plane and builds narrow strip triangles directly. This is a
finite default course of width 42800 from x = -1600 to 41200, with a flat starting area.


Render a diagnostic frame sequence with `dotnet run --project App2d.Trials -- --render-smoke artifacts/smoke`.
