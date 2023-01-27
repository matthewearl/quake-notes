# Platform boost

There is a strange bug on E2M2, just beyond the gold key door.  If you position
yourself just right, you'll get 100 points of damage and a massive boost into
the air.

What's actually happening is that due to some quirks in the collision code the
player sinks into the platform.

When the platform's physics function `SV_PushMove` is called, it detects that
the player is intersecting the platform and attempts to push them down.
Since the player is on top of the platform this doesn't work, so the function
resolves to apply the platform's (a `func_door` in this case) blocked function.
This is the same function that would be applied if the player were being
squished under the platform, which does 100 damage to the player and an
equivalent speed boost.

But why does the player sink into the platform in the first place?

To understand this, we must delve into Quake's physics code.  The story starts
off in `SV_FlyMove`.  This funcion takes a player's position and velocity, and
integrates them for the duration of a frame.  By this I mean it moves the player
in the direction of their velocity the appropriate distance, and resolves any
collisions that it encounters.

For the bug in question, we are moving along a flat surface.  Our velocity is
mostly forwards, with a small downwards component owing to the effects of
gravity.  What happens in this situation is `SV_FlyMove` performs a downward
trace into the level geometry.  Since we're on the ground the trace pretty
quickly intersects with the ground, and so the function removes the vertical
component from the player's velocity and performs the rest of the movement
parallel to the ground.

Note that the movement due to the lift is separate --- for the purposes of the
player's `SV_FlyMove` call the platform is stationary.  The lift simply shifts
down to match the movements of the player every frame.

You can imagine that due to floating point errors this might immediately end up
with the player slightly intersecting with the platform.  `SV_Move`, the
function that performs tracing into the world, has a counter measure for that.
Whenever it hits a surface, it reduces the distance travelled such that the
player is separated by a distance of 1/32 (0.03125) of a unit from the surface.

It is when this back-off logic fails that the bug occurs.  But first, a bit of
background on `SV_Move` and collision hulls.  `SV_Move` solves the problem of
tracing a bounding box through space and working out the point just before where
the bounding box first collides with the world.  Tracing a box is equivalent to
making the brushes of the level
[a big larger](https://en.wikipedia.org/wiki/Minkowski_addition) and tracing a
point through the level, which is significantly easier.

Quake's map compiler produces collision hulls for various different "inflations"
of the base geometry --- one for each bounding box size.  Each moving part of
the level gets its own set of collision hulls.  In this case we're interested in
the collision hull for the E2M2 lift platform that corresponds with the player's
bounding box.

Collision hulls are represented as BSP trees.  Let's illustrate with a simple
2D example that is enough to illustrate what's going on with the E2M2 lift.
BSP trees, or binary space partition trees, are a recursive splitting of space
into two.  The splits are planar (or linear) in 2D, and in Quake normally
correspond with surfaces of the object being partitioned.  The leaves of the
tree are labelled as either solid or empty.

A trace entails recursively walking through this tree in distance order until a
solid leaf is encountered.  Starting at the root node:
- If the line segment being traced is entirely on one side of the splitting
  plane, recurse into the child node on that side.
- If the line segment spans the splitting plane, recurse into the nearest child
  first, then the furthest child.

In the second case the _line segment is divided by the splitting plane_.  Each
time the line segment is split like this,  the split point is pushed back such
that it is 0.03125 units away from the plane, however, the point will not be
pushed back beyond the start of the line segment, including splits before!

What happens with the bug is our line segment is first divided across a vertical
split plane, with the crossing point just above the surface.  The collision
doesn't occur in the near side of the split, so the function moves onto the far
side of the split, but with a line segment that starts very close to the ground,
approx 0.0001 units.  When the function finally does the intersection with the
ground plane it attempts to push 0.03125 units upwards but can't since the line
segment only goes as high as ~0.0001 units, meaning the player ends up just a
hair above the surface.

The player can freely move around in this state, and generally won't move away.
In fact, as the lift descends, accumulating floating point error means the
player sinks into the platform and eventually intersects, triggering the boost.

## Resources

[Save game for repro'ing the E4M2 jump](assets/e2m2-squish-repro.sav)
[Diff for debugging](assets/debug.diff)

