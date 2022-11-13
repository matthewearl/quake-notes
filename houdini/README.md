# Houdini ogre

This bug involves an ogre "teleporting" a large distance. It Can be easily
reproduced on the
[gor1f map](https://quake.speeddemosarchive.com/quake/maps/gor1.zip).  Quoting
[Morfans, circa 2006](https://quake.speeddemosarchive.com/quake/oldnews/2006.html):

> By the way, if you've never seen the teleporting ogre phenomenon you might
> like to try the following, which is the easiest way I have found to force it
> to happen. Start gor1 on Nightmare (might need god mode on if you are of a
> weak or "Scandinavian" constitution). Go down the ramp, round the corner and
> up to the end of the wooden jetty so that you are stood in the water and the
> ogre is on the end of the jetty above you. Now stand close enough so that he
> tries to chainsaw you then step back out of range. If it behaves the same for
> you as it does for me then about 50% of the time he will disappear. Now go
> play hunt the ogre. :-) 

I made a save game that can reproduce it fairly reliably, after which I could
start dumping edict info on every frame, to start my investigation.

The ogre in question (entity 143 for those following along at home) translates
from around (71.7 -2095.8 -280.0) to (-216.8 -1807.3 -280.0).  Inserting a gdb
watchpoint to check when the x component goes below zero yields the following
stack trace:

```
Old value = 71.7053146
New value = -216.794189
0x00005555555e7e9c in SV_movestep (ent=0x555557abde48, move=0x7fffffffd900, relink=false)
    at sv_move.c:194
194             VectorCopy (trace.endpos, ent->v.origin);
(gdb) bt
#0  0x00005555555e7e9c in SV_movestep (ent=0x555557abde48, move=0x7fffffffd900, relink=false)
    at sv_move.c:194
#1  0x00005555555e80fd in SV_StepDirection (ent=0x555557abde48, yaw=5.497787, dist=-408)
    at sv_move.c:249
#2  0x00005555555e8413 in SV_NewChaseDir (actor=0x555557abde48, enemy=0x555557a9bc78, dist=-408)
    at sv_move.c:316
#3  0x00005555555e8881 in SV_MoveToGoal () at sv_move.c:416
#4  0x00005555555e3700 in PR_ExecuteProgram (fnum=565) at pr_exec.c:616
#5  0x00005555555e8be1 in SV_RunThink (ent=0x555557abde48) at sv_phys.c:144
#6  0x00005555555ebe2f in SV_Physics_Step (ent=0x555557abde48) at sv_phys.c:1160
#7  0x00005555555ec01f in SV_Physics () at sv_phys.c:1221
#8  0x00005555555d3693 in Host_ServerFrame () at host.c:648
#9  0x00005555555d37b2 in _Host_Frame (time=0.00100000005) at host.c:721
#10 0x00005555555d39b0 in Host_Frame (time=0.00100000005) at host.c:784
#11 0x00005555555f7fce in main (argc=7, argv=0x7fffffffdc88) at main_sdl.c:183
```

From the stack, we can see the ogre is attempting to move a distance of -408
units, at an angle of 5.498 radians (315 degrees).  This function
`SV_MoveToGoal` is called directly from QuakeC, and the distance value is a
parameter passed into the function.

We can also print the QuakeC stack trace in GDB:

```
(gdb) p PR_StackTrace()
    fight.qc : ai_charge
     ogre.qc : ogre_smash12
<NO FUNCTION>
$14 = void
```

I believe the culprit is in the `ogre_smash12` definition:

```
void() ogre_smash1      =[      $smash1,                ogre_smash2     ] {ai_charge(6);
sound (self, CHAN_WEAPON, "ogre/ogsawatk.wav", 1, ATTN_NORM);
};
void() ogre_smash2      =[      $smash2,                ogre_smash3     ] {ai_charge(0);};
void() ogre_smash3      =[      $smash3,                ogre_smash4     ] {ai_charge(0);};
void() ogre_smash4      =[      $smash4,                ogre_smash5     ] {ai_charge(1);};
void() ogre_smash5      =[      $smash5,                ogre_smash6     ] {ai_charge(4);};
void() ogre_smash6      =[      $smash6,                ogre_smash7     ] {ai_charge(4); chainsaw(0);};
void() ogre_smash7      =[      $smash7,                ogre_smash8     ] {ai_charge(4); chainsaw(0);};
void() ogre_smash8      =[      $smash8,                ogre_smash9     ] {ai_charge(10); chainsaw(0);};
void() ogre_smash9      =[      $smash9,                ogre_smash10 ] {ai_charge(13); chainsaw(0);};
void() ogre_smash10     =[      $smash10,               ogre_smash11 ] {chainsaw(1);};
void() ogre_smash11     =[      $smash11,               ogre_smash12 ] {ai_charge(2); chainsaw(0);
self.nextthink = self.nextthink + random()*0.2;};       // slight variation
void() ogre_smash12     =[      $smash12,               ogre_smash13 ] {ai_charge();};
void() ogre_smash13     =[      $smash13,               ogre_smash14 ] {ai_charge(4);};
void() ogre_smash14     =[      $smash14,               ogre_run1       ] {ai_charge(12);};
```

As you can see, most of the calls to `ai_charge` take an argument, which is in
fact a distance to move.  All except for `ogre_smash12`.  I believe this is the
source of the bug --- without an argument the QuakeC interpreter will just
re-use the argument of the most recent function call that takes an argument,
which in this case happens to be -408.


## Thomas Stubgaard's legendary e3m7 ER

If you don't know about it already, I recommend you read Thomas Stubgaard's
account of this run, in the
[SDA archives](https://speeddemosarchive.com/quake/oldnews/2021.html) (search
for "e3m7").  To summarize, there is an ogre that normally blocks a corridor on
this level, requiring several seconds to kill.  Very rarely, and I mean *very*
rarely, the ogre will "teleport" into an open area next to the corridor it
normally blocks.  Thomas estimates he's had it happen 4 or 5 times out of
hundreds of hours attempting runs on the map.

He attributes the teleporting to the houdini ogre trick, however I am not so
sure.  To trigger the trick (as per the analysis above) the ogre needs to go
into a smash animation, and I don't think this can happen, since there's nothing
in the vicinity for the ogre to attack.  It is of course possible that there's
another bug or behaviour that triggers the smash animation, but I have an
alternate hypothesis.

I'll explain with reference to in-game times, which can be seen in the top right
of the SDA [video](https://youtu.be/f-xJ-WqqUIo).  At around 6.5 seconds, the
player hits an invisible trigger which wakes two ogres above.  The first ogre
drops down and is shot at by the player at the 9.7 second mark.  The other ogre
(still above) is the houdini ogre.  By the time the player makes it to the
corridor the ogre normally blocks (at ~12 seconds), the houdini ogre is out of
the way in the next room.

This time of approximately 5.5 seconds seems like enough time for the ogre to
make its way into the next room.   Importing the demo into blender, you can
briefly catch the ogre in the act of running into the side room:

![ogre sneaking into side room](blender.mp4?raw=true)

The ogre isn't always visible due to PVS, but the red arrow shows when he
briefly appears.
