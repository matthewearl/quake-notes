# Spawn wake

Muty's recent [e4m3 ER record](https://www.youtube.com/watch?v=ijitAefEvKs)
demonstrates a very skillful, but also very luck dependent, spawn-grenade boost
across the lava pit.  The trick involves luring the spawn across the pit,
grabbing the silver key, then firing a grenade at the spawn to get a combined
grenade and spawn boost to clear the gap.

One factor in the randomness is the time at which the spawn decides to wake up
and attack the player.  Watch Muty's [Twitch VOD](https://www.twitch.tv/videos/1887666969?t=19m45s)
and you'll see many runs die when the spawn delays its attack, or simply
refuses to move at all. This post explores this behaviour, and devises a
strategy to get the spawn to wake up as early as possible, with a >99%
probability.

When the spawn first sees you, that is, when there is first line of sight
between your origin and the spawn's origin, the spawn goes into its `run`
animation:

```
void()  tbaby_run1      =[      $run1,          tbaby_run2      ] {ai_face();};
void()  tbaby_run2      =[      $run2,          tbaby_run3      ] {ai_face();};
void()  tbaby_run3      =[      $run3,          tbaby_run4      ] {ai_face();};
void()  tbaby_run4      =[      $run4,          tbaby_run5      ] {ai_face();};
void()  tbaby_run5      =[      $run5,          tbaby_run6      ] {ai_face();};
void()  tbaby_run6      =[      $run6,          tbaby_run7      ] {ai_face();};
void()  tbaby_run7      =[      $run7,          tbaby_run8      ] {ai_face();};
void()  tbaby_run8      =[      $run8,          tbaby_run9      ] {ai_face();};
void()  tbaby_run9      =[      $run9,          tbaby_run10     ] {ai_face();};
void()  tbaby_run10     =[      $run10,         tbaby_run11     ] {ai_face();};
void()  tbaby_run11     =[      $run11,         tbaby_run12     ] {ai_run(2);};
void()  tbaby_run12     =[      $run12,         tbaby_run13     ] {ai_run(2);};
...
void()  tbaby_run25     =[      $run25,         tbaby_run1      ] {ai_run(2);};
```

Notice for the first 10 frames it just calls `ai_face()`.  It'll never attack
during this time, just turn to face you.  After this it calls `ai_run()`, which
switches to an attack animation with a probability based on distance to the
player:
- At a distance < 120 units, there's a 90% chance of an attack, every 0.1
seconds.  (Ie. almost certainly within 0.2 seconds)
- At a distance < 500 there's a 20% chance of an attack every 0.1 seconds. (On
average every 0.5 seconds)
- At a distance < 1000 there's a 5% chance of an attack every 0.1 seconds. (On
average every 2 seconds)
- At a distance > 1000 it doesn't attack at all.

If you can time the 11th frame after seeing the spawn to be when you're within
120 units of it, you can get an almost guaranteed attack.  The spawn's
animation, like all other animations, runs at 10 Hz, so this corresponds with a
delay between line-of-sight and passing the spawn of around 1.1 seconds.

[Here's a video demonstrating the trick](https://www.youtube.com/watch?v=NBgwM6_U31I)

In the top left you can see the distance to the spawn, the time, and the current
animation frame.  `tbaby_jump*` means the spawn has gone into attack.  In the
video I take an intentionally wide line to delay passing by the spawn, and I'm
duly rewarded with a near instant wake up.

As it happens this trick didn't end up working for Muty's run.  I think perhaps
the wake up is a little _too_ early.  With any run of this level, the player
has to spend a little time on the far side of the lava pit waiting for a door to
open and grabbing the silver key.

Ideally the spawn would wake _after_ the player has passed it, such that it is
in the right position following the key grab.  With this technique you don't
have to delay much, if at all, before passing the spawn, which saves time.
Sadly this means you're at the whim of the random number generator since the
first `ai_run()` call occurs when you're more than 120 units away.
