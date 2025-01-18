*Below is reproduced the text of Cadaver's excellent explanation of
the code for his side-scrolling platform game "Escape from New
York". This game is a lot simpler than Cadaver's "serious" games such
as Hessian, Steel Ranger, or the Metal Warrior series, and is thus
suitable for beginner's to learn from. If you enjoy this explanation,
check out Cadaver's other excellent articles (which he refers to as
"rants") at his site: https://cadaver.github.io/rants.html.*

*I've converted the orignal text from a hybrid of plain text and html
to Markdown for easier reading; I have not included the assets here as
they aren't needed to understand the code. Should you want them,
there's a download link in the original article:
https://cadaver.github.io/rants/dissect.html.*

*P.S. Despite EFNY being an entry for a "crap game contest" IMO it's
actually on par with commercially released budget games published by
the likes of Mastertronic back in the '80s.*


Escape From New York dissected by Cadaver
=========================================

This rant dissects and explains my Crap Game Compo 1999 entry "Escape
From New York" for your enjoyement & learning exclusively.

Because the game had graphical bugs deliberately included for more
crappiness, I wanted to fix them first before beginning the
dissection. Naturally, I don't want to give examples of buggy coding
:)


General info
------------

First something general about this game. EFNY is a simple platform
game/ shoot'em-up, that features scrolling in one direction (from
right to left), maximum 8 sprites (no multiplexing), bullets done with
character graphics and 4 different kind of enemies. All sprite
position calculations are done in the "screen" coordinates, not in
"world" coordinates like in Metal Warrior games.

EFNY uses 2 raster interrupts: one at the top of the screen to
initialize the game screen/title picture display, and one at the
bottom to initialize the score panel display and to play music.


Definitions
-----------

The source code begins with memory location defines. Sprites,
background character set, music data, background map data (how blocks
are arranged), background block data and the title screen bitmap
picture.

    SPRITES         = $2000
    CHARS           = $3800
    MUSIC           = $4000
    MAP             = $4800
    BLOCKS          = $5000
    PICTURE         = $6000

These are the raster line Y-positions on which raster interrupts
happen.  "Raster0" is the bottom interrupt that displays the score
panel and plays music.  It also increments a counter called
"rastercount" which is used to stabilize execution speed of the main
program. "Raster1" displays the game screen or title screen bitmap
picture.

    RASTER0POS      = 242
    RASTER1POS      = 20

Here are the "actor type" defines. "Actors" are the heart of almost
all my games, basically they are all the objects that move onscreen
(player, enemies, explosions). Sometimes I also define the bullets as
actors (Metal Warrior 1 & 3) but in this case it isn't done. For each
actor there is a move routine that will be called each time the game
situation is updated. Actors can transform from one type to another,
for example the motorist turns into an explosion when he is
destroyed. This is what makes the system very flexible.

    ACT_NONE        = 0
    ACT_PLISSKEN    = 1
    ACT_MAN1        = 2
    ACT_MAN2        = 3
    ACT_MAN3        = 4
    ACT_MOTORIST    = 5
    ACT_EXPLOSION   = 6

Here are the screen mode defines for "raster1" interrupt. There are 3
modes; game graphics, title bitmap, and text display. More of these
later.

    DISPGAME        = 0
    DISPTITLE       = 1
    DISPTEXT        = 2

Joystick bit value defines for each direction and the fire
button. These are read from the $dc00 register (joystick port 2)

    JOY_UP          = 1
    JOY_DOWN        = 2
    JOY_LEFT        = 4
    JOY_RIGHT       = 8
    JOY_FIRE        = 16

Next come all zero-page variable definitions. The first is the display
mode for "raster1" interrupt.

    dispmode        = $02

The next is the X-direction hardware-scrolling (0-7) for the game
screen.

    scrollx         = $03

This is the joystick control data that has been read from $dc00 (With
bits inverted so that a 1 bit means direction/fire button active).

    joystick        = $04

Previous joystick control value. This is used to check that player has
released fire button or up direction (jumping) before a new shot can
be fired or new jump be done.

    prevjoy         = $05

A 16-bit memory location that points at the current level's background
map data.

    mapadrlo        = $06
    mapadrhi        = $07

The position in background map data (measured from its left edge in
blocks) that is used to draw new background graphics to the right edge
of screen when the screen scrolls. Each level is 100 blocks wide (10
whole screens) so this gets values from 0 to 100.

    mapx            = $08

Position in the background data, within a block. Each block is 4 chars
wide so this gets the values 0-3.

    blockx          = $09

Various temp variables, used when needed.

    temp1           = $0a
    temp2           = $0b
    temp3           = $0c
    temp4           = $0d
    temp5           = $0e

The raster interrupt counter incremented by "raster0", used in
stabilizing the game execution speed.

    rastercount     = $0f

DASM assembler requires the processor to be defined, so here that is
taken care. Program code starts at memory location $0800 (2048)

                    processor 6502
                    org $0800


The main program
----------------

First things to do are to clear the processor's decimal flag (just to
be sure), init raster interrupts ("initraster") and initialize screen
colors ("initscreen"). The subroutines are described in more detail
when they are actually encountered in the source code.

    efnystart:      cld
                    jsr initraster
                    jsr initscreen

Here we init music. EFNY's music has been done with the SadoTracker
that uses the "standard" convention for music init/play JSR addresses:
MUSIC+0 is the init routine (subtune number passed in accumulator) and
MUSIC+3 is the play routine to be called each frame. Basically, EFNY
uses only one subtune and subtune numbering starts from 0.

                    lda #$00
                    jsr MUSIC

Title screen code starts here. We come back here whenever a game
ends. The thing we check here is whether a new hiscore has been
made. Both the score and hiscore are 3 bytes of binary coded decimals,
with 2 digits in each byte. The compare is started from the most
significant byte and if no conclusion is made, then the next most
significant byte is checked.

    title:          lda score+2
                    cmp hiscore+2
                    bcc title_nohiscore
                    beq title_check2
                    bcs title_hiscore
    title_check2:   lda score+1
                    cmp hiscore+1
                    bcc title_nohiscore
                    beq title_check3
                    bcs title_hiscore
    title_check3:   lda score
                    cmp hiscore
                    bcc title_nohiscore

If a new hiscore was made, copy the "score" contents (3 bytes) to
"hiscore".

    title_hiscore:  lda score
                    sta hiscore
                    lda score+1
                    sta hiscore+1
                    lda score+2
                    sta hiscore+2

Display the title picture ("showpic") and update the numbers in the
score panel ("drawscores")

    title_nohiscore:jsr showpic
                    jsr drawscores

Title screen waiting loop. Wait for the "rastercount" to increase
("waitras"), get joystick control value ("getjoystick") and check the
fire button bit. Loop until fire button has been pressed.

    titleloop:      jsr waitras
                    jsr getjoystick
                    lda joystick
                    and #JOY_FIRE
                    beq titleloop

Game starts. First thing to do is to clear the score (3 bytes), init
number of lives and the level number.

    gamestart:      lda #$00
                    sta score
                    sta score+1
                    sta score+2
                    lda #3
                    sta lives
                    lda #1
                    sta level

Code to move onto the next level. All sprites are turned off ($d015),
score panel is updated and scrolling of the level background graphics
is initialized ("initscroll")

    initlevel:      lda #$00
                    sta $d015
                    jsr drawscores
                    jsr initscroll

At the end of a level a "kill" meter activates. Here it is deactivated
at first by storing 0 to the variable "killactive", as well as
clearing the meter itself (consisting of the meter length in chars
"killmeter" and the "fine" component of the meter "killmeterd")

                    lda #0
                    sta killactive
                    sta killmeter
                    sta killmeterd

Next we scroll the first screen "in" from the right edge, as seen in
the beginning of each level. The scrolling speed (4 pixels) is given
to "doscroll" routine in the accumulator. We loop until the map
X-position (in blocks) has reached 10, which means that the entire
screen has been scrolled in.

    initlevel2:     jsr waitras
                    lda #4
                    jsr doscroll
                    lda mapx
                    cmp #10
                    bcs initlevel3
                    jmp initlevel2

The next thing to do is to set all the 8 "actors" to inactive state,
which is done by setting the type of each actor to the inactive (zero)
value. The actor type is stored in an indexed array called "actt",
with the index going from 0 to 7. All other actor properties are
stored in similarly indexed arrays as well. Note: do not confuse actor
types and indexes! The player is actor index 0 and enemies are indexes
1-7, but they can be of any type.

    initlevel3:     ldx #7
    initlevel4:     lda #ACT_NONE
                    sta actt,x
                    dex
                    bpl initlevel4

There's a similar array for the bullets, with "bullett" (bullet type)
indicating if a bullet is active or not. Bullets are character
graphics, so there's no limit for their amount, but 16 seems like a
sensible value.

                    ldx #15
    initlevel5:     lda #$00
                    sta bullett,x
                    dex
                    bpl initlevel5

Program execution goes back here also whenever a life has been
lost. Here the time counter ("time", binary coded decimal number) is
reset to the maximum 99. Also the time decrement delay counter
("timedl") is reset.

    initlife:       lda #$99
                    sta time
                    lda #$00
                    sta timedl

The first actor index (index 0) is always the player. So player's
position is reset here. An actor has its X-position stored as a 16-bit
value ("actxl" and "actxh" and the Y-position ("acty") as 8 bits. The
origin of this coordinate system is the first visible pixel in the top
left corner of the screen. Note: The actor position corresponds to the
Y-position of the actor's feet and the center in X-direction.

                    lda #128
                    sta actxl
                    lda #0
                    sta actxh
                    lda #20*8
                    sta acty

Here a number of other actor properties are reset. "Actd" is the actor
direction, 0 is facing right, 1 is facing left. "Actf" is the frame
number (animation). "Actj" indicates whether an actor is jumping (1 =
is jumping) "Actsx" and "actsy" are the X and Y speeds of an
actor. "Actyd" is a delay counter for slowly changing the Y-speed of
an actor, for acceleration due to gravity.

                    lda #0
                    sta actd
                    sta actf
                    sta actj
                    sta actsx
                    sta actsy
                    sta actyd

This is the player's firing delay counter.

                    sta firedelay

Initialize player actor type and give an immortality time of 200 game
frames- ("actimm") Player has only 1 hitpoint ("acthp") so any hit
kills the player.  (enemies have more)

                    lda #ACT_PLISSKEN
                    sta actt
                    lda #200
                    sta actimm
                    lda #1
                    sta acthp

Here begins the game main loop. First we get the joystick controls.

    gameloop:       jsr getjoystick

Handle movement of all actors ("moveactors")

                    jsr moveactors

Handle player shooting ("plrshoot")

                    jsr plrshoot

Generate new enemies at the edges of the screen ("spawnenemies")

                    jsr spawnenemies

Wait for the raster interrupt

                    jsr waitras

Erase bullet characters from the screen ("erasebullets")

                    jsr erasebullets

Move bullets ("movebullets")

                    jsr movebullets

Update score panel.

                    jsr drawscores

Activate and display the kill meter at the end of levels ("killmeterr")

                    jsr killmeterr

Wait for the raster interrupt again, to get as much rastertime as
possible for the screen scrolling (takes quite a lot of time when the
screen memory has to be shifted to the left)

                    jsr waitras

Check for need of scrolling (player moved far enough to the right?)
"Checkscroll" returns the scrolling speed in accumulator, or 0 if no
scrolling required.

                    jsr checkscroll

Here the screen is scrolled, with speed indicated by accumulator.

                    jsr doscroll

Draw all actors ("drawactors")

                    jsr drawactors

Draw all bullet characters ("drawbullets")

                    jsr drawbullets

Decrement time counter ("dectime")

                    jsr dectime

Check player actor death ("checkdeath"). If player actor has died,
this routine does not return but pulls the return address from stack
and jumps to the location "initlife", if lives remain, or to "title"
when game is over.

                    jsr checkdeath

Checks completion of level ("checklevelend"). This routine also does
not return if a level is completed.

                    jsr checklevelend

Go back to the beginning of the gameloop.

                    jmp gameloop

The rest of the code consists of the subroutines.


"Checklevelend" subroutine
--------------------------

If the kill meter is active (nonzero value), compare the meter value
to the limit required by each level.

    checklevelend:  lda killactive
                    beq cle2
                    lda killmeter
                    cmp killlimit
                    bcc cle2
                    pla
                    pla

Counting time bonus at the end of the level. By using binary coded
decimal arithmetic, increase score by 100 points until "time" is zero.

    countbonus:     lda time
                    beq countbonus2
                    sed
                    lda time
                    sec
                    sbc #$01
                    sta time
                    lda score+1
                    clc
                    adc #$01
                    sta score+1
                    bcc noextra

Give an extra life each ten thousand points (when the most significant
byte of the score increments).

                    inc lives
    noextra:        lda score+2
                    adc #$00
                    sta score+2
                    cld

Update the score panel and perform some delaying.

                    jsr drawscores
                    jsr waitras
                    jsr waitras
                    jsr waitras
                    jsr waitras
                    jmp countbonus

Move onto the next level, or if we were on level 3, do the
endsequence.

    countbonus2:    lda level
                    cmp #$03
                    beq complete
                    inc level
                    jmp initlevel

Level not completed: go back to main loop

    cle2:           rts

Endsequence code. Initialize scrolling again, set the "text mode" to
be displayed and clear sprites at first.

    complete:       jsr initscroll
                    lda #DISPTEXT
                    sta dispmode
                    ldx #0
                    stx $d015

Loop to display the ending text (4 rows) on the screen.

    complete1:      lda ctext1,x
                    and #$3f
                    sta $400+4*40+8,x
                    lda ctext2,x
                    and #$3f
                    sta $400+6*40+8,x
                    lda ctext3,x
                    and #$3f
                    sta $400+8*40+8,x
                    lda ctext4,x
                    and #$3f
                    sta $400+10*40+8,x
                    lda #$01
                    sta $d800+4*40+8,x
                    sta $d800+6*40+8,x
                    sta $d800+8*40+8,x
                    sta $d800+10*40+8,x
                    inx
                    cpx #24
                    bne complete1

Use temp1 as a delay counter to wait for a "long" time and then return
to title screen.

                    lda #255
                    sta temp1
    complete2:      jsr waitras
                    jsr waitras
                    dec temp1
                    bne complete2
                    jmp title

    ctext1:         dc.b "     WELL DONE SNAKE    "
    ctext2:         dc.b "YOUR MISSION IS COMPLETE"
    ctext3:         dc.b "AND YOU HAVE EARNED YOUR"
    ctext4:         dc.b "        FREEDOM.        "


"Killmeterr" subroutine
-----------------------

First check if kill meter is yet inactive and we have arrived at the
end of the level (X-position in blocks 100), in which case it must be
activated.

    killmeterr:     lda killactive
                    bne killmeter2
                    lda mapx
                    cmp #100
                    bcc killmeter2
                    inc killactive
                    sta killactive

Draw the word "KILL" on the screen.

                    lda #96
                    sta $400+42
                    lda #97
                    sta $400+43
                    lda #98
                    sta $400+44
                    lda #99
                    sta $400+45

Get the kill meter limit corresponding to the current level.

                    ldx level
                    dex
                    lda killlimittbl,x
                    sta killlimit
                    tax

Draw the empty kill meter bar on screen.

    drawkillloop:   lda #100
                    sta $400+45,x
                    dex
                    bne drawkillloop
    killmeter2:     rts

    killlimittbl:   dc.b 8,12,24


"Dectime" subroutine
--------------------

Decrement the time counter by using decimal mode arithmetic each time
the "timedl" variable has counted 50 game frames.

    dectime:        lda time
                    beq nodectime
                    inc timedl
                    lda timedl
                    cmp #50
                    bcc nodectime
                    lda #$00
                    sta timedl
                    sed
                    lda time
                    sec
                    sbc #$01
                    sta time
                    cld
                    bne nodectime

If time has run out, kill the player actor by setting its hitpoints to
zero.

                    lda #0
                    sta acthp
    nodectime:      rts


"Checkdeath" subroutine
-----------------------

Frame number 7 of the player actor is displayed when he's dead, so
first check for that.

    checkdeath:     lda actf
                    cmp #7
                    bne cdeath_not

Then check if the dead player actor has reached the bottom of the
screen.

                    lda acty
                    cmp #240
                    bcc cdeath_not

If so, pull the return address from the stack and do not return with
RTS; instead decrement lives and jump back to the "initlife" code if
lives still remain, or to "title" when player is out of lives.

                    pla
                    pla
                    dec lives
                    lda lives
                    beq cdeath_gameover
                    jmp initlife
    cdeath_gameover:jmp title
    cdeath_not:     rts


"Plrshoot" subroutine
---------------------

Handle player's shooting attack. First check if the "firedelay"
counter is zero, which means that firing the next bullet is
allowed. If it's nonzero, decrement it and return.

    plrshoot:       lda firedelay
                    beq plrshootok
                    dec firedelay
                    rts

Check for the death frame of player actor. Don't allow firing when
dead.

    plrshootok:     lda actf
                    cmp #7
                    bne plrshootok2
    plrshootnot:    rts

Check that fire button is pressed, and the fire button has been
released previously:

    plrshootok2:    lda joystick
                    and #JOY_FIRE
                    beq plrshootnot
                    lda prevjoy
                    and #JOY_FIRE
                    bne plrshootnot

Then start searching for an unused bullet index (zero value in the
"bullett" array). Bullet indexes 0-7 are reserved for player
bullets. If no unused bullet is found, return.

                    ldx #7
    plrshootfind:   lda bullett,x
                    beq plrshootfound
                    dex
                    bpl plrshootfind
                    rts

Copy player location & direction to bullet location & direction
("bulletxl", "bulletxh", "bullety" and "bulletd"). Y-coord is modified
to make the bullets appear at the height of the player's weapon
(there's a table for this, "actshootymod" (Y-modification) of which we
use the first value, which corresponds to the player actor type,
ACT_PLISSKEN)

    plrshootfound:  lda actxl
                    sta bulletxl,x
                    lda actxh
                    sta bulletxh,x
                    lda acty
                    sec
                    sbc actshootymod
                    sta bullety,x
                    lda actd
                    sta bulletd,x

Set the bullet to active state and set the firing delay.

                    lda #1
                    sta bullett,x
                    lda #2
                    sta firedelay
                    rts


"Plr": Player actor move routine
--------------------------------

Called by "moveactors" routine, X register contains now the current
actor index.

Assume that the player is in standing position; modify the values in
the actortype's Y-size ("actsizey") and shooting Y-modification
tables.

    plr:            lda #42
                    sta actsizey
                    lda #20
                    sta actshootymod

Check for hitpoints running out.

                    lda acthp,x
                    bne plr_nodeath

If player actor is already in the "death" frame, do not re-init the
death sequence.

                    lda actf,x
                    cmp #7
                    beq plr_nodinit

Set the "death" frame and give upwards Y-speed. Reset Y-speed
increment (gravity) delay counter.

                    lda #7
                    sta actf,x
                    lda #-5
                    sta actsy,x
                    lda #0
                    sta actyd,x

Gravity handling for the dead actor. After the Y-speed delay counter
has increased to 3, increase the Y-speed.

    plr_nodinit:    inc actyd,x
                    lda actyd,x
                    cmp #3
                    bcc plr_d2
                    lda #$00
                    sta actyd,x
                    inc actsy,x

After performing the acceleration, move the player actor in
Y-direction.  ("moveactory") The actor index parameter that is
required is already in the X-register.

    plr_d2:         jsr moveactory
                    rts

Player actor death code ends here, now we check if the actor is
jumping.  ("actj" has nonzero value when jumping)

    plr_nodeath:    lda actj,x
                    beq plr_nofly

Set the frame 5 (jumping frame). Do a similar gravity acceleration &
actor Y-movement like above, but with a larger delay value.

    plr_fly:        lda #5
                    sta actf,x
                    inc actyd,x
                    lda actyd,x
                    cmp #5
                    bcc plr_fly2
                    lda #$00
                    sta actyd,x
                    inc actsy,x

Before the Y-movement, perform X-movement by a subroutine
("plisskenmovex").

    plr_fly2:       jsr plisskenmovex
                    jsr moveactory

Now check the Y-speed. If Y-speed is positive & greater than zero, it
is possible for the player to land on a platform (ends the jump).

                    lda actsy,x
                    beq plr_noground
                    bmi plr_noground

Check for ground below feet ("checkground"). This subroutine returns
carry 0 if there is ground.

                    jsr checkground
                    bcs plr_noground

Player landed on ground. Reset Y-speed, jumping indicator and align
Y-position on a character boundary with the and operation.

                    lda #0
                    sta actsy,x
                    sta actj,x
                    lda acty,x
                    and #$f8
                    sta acty,x
    plr_noground:   rts

"Player not jumping"-code. If player has moved into a location where
there's no ground under feet, initiate falling (jumping without
initial upwards Y-speed)

    plr_nofly:      jsr checkground
                    bcc plr_nofall

Set jumping indicator nonzero.

                    lda #1
                    sta actj,x

Initial Y-speed is 0, it will start to increase (downwards speed)

                    lda #0
                    sta actsy,x

Reset Y-acceleration delay counter.

                    sta actyd,x
                    jmp plr_fly

If player not falling, check for various joystick movements. First
comes the check of "joystick up" to initiate a new jump. Up must not
have been previously pressed.

    plr_nofall:     lda joystick
                    and #JOY_UP
                    beq plr_nojump
                    lda prevjoy
                    and #JOY_UP
                    bne plr_nojump
                    lda #1
                    sta actj,x
                    lda #0
                    sta actyd,x

Give the initial upwards (negative) Y-speed.

                    lda #-4
                    sta actsy,x
                    jmp plr_fly
    plr_nojump:     lda #0
                    sta actsx,x

Check for moving left. If moving left, set player facing left ("actd"
1), give X-speed of -2 pixels (left) and jump to the walk animation
code.

                    lda joystick
                    and #JOY_LEFT
                    beq plr_notleft
                    lda #1
                    sta actd,x
                    lda #-2
                    sta actsx,x
                    jmp plr_walkanim

Similar code for moving right.

    plr_notleft:    lda joystick
                    and #JOY_RIGHT
                    beq plr_notright
                    lda #0
                    sta actd,x
                    lda #2
                    sta actsx,x
                    jmp plr_walkanim

If not either left or right, set standing frame (frame 0).

    plr_notright:   lda #0
                    sta actf,x
                    jmp plr_domove

Walk animation. Increase animation frame delay counter, and when it
has counted to five, increase frame. And-operation is used to limit
the frame between 0-3, and 1 is then added (so the final frame range
for walking animation is 1-4).

    plr_walkanim:   inc actfd,x
                    lda actfd,x
                    cmp #5
                    bcc plr_walkanim2
                    lda #$00
                    sta actfd,x
                    lda actf,x
                    and #$03
                    clc
                    adc #$01
                    sta actf,x
    plr_walkanim2:

Finally check for down direction (crouching).

    plr_domove:     lda joystick
                    and #JOY_DOWN
                    beq plr_noduck

Set animation frame to 6 (crouching) and make the player's Y-size now
smaller (for correct collision detection) and modify the shooting
Y-modification as well, as the weapon is held lower now.

                    lda #6
                    sta actf,x
                    lda #18
                    sta actsizey
                    lda #10
                    sta actshootymod
                    rts
    plr_noduck:     jsr plisskenmovex
                    rts

This is a subroutine for player actor's movement in X-direction. It
allows movement left (by calling the "moveactorx" subroutine) only if
the actor's X-coordinate is greater than 10, and movement right only
if the X-coordinate is less than 310, so the player actor isn't
allowed to move outside the visible screen.

    plisskenmovex:  lda actsx,x
                    bmi plr_moveleft
                    lda actxh,x
                    beq plr_moveok
                    lda actxl,x
                    cmp #(310-256)
                    bcc plr_moveok
    plr_nomove:     rts
    plr_moveok:     jsr moveactorx
                    rts
    plr_moveleft:   lda actxh,x
                    bne plr_moveok
                    lda actxl,x
                    cmp #11
                    bcs plr_moveok
                    rts


"Man": Enemy man actor move routine
-----------------------------------

This code is very similar to the "plr" move routine, so it isn't
explained in as much detail. Like before, X register contains the
current actor index.

Check for enemy becoming dead.

    man:            lda acthp,x
                    bne man_notdead

For these kind of enemies, 3 is the death animation frame. If frame
wasn't already 3, init death animation & movement (similar upward
Y-speed like with the player) and increase player score.

                    lda actf,x
                    cmp #3
                    beq man_nodinit
                    jsr addenemyscore
                    lda #3
                    sta actf,x
                    lda #-5
                    sta actsy,x
                    lda #0
                    sta actyd,x
    man_nodinit:    inc actyd,x
                    lda actyd,x
                    cmp #2
                    bcc man_d2
                    lda #$00
                    sta actyd,x
                    inc actsy,x
    man_d2:         jsr moveactory
                    lda acty,x
                    cmp #240
                    bcc man_noremove

The enemy is removed (by setting the actor type to zero) when it moves
off the bottom of screen after death.

                    lda #0
                    sta actt,x
    man_noremove:   rts

    man_notdead:    lda actj,x
                    beq man_nofly

Enemy jumping code. 2 is the jumping animation frame for these
enemies.

    man_fly:        lda #2
                    sta actf,x
                    inc actyd,x
                    lda actyd,x
                    cmp #5
                    bcc man_fly2
                    lda #$00
                    sta actyd,x
                    inc actsy,x
    man_fly2:       jsr moveactorx
                    jsr moveactory
                    lda actsy,x
                    beq man_noground
                    bmi man_noground
                    jsr checkground
                    bcs man_noground
                    lda #0
                    sta actsy,x
                    sta actj,x
                    lda acty,x
                    and #$f8
                    sta acty,x
    man_noground:   jmp man_shoot
                    rts

Enemy walking code. If an enemy has moved to a location where it
doesn't have ground under its feet, it jumps (this is a bit of
cheating, since the player actor would fall in a similar situation)

    man_nofly:      jsr checkground
                    bcc man_nojump
                    lda #1
                    sta actj,x
                    lda #-4
                    sta actsy,x
                    sta actyd,x
                    jmp man_fly
    man_nojump:     lda actd,x
                    beq man_right

The enemy's move speed depends on its type (there are 3 kinds of these
enemies). Actually the enemy type's maximum hitpoints are used as the
move speed.

                    ldy actt,x
                    dey
                    lda actmaxhp,y

Here the speed needs to be negated by two's complement for moving
left.

                    eor #$ff
                    clc
                    adc #$01
                    sta actsx,x
                    jmp man_walkanim
    man_right:      ldy actt,x
                    dey
                    lda actmaxhp,y
                    sta actsx,x
    man_walkanim:   inc actfd,x
                    lda actfd,x
                    cmp #6
                    bcc man_walkanim2
                    lda #0
                    sta actfd,x

The enemy actor uses only frames 0 & 1 for walking animation, so it
looks quite primitive:

                    lda actf,x
                    eor #$01
                    and #$01
                    sta actf,x
    man_walkanim2:  jsr moveactorx

Enemy shooting routine. Call "random" subroutine to get a pseudorandom
number for shooting decision.

    man_shoot:      jsr random
                    sta temp1

Compare the level number against the random value. If it's smaller
then do not shoot. This has the effect that enemies start to shoot
more frequently in later levels.

                    lda level
                    cmp temp1
                    bcc man_noshoot

Now search for a free enemy bullet (nonzero value in "bullett" array),
using the Y register as an index. Bullet indexes 8-15 are reserved for
enemies.

                    ldy #8
    manshootfind:   lda bullett,y
                    beq manshootfound
                    iny
                    cpy #$10
                    bcc manshootfind

No bullet found, just exit

    man_noshoot:    rts

Copy the actor location & direction to bullet location & direction,
and perform Y-position modification, like in the player actor's shoot
routine.

    manshootfound:  lda actxl,x
                    sta bulletxl,y
                    lda actxh,x
                    sta bulletxh,y
                    lda acty,x
                    sec
                    sbc #20
                    sta bullety,y
                    lda actd,x
                    sta bulletd,y
                    lda #1
                    sta bullett,y
                    rts


"Mc": Enemy motorist actor move routine
---------------------------------------

When the motorist is killed, he turns into an explosion.

    mc:             lda acthp,x
                    bne mc_notdead
                    jsr addenemyscore

Initialize animation frame for the explosion, and change actor type.

                    lda #0
                    sta actf,x
                    lda #ACT_EXPLOSION
                    sta actt,x
                    rts

Rest of the code is very similar to the player & enemy move routines
seen before.

    mc_notdead:     lda actj,x
                    beq mc_nofly
    mc_fly:         lda #0
                    sta actf,x
                    inc actyd,x
                    lda actyd,x
                    cmp #5
                    bcc mc_fly2
                    lda #$00
                    sta actyd,x
                    inc actsy,x
    mc_fly2:        jsr moveactorx
                    jsr moveactory
                    lda actsy,x
                    beq mc_noground
                    bmi mc_noground
                    jsr checkground
                    bcs mc_noground
                    lda #0
                    sta actsy,x
                    sta actj,x
                    lda acty,x
                    and #$f8
                    sta acty,x
    mc_noground:    rts
    mc_nofly:       jsr checkground
                    bcc mc_nojump
                    lda #1
                    sta actj,x
                    lda #-3
                    sta actsy,x
                    sta actyd,x
                    jmp mc_fly
    mc_nojump:      lda actd,x
                    beq mc_right
                    lda #-4
                    sta actsx,x
                    jmp mc_walkanim
    mc_right:       lda #4
                    sta actsx,x
    mc_walkanim:    lda actf,x
                    eor #$01
                    and #$01
                    sta actf,x
                    jsr moveactorx

Here is the difference: The motorist can kill the player actor by
colliding.  Check collision between the enemy actor (index in X
register) and player (Y register loaded with 0, player actor's index)
with the "actactcoll" (actor-actor collision) subroutine.

                    ldy #0
                    jsr actactcoll
                    bcc mc_nocoll

Carry 1 indicates collision. Decrease player actor hitpoints (player
actor has only one so he's killed)

                    lda acthp,y
                    sec
                    sbc #$01
                    sta acthp,y
    mc_nocoll:      rts


"Expl": Explosion actor move routine
------------------------------------

This is a simple "move" routine because it only involves
animation. Count the frame delay up to 6 and after that increase the
animation frame. The explosion has 4 animation frames, after all these
have been shown the explosion is removed by putting zero value to
actor type ("actt").

    expl:           inc actfd,x
                    lda actfd,x
                    cmp #6
                    bcc expl2
                    lda #0
                    sta actfd,x
                    inc actf,x
                    lda actf,x
                    cmp #4
                    bcc expl2
                    lda #0
                    sta actt,x
    expl2:          rts


"Addenemyscore" subroutine
--------------------------

Add score according to the actor type of the dead enemy (use a value
from a table), using decimal arithmetic.

    addenemyscore:  ldy actt,x
                    dey
                    sed
                    lda score
                    clc
                    adc actscorelo,y
                    sta score
                    lda score+1
                    adc actscorehi,y
                    sta score+1
                    bcc noextra2

When tens of thousands increase, give an extra life.

                    inc lives

    noextra2:       lda score+2
                    adc #$00
                    sta score+2
                    cld

If the kill meter is active, lenghten the kill meter bar on
screen. "Killmeterd" goes through values 0-4 to specify what char is
drawn at the end of the meter, to make it increase
smoothly. "Killmeter" is the meter length in whole chars (indicates
the position of the meter's endpoint on screen). The chars used for
the meter are 100-104.

                    lda killactive
                    beq addenemyscore2
                    inc killmeterd
                    ldy killmeter
                    lda #100
                    clc
                    adc killmeterd
                    sta $400+46,y
                    lda killmeterd
                    cmp #4
                    bcc addenemyscore2
                    lda #$00
                    sta killmeterd
                    inc killmeter
    addenemyscore2: rts


"Moveactorx" subroutine
-----------------------

Moves an actor (index indicated by the X-register) horizontally,
according to its speed ("actsx" array). "Actsx" is only 8 bits so to
perform the 16-bit position addition correctly we must check its sign
before adding.

    moveactorx:     clc
                    lda actsx,x
                    bmi negmovex
                    adc actxl,x
                    sta actxl,x
                    lda actxh,x
                    adc #$00
                    sta actxh,x
                    rts
    negmovex:       adc actxl,x
                    sta actxl,x
                    lda actxh,x
                    adc #$ff
                    sta actxh,x
                    rts


"Moveactory" subroutine
-----------------------

Moves an actor (index indicated by the X-register) vertically,
according to its speed ("actsy" array). Y-position is only 8 bits like
the speed so this movement is easy to do.

    moveactory:     lda actsy,x
                    clc
                    adc acty,x
                    sta acty,x
                    rts


"Checkground" subroutine
------------------------

Checks for ground under the actor's (index indicated by X-register)
feet. First divide the actor's Y-position by 8 to get the screen row
where we must check.

    checkground:    lda acty,x
                    lsr
                    lsr
                    lsr

If it's over the gamescreen portion of the screen, limit the row
number.

                    cmp #24
                    bcc cg_notover1
                    lda #23

Fetch the screen row's memory address from a table.

    cg_notover1:    tay
                    lda rowtbllo,y
                    sta temp1
                    lda rowtblhi,y
                    sta temp2

Now divide the X-position by 8 to get the char column we need.

                    lda actxh,x
                    sta temp3
                    lda actxl,x
                    lsr temp3
                    ror
                    lsr temp3
                    ror
                    lsr temp3
                    ror

Move the column to Y-register, limit it to the visible screen
boundaries (0-39) if it's less or greater.

                    tay
                    bpl cg_notoverleft
                    ldy #0
                    jmp cg_notoverright
    cg_notoverleft: cpy #39
                    bcc cg_notoverright
                    ldy #39

Then read the char from that location at screen memory. Characters
0-31 can be walked on, so carry will be 0 in that case, otherwise 1.

    cg_notoverright:lda (temp1),y
                    cmp #32
                    rts

This is the table of memory addresses of the 24 gamescreen rows.

    rowtbllo:
    N               SET 0
                    REPEAT 24
                    dc.b #<($400+N*40)
    N               SET N+1
                    REPEND
    rowtblhi:
    N               SET 0
                    REPEAT 24
                    dc.b #>($400+N*40)
    N               SET N+1
                    REPEND


"Random" subroutine
-------------------

Pseudorandom generator. Reads a byte from a certain memory range
($0b00-$bff, the program code), then adds the previous random number
and $d012 (raster line Y-position) to it, returning the new random
number in the accumulator. Code is selfmodified to get the next byte
to read on the next execution of this subroutine.

    random:         lda $b00
                    adc randseed
                    adc $d012
                    sta randseed
                    inc random+1
                    rts
    randseed:       dc.b $73


"Bullactcoll" subroutine
------------------------

Bullet-actor collision. Parameters and return value (carry flag) are
shown below.

        ;X = bullet number (bullet must exist!)
        ;Y = actor number
        ;C=1 collision happened

The collision routines are all based on coordinate range checking. The
bullets use the same coordinate system as the actors.

Check that the actor exists and it doesn't have immortality time left.

    bullactcoll:    lda actt,y
                    beq bsc_nocoll
                    lda actimm,y
                    bne bsc_nocoll

Also, if an actor doesn't have hitpoints left, it doesn't participate
in the collision checking.

                    lda acthp,y
                    bne bsc1
    bsc_nocoll:     clc
                    rts

Get the actor X & Y size from a table, based on the actor type (modify
the CMP instructions directly). Note that registers have to be saved
to temporary locations.

    bsc1:           sty temp1
                    lda actt,y
                    tay
                    dey
                    lda actsizex,y
                    sta bsc_xcmp+1
                    lda actsizey,y
                    sta bsc_suby+1

Compare Y coordinate ranges first. The bullet is considered a
point-like object. If bullet is outside the actor's Y-size range, no
collision has happened.

                    ldy temp1
                    lda acty,y              ;Check against bottom of actor
                    cmp bullety,x
                    bcc bsc_nocoll
                    sec
    bsc_suby:       sbc #$00                ;Check against top of actor
                    cmp bullety,x
                    bcs bsc_nocoll

Then compare X-coordinates. Get X distance between bullet & actor by
subtraction, and its absolute value (negate it if it's negative)

                    lda actxl,y             ;Get X distance between bullet&actor
                    sec
                    sbc bulletxl,x
                    sta temp1
                    lda actxh,y
                    sbc bulletxh,x
                    sta temp2
                    bpl bsc_posofs
                    eor #$ff
                    sta temp2
                    lda temp1
                    eor #$ff
                    clc
                    adc #$01
                    sta temp1
                    lda temp2
                    adc #$00
                    sta temp2

Then check that bullet is not farther away than the actor's X size, or
collision hasn't happened.

    bsc_posofs:     lda temp2                       ;X distance must not be
                    bne bsc_nocoll                  ;greater than X size
                    lda temp1
    bsc_xcmp:       cmp #$00
                    bcs bsc_nocoll
                    sec
                    rts


"Actactcoll" subroutine
-----------------------

Actor-actor collision. This is similar to the subroutine above, but
both actors have a size, and that must be taken into account in the
coordinate comparisions.

        ;X = actor number (must exist)
        ;Y = actor number
        ;C=1 collision happened

Check for actor existing, being not immortal and having hitpoints,
like before.

    actactcoll:     lda actt,y
                    beq ssc_nocoll
                    lda actimm,y
                    bne ssc_nocoll
                    lda acthp,y
                    bne ssc1
    ssc_nocoll:     clc
                    rts

Get the X & Y sizes of actors. Again, register saving to temporary
locations must happen, because of the need to use many table indexes.

    ssc1:           sty temp1
                    stx temp2
                    lda actt,y
                    tay
                    dey
                    lda actt,x
                    tax
                    dex
                    lda actsizex,y
                    clc
                    adc actsizex,x
                    sta ssc_xcmp+1
                    lda actsizey,y
                    sta ssc_ycmp+1
                    ldy temp1
                    ldx temp2
                    lda acty,y              ;Check against bottom of actor
                    sec
                    sbc acty,x
                    bpl ssc_ypos
                    eor #$ff
                    clc
                    adc #$01
    ssc_ypos:
    ssc_ycmp:       cmp #$00
                    bcs ssc_nocoll
                    lda actxl,y             ;Get X distance between actors
                    sec
                    sbc actxl,x
                    sta temp1
                    lda actxh,y
                    sbc actxh,x
                    sta temp2
                    bpl ssc_posofs
                    eor #$ff
                    sta temp2
                    lda temp1
                    eor #$ff
                    clc
                    adc #$01
                    sta temp1
                    lda temp2
                    adc #$00
                    sta temp2
    ssc_posofs:     lda temp2                       ;X distance must not be
                    bne ssc_nocoll                  ;greater than X size
                    lda temp1
    ssc_xcmp:       cmp #$00
                    bcs ssc_nocoll
                    sec
                    rts


"Spawnenemies" subroutine
-------------------------

Creates enemy actors to the left & right borders of the screen. Enemy
appearance frequency and enemy actor types that will be spawned
depends on the level we're on.

Multiply level number by 16 to get an index to the spawn table.

    spawnenemies:   lda level
                    sec
                    sbc #$01
                    asl
                    asl
                    asl
                    asl
                    sta spawnadd+1

Decision for spawning, if random number is in the range $10-$3f then
don't spawn.

                    jsr random
                    and #$3f
                    cmp #$10
                    bcc spawnok1
                    rts
    spawnok1:       clc

Add the random number to the spawn table index, to get the final
position in the table.

    spawnadd:       adc #$00
                    tax

Get enemy actor type from the spawn table. If it's 0, don't spawn.

                    lda levelspawntable,x
                    bne spawnok2
                    rts

Search for a free actor (actor type zero) in the actor index range 1-7
(enemies).

    spawnok2:       ldx #1
    spawnsearch:    ldy actt,x
                    beq spawnfound
                    inx
                    cpx #7
                    bcc spawnsearch
                    rts
    spawnfound:     sta actt,x

Enemy actor index is now in the X-register. Decision whether the enemy
appears on left or right. Give the corresponding X-coordinate to the
enemy.

                    jsr random
                    and #$01
                    tay
                    sta actd,x
                    lda spawnxlo,y
                    sta actxl,x
                    lda spawnxhi,y
                    sta actxh,x

Random decision for the Y position of the actor.

                    jsr random
                    and #$03
                    clc
                    adc #$02

Multiply Y-coordinate by 32; enemies' initial Y-coordinates are
aligned to the blocks (4 chars high) on screen.

                    asl
                    asl
                    asl
                    asl
                    asl
                    sta acty,x

If there is no ground under the feet of an enemy, it will not be
spawned. In that case, simply zero the actor type and exit the
routine.

                    jsr checkground
                    bcc spawnground
                    lda #0
                    sta actt,x
                    rts

Reset jumping, speed, animation frame, frame delay, immortality.

    spawnground:    lda #0
                    sta actj,x
                    sta actsx,x
                    sta actsy,x
                    sta actf,x
                    sta actfd,x
                    sta actyd,x
                    sta actimm,x

Give the initial hitpoints according to enemy actor type. (from a
table)

                    ldy actt,x
                    dey
                    lda actmaxhp,y
                    sta acthp,x
                    rts

The types of enemies that will be spawned in each level.

                    ;level1
    levelspawntable:dc.b 0,2,0,2,0,3,0,2,0,2,0,3,0,2,0,2
                    ;level2
                    dc.b 0,2,2,3,0,2,3,4,0,3,4,0,2,3,2,5
                    ;level3
                    dc.b 2,2,3,3,2,3,4,4,3,3,4,4,3,2,5,5

Initial X-coordinates are either 0 or 320

    spawnxlo:       dc.b 0,<320
    spawnxhi:       dc.b 0,>320


"Checkscroll" subroutine
------------------------

Checks if the player actor's X-position is on the right side of the
screen and gives the scrolling speed in the accumulators (zero if no
scrolling.)

    checkscroll:    ldx #$00
                    lda actxh
                    cmp #1
                    bcs needscroll
                    lda actxl
                    cmp #200
                    bcc noneedscroll
    needscroll:     ldx #$02
    noneedscroll:   txa
                    rts


"Drawbullets" subroutine
------------------------

Saves the chars that are underneath the bullet positions and draws the
bullet chars. There are 16 bullets (possibly) to draw.

The bullet index (X-register) will go from last to first.

    drawbullets:    ldx #15

Check if bullet is active. Skip if not.

    dbloop:         lda bullett,x
                    beq dbnext

Divide bullet Y-pos by 8 to get character row number.

                    lda bullety,x
                    lsr
                    lsr
                    lsr

If outside the visible gamescreen, skip.

                    cmp #23
                    bcs dbnext

Get the corresponding screen memory row address.

                    tay
                    lda rowtbllo,y
                    sta temp1
                    lda rowtblhi,y
                    sta temp2

Divide bullet X-pos by 8 to get column number.

                    lda bulletxh,x
                    sta temp3
                    lda bulletxl,x
                    lsr temp3
                    ror
                    lsr temp3
                    ror
                    lsr temp3
                    ror

If outside the visible gamescreen, skip.

                    cmp #40
                    bcs dbnext

Add the column to the row address to get the final memory
location. Store also the memory location to the bullet array for fast
retrieval later.

                    clc
                    adc temp1
                    sta temp1
                    sta bulletlo,x
                    lda temp2
                    adc #$00
                    sta temp2
                    sta bullethi,x
                    ldy #0

Save the char under the bullet position and draw the bullet (char
number 255)

                    lda (temp1),y
                    sta bulletunder,x
                    lda #255
                    sta (temp1),y

Set bullet type to 2 to indicate the bullet has been drawn on screen.

                    lda #2
                    sta bullett,x

Loop until all bullets done.

    dbnext:         dex
                    bpl dbloop
                    rts


"Erasebullets" subroutine
-------------------------

Restores the chars that were overwritten by bullets on screen. To
ensure correct restore, the index must go in reverse direction (from
first to last) than in the "drawbullets" routine.

Start from bullet index 0 (in X-register)

    erasebullets:   ldx #0

Skip if the bullet hasn't been drawn

    ebloop:         lda bullett,x
                    cmp #2
                    bne ebnext
                    ldy #0

The screen memory position has already been calculated.

                    lda bulletlo,x
                    sta temp1
                    lda bullethi,x
                    sta temp2

Restore the char that was under the bullet.

                    lda bulletunder,x
                    sta (temp1),y

Reset the bullet type to 1 to indicate it has been erased from the
screen

                    lda #1
                    sta bullett,x
    
Loop until all 16 bullets have been checked.

    ebnext:         inx
                    cpx #16
                    bcc ebloop
                    rts


"Movebullets" subroutine
------------------------

Moves the bullets, if they exist, and checks their collisions to
player & enemies.

Start from the last bullet (X-register as index).

    movebullets:    ldx #15

Does the bullet exist?

    mbloop:         lda bullett,x
                    beq mbnext

Is it a player or enemy bullet?

                    cpx #8
                    bcs checkenemybull

It's a player bullet, loop through the enemy actor indexes 1-7 to
check collisions.

                    ldy #1
    checkplayerbull:
                    jsr bullactcoll
                    bcc cpb_nocoll

If a collision happened, reduce the enemy's hitpoints by one and
remove the bullet that collided.

                    lda acthp,y
                    sec
                    sbc #$01
                    sta acthp,y
                    lda #$00
                    sta bullett,x
                    jmp mbnext
    cpb_nocoll:     iny
                    cpy #8
                    bcc checkplayerbull
                    jmp bullcheckdone

It's an enemy bullet, check collision to player actor and reduce
player actor's hitpoints (kill player actor!) + remove bullet if
collided

    checkenemybull: ldy #0
                    jsr bullactcoll
                    bcc bullcheckdone
                    lda acthp,y
                    sec
                    sbc #$01
                    sta acthp,y
                    lda #$00
                    sta bullett,x
                    jmp mbnext

Collision checking has been done, and the bullet wasn't removed. Next,
move the bullet. Check direction ("bulletd" array); move the bullet 8
pixels either left or right depending on that.

    bullcheckdone:  lda bulletd,x
                    bne mbleft

Movement right.

    mbright:        lda bulletxl,x
                    clc
                    adc #8
                    sta bulletxl,x
                    lda bulletxh,x
                    adc #0
                    sta bulletxh,x
                    beq mbnext

If the bullet goes outside the screen, remove it.

                    lda bulletxl,x
                    cmp #(320-256)
                    bcc mbnext
    mberase:        lda #0
                    sta bullett,x
                    jmp mbnext

Movement left.

    mbleft:         lda bulletxl,x
                    sec
                    sbc #8
                    sta bulletxl,x
                    lda bulletxh,x
                    sbc #0
                    sta bulletxh,x

If the bullet goes outside the screen, remove it.

                    bmi mberase

Loop until all bullets have been moved.

    mbnext:         dex
                    bpl mbloop
                    rts


"Moveactors" subroutine
-----------------------

Calls the move routine of each actor (0-7) and decreases their
immortality counter (used only for the player). Also, for enemy actors
(1-7) a check is made to see if they have gone are outside the screen;
in this case they are removed.

Loop from last actor to first, with X register as index.

    moveactors:     ldx #7

Does the actor exist?

    mactloop:       lda actt,x
                    beq mactnext

If it has immortality left, decrease the immortality counter.

                    lda actimm,x
                    beq mact_noimm
                    dec actimm,x

If it's the player, skip the removal check.

    mact_noimm:     cpx #0
                    beq mact_noremove
                    lda actxh,x
                    beq mact_noremove
                    bmi mact_rleft

Check for X-coordinates greater/equal to 330 or less than -10, and
remove the actor in that case.

    mact_rright:    lda actxl,x
                    cmp #<(330)
                    bcc mact_noremove
                    lda #0
                    sta actt,x
                    jmp mactnext
    mact_rleft:     lda actxl,x
                    cmp #(256-10)
                    bcs mact_noremove
                    lda #0
                    sta actt,x
                    jmp mactnext

Make sure X is preserved for the next actor (although no actor move
routine should modify the X register)

    mact_noremove:  stx mact_restx+1

Get the move routine JSR address corresponding to the actor type's
move routine and call the move routine.

                    lda actt,x
                    tay
                    dey
                    lda actroutlo,y
                    sta mactjsr+1
                    lda actrouthi,y
                    sta mactjsr+2
    mactjsr:        jsr $0000

Go to next actor, loop until all have been done.

    mact_restx:     ldx #$00
    mactnext:       dex
                    bpl mactloop
                    rts


"Drawactors" subroutine
-----------------------

Transforms the actors' position and animation frame into actual sprite
data put to the sprite registers.

Init a few "virtual" sprite registers for the X-coordinate MSB, sprite
on bits and X & Y expansion. This is to prevent flicker.

    drawactors:     lda #$00
                    sta virtd010
                    sta virtd015
                    sta virtd017
                    sta virtd01d

Loop through all actors (X is the index).

                    ldx #7

Check that the actor exists, and move its type to the Y register, for
use in actortype properties (color, expansion etc.) table lookups.

    dactloop:       lda actt,x
                    bne dactok
                    jmp dactnext
    dactok:         tay
                    dey

If the actor is immortal, it flashes at the rate given by the 3th bit
of the immortality counter. When that bit is on, don't draw the actor.

                    lda actimm,x
                    and #$08
                    beq dact_noflash
                    jmp dactnext

Set the corresponding $d015 bit (sprite is on)

    dact_noflash:   lda virtd015
                    ora bittable,x
                    sta virtd015

Get the actor X-coordinate and subtract the actor's hotspot
(X-center).

                    lda actxl,x
                    sec
                    sbc acthotx,y
                    sta temp1
                    lda actxh,x
                    sbc #$00
                    sta temp2

Do same for Y-coordinate.

                    lda acty,x
                    sec
                    sbc acthoty,y
                    sta temp3

Because the actor coordinate system's origin was (0,0) but the
top-left edge is (24,50) for the actual sprite coordinates, add those
values.

                    lda temp3
                    clc
                    adc #50
                    sta temp3
                    lda temp1
                    clc
                    adc #24
                    sta temp1
                    lda temp2
                    adc #0
                    sta temp2

Get actor's "base frame" depending on its direction.

                    lda actd,x
                    beq dact_right
    dact_left:      lda actbaseframel,y
                    jmp dact_frame
    dact_right:     lda actbaseframer,y
    dact_frame:     clc

Add the animation frame to the base frame, and store the frame number
to the spriteframe pointers (last 8 bytes of screen memory)

                    adc actf,x
                    sta 2040,x

Get actor's color and store it to the sprite color register

                    lda actcolor,y
                    sta $d027,x

If actor is expanded in X or Y direction, set the corresponding expand
bits.

                    lda actmagx,y
                    beq dact_nomagx
                    lda virtd01d
                    ora bittable,x
                    sta virtd01d
    dact_nomagx:    lda actmagy,y
                    beq dact_nomagy
                    lda virtd017
                    ora bittable,x
                    sta virtd017

Multiply actor (sprite) number by 2 to get the index to the X/Y
coordinate registers.

    dact_nomagy:    txa
                    asl
                    tay

Store the X-coordinate least significant byte and Y-coordinate.

                    lda temp1
                    sta $d000,y
                    lda temp3
                    sta $d001,y

Then handle X-coordinate most significant bit; set the $d010 bit if X-
coordinate is in the range 256-511

                    lda temp2
                    beq dactnext
                    lda virtd010
                    ora bittable,x
                    sta virtd010

Loop until all actors done.

    dactnext:       dex
                    bmi dactdone
                    jmp dactloop

Then dump the virtual sprite bit registers to the actual video
registers.

    dactdone:       lda virtd010
                    sta $d010
                    lda virtd015
                    sta $d015
                    lda virtd017
                    sta $d017
                    lda virtd01d
                    sta $d01d
                    rts

Powers of two for the corresponding bits of each sprite

    bittable:       dc.b 1,2,4,8,16,32,64,128
    virtd010:       dc.b 0
    virtd015:       dc.b 0
    virtd017:       dc.b 0
    virtd01d:       dc.b 0


"Waitras" subroutine
--------------------

Waits until the raster interrupt counter increased by "raster0" has
changed.  Resets the counter afterwards.

    waitras:        lda rastercount
                    cmp #$01
                    bcc waitras
                    lda #$00
                    sta rastercount
                    rts


"Initscroll" subroutine
-----------------------

Clears the screen and sets the correct color memory value (multicolor
white) for gamescreen displaying. Resets the map & block positions
("mapx", "blockx") as well as the fine scrolling ("scrollx") and sets
the map data pointer based on the level we're on. The background
graphics map data for the levels is organized in the memory as
follows:

    1st block-row of 1st level (100 blocks = 100 bytes)
       ...
    5th block-row of 1st level (100 blocks = 100 bytes)

    1st block-row of 2nd level (100 blocks = 100 bytes)
       ...
    5th block-row of 2nd level (100 blocks = 100 bytes)

    1st block-row of 3rd level (100 blocks = 100 bytes)
       ...
    5th block-row of 3rd level (100 blocks = 100 bytes)

    initscroll:
                    ldx #39
    iscr1:
    N               SET 0
                    REPEAT 24
                    lda #$20
                    sta $400+N*40,x
                    lda #$09
                    sta $d800+N*40,x
    N               SET N+1
                    REPEND
                    dex
                    bmi iscrdone1
                    jmp iscr1
    iscrdone1:      lda #$00
                    sta mapx
                    sta blockx
                    lda #$07
                    sta scrollx
                    lda level
                    sec
                    sbc #$01
                    asl
                    tax
                    lda levelmaptbl,x
                    sta mapadrlo
                    lda levelmaptbl+1,x
                    sta mapadrhi

;Finally, set the display mode used by "raster1" interrupt.

                    lda #DISPGAME
                    sta dispmode
                    rts
    levelmaptbl:    dc.w MAP, MAP+500, MAP+1000


"Doscroll" subroutine
---------------------

Performs X-scrolling. Amount of pixels to scroll (scrolling speed) is
given in the accumulator.

If already at the right edge of a level, do not scroll further.

    doscroll:       ldx mapx
                    cpx #100
                    bcc doscrollok
                    rts
    doscrollok:     sta scrsub+1
                    sta sprsub+1

Move all actors to the left by the amount of pixels to scroll.

                    ldx #7
    doscrollspr:    lda actxl,x
                    sec
    sprsub:         sbc #$00
                    sta actxl,x
                    lda actxh,x
                    sbc #$00
                    sta actxh,x
                    dex
                    bpl doscrollspr

Then subtract the scrolling amount from the X fine-scroll. If it goes
to negative, screen data must be shifted.

                    lda scrollx
                    sec
    scrsub:         sbc #$00
                    bmi scrshift
                    sta scrollx
                    rts
    scrshift:       and #$07
                    sta scrollx

First shift the top 10 rows of gamescreen (screen rows 4-13) one char
to the left (all rows not done at once to eliminate tearing effects on
NTSC machines, that have less rastertime).

                    ldx #$00
    scrshiftloop1:
    N               SET 4
                    REPEAT 10
                    lda $400+N*40+1,x
                    sta $400+N*40,x
    N               SET N+1
                    REPEND
                    inx
                    cpx #39
                    bne scrshiftloop1

Then shift the bottom 10 rows of gamescreen (screen rows 14-23)

                    ldx #$00
    scrshiftloop2:
    N               SET 14
                    REPEAT 10
                    lda $400+N*40+1,x
                    sta $400+N*40,x
    N               SET N+1
                    REPEND
                    inx
                    cpx #39
                    bne scrshiftloop2

Next it's time to draw new background graphics to the edge of the
screen.  The mapdata tells what numbered blocks must be drawn, and the
blocks (4x4 char sized) tell what chars must be drawn on screen.

Add the map x-position to the left edge address of map.

                    lda mapadrlo
                    clc
                    adc mapx
                    sta temp1
                    lda mapadrhi
                    adc #$00
                    sta temp2

This is the destination screen pointer, starting from the rightmost
column of screen row 4 (first gamescreen row).

                    lda #<($400+4*40+39)
                    sta temp3
                    lda #>($400+4*40+39)
                    sta temp4

This is the row counter (20 rows to do)

                    lda #20
                    sta temp5

                    ldy #$00

Data for each 4x4 block is stored in the following way:

    0 1 2 3
    4 5 6 7
    8 9 a b
    c d e f

So, to get on the next row in a block, 4 must be added to the memory
address from where fetching the block data. To get the horizontal
position within a block, the block-x position (0-3) can just be added
to that address.

    scrblockloop:   ldx blockx

Get the block number from the map data.

                    lda (temp1),y
                    tay

Modify the LDA instruction to fetch block data, to point to the
address of just that block.

                    lda blocktbllo,y
                    sta scrblockget+1
                    lda blocktblhi,y
                    sta scrblockget+2

To get onto the next map-row, increase the map address by 100 bytes
(done here already)

                    lda temp1
                    clc
                    adc #100
                    sta temp1
                    lda temp2
                    adc #$00
                    sta temp2
                    ldy #$00

Now get the chars from the blockdata and put them on the screen. X
register is the position within the block.

    scrblockget:    lda $1000,x
                    sta (temp3),y

Add 40 to the destination screen address to get on the next row.

                    lda temp3
                    clc
                    adc #40
                    sta temp3
                    lda temp4
                    adc #0
                    sta temp4

Increase position within block with 4 to get on the next block row (as
told earlier)

                    txa
                    adc #4
                    tax

All 20 rows done?

                    dec temp5
                    beq scrblockready

If the block-position went to 16 or over that it's time to fetch the
next block from the map data.

                    cpx #$10
                    bcc scrblockget
                    jmp scrblockloop

New data has been drawn. Now increase the block & map-positions, so
that the next column of background graphics will be drawn next time.

    scrblockready:  inc blockx
                    lda blockx
                    cmp #$04
                    bcc scrblockready2
                    lda #$00
                    sta blockx
                    inc mapx
    scrblockready2: rts

This is a table for the addresses of all background graphics blocks.

    blocktbllo:
    N               SET 0
                    REPEAT 128
                    dc.b #<(BLOCKS+N*16)
    N               SET N+1
                    REPEND
    blocktblhi:
    N               SET 0
                    REPEAT 128
                    dc.b #>(BLOCKS+N*16)
    N               SET N+1
                    REPEND


"Getjoystick" subroutine
------------------------

First set all bits of $dc00 to 1 to be able to read them correctly,
then save the current joystick control status to the previous status,
then get new status from $dc00, negating all the bits.

    getjoystick:    lda #$ff
                    sta $dc00
                    lda joystick
                    sta prevjoy
                    lda $dc00
                    eor #$ff
                    sta joystick
                    rts


"Showpic" subroutine
--------------------

Displays the title bitmap picture. Transfers the screen & color data
that has been stored after the bitmap to their correct locations (for
bitmap display, the screen memory resides at $5c00).

    showpic:        ldx #$00
    showpicloop:    lda $8000,x
                    sta $5c00,x
                    lda $8100,x
                    sta $5d00,x
                    lda $8200,x
                    sta $5e00,x
                    lda $8400,x
                    sta $d800,x
                    lda $8500,x
                    sta $d900,x
                    lda $8600,x
                    sta $da00,x
                    inx
                    bne showpicloop
    showpic2:       lda $8300,x
                    sta $5f00,x
                    lda $8700,x
                    sta $db00,x
                    inx
                    cpx #192
                    bne showpic2

Set titlescreen display mode for the "raster1" interrupt.

                    lda #DISPTITLE
                    sta dispmode
                    rts


"Initscreen" subroutine
-----------------------

Sets color registers (background graphics multicolors and sprite
multicolors), turns all sprites multicolored and makes them be display
over the background.  Draws also the initial scorepanel display and
turns it yellow.

    initscreen:     lda #$00
                    sta $d020
                    sta $d021
                    lda #$0e
                    sta $d022
                    lda #$06
                    sta $d023
                    lda #$ff
                    sta $d01c
                    lda #$00
                    sta $d01b
                    lda #$0a
                    sta $d025
                    lda #$00
                    sta $d026
                    ldx #39
    ip_loop:        lda paneltext,x
                    and #$3f
                    sta $400+24*40,x
                    lda #$07
                    sta $d800+24*40,x
                    dex
                    bpl ip_loop
                    rts


"Drawscores" subroutine
-----------------------

Draws all the elements of the status bar, like score, lives, level,
time & hiscore.

    drawscores:     ldy #$02
                    ldx #$02
    ds1:            lda score,x

Get the binary coded decimal at the high 4 bits of the score byte.

                    lsr
                    lsr
                    lsr
                    lsr

Add 48 - character code of '0'.

                    clc
                    adc #48

Store to screen.

                    sta $400+24*40,y
                    iny

Then, get the binary coded decimal at the low 4 bits of the score
byte.

                    lda score,x
                    and #$0f
                    clc
                    adc #48
                    sta $400+24*40,y
                    iny

Loop for all 3 bytes of the score.

                    dex
                    bpl ds1
                    ldx #$02
                    ldy #34

Display the hiscore in a similar fashion.

    ds2:            lda hiscore,x
                    lsr
                    lsr
                    lsr
                    lsr
                    clc
                    adc #48
                    sta $400+24*40,y
                    iny
                    lda hiscore,x
                    and #$0f
                    clc
                    adc #48
                    sta $400+24*40,y
                    iny
                    dex
                    bpl ds2

Display lives. This is just one digit.

                    lda lives
                    clc
                    adc #48
                    sta $400+24*40+14

Display time, that has 2 binary coded digits (similar to what was done
for the score & hiscore).

                    lda time
                    lsr
                    lsr
                    lsr
                    lsr
                    clc
                    adc #48
                    sta $400+24*40+20
                    lda time
                    and #$0f
                    clc
                    adc #48
                    sta $400+24*40+21

Display level number (only one digit)

                    lda level
                    clc
                    adc #48
                    sta $400+24*40+28
                    rts

    paneltext:      dc.b "SC         MEN    TI     LEV    HI      "


"Initraster" subroutine
-----------------------

Activates raster interrupts. "Raster0" interrupt is to be executed
first.

    initraster:     sei
                    lda #<raster0                   ;Set main IRQ vector
                    sta $0314
                    lda #>raster0
                    sta $0315
                    lda #$7f                        ;Set timer interrupt off
                    sta $dc0d
                    lda #$01                        ;Set raster interrupt on
                    sta $d01a
                    lda $d011
                    and #$7f
                    sta $d011
                    lda #RASTER0POS                 ;Set low bits of position
                    sta $d012                       ;for first raster interrupt
                    lda $dc0d                       ;Acknowledge timer interrupt
                    cli                             ;(for safety)
                    rts


"Raster0" interrupt
-------------------

Sets video registers for the display of the score panel (X-scrolling
is stationary and singlecolor, screen memory at $0400-$07ff, videobank
is at $0000-$3fff), plays music and increases "rastercount", then sets
up "raster1" to be executed next.

    raster0:        cld
                    lda #27
                    sta $d011
                    lda #$03
                    sta $dd00
                    lda #21
                    sta $d018
                    lda #8
                    sta $d016
                    inc $d019
                    jsr MUSIC+3
                    lda #<raster1
                    sta $0314
                    lda #>raster1
                    sta $0315
                    lda #RASTER1POS
                    sta $d012
                    inc rastercount
                    jmp $ea81


"Raster1" interrupt
-------------------

Sets video registers according to the displaymode ("dispmode"). The
game screen & textscreen both are at $0400-$07ff, videobank at
$0000-$3fff, but the difference is the X-scrolling: gamescreen has
variable X-finescrolling ("scrollx") while the textscreen has
stationary X-scrolling. The bitmap screen is at videobank $4000-$7fff,
screen memory $5c00-$5fff, with multicolor bitmap graphics. Finally,
"raster0" is set to be executed next to form a loop.

    raster1:        cld
                    lda dispmode
                    beq r1_gamemode
    r1_titlemode:   cmp #DISPTEXT
                    bne r1_pic
                    lda #3
                    sta $dd00
                    lda #27
                    sta $d011
                    lda #$18
                    sta $d016
                    lda #21
                    sta $d018
                    jmp r1_end
    r1_pic:         lda #2
                    sta $dd00
                    lda #59
                    sta $d011
                    lda #24
                    sta $d016
                    lda #$78
                    sta $d018
                    lda #$00
                    sta $d015
                    jmp r1_end
    r1_gamemode:    lda #$03
                    sta $dd00
                    lda #27
                    sta $d011
                    lda #30
                    sta $d018
                    lda scrollx
                    and #$07
                    ora #$10
                    sta $d016
    r1_end:         inc $d019
                    lda #<raster0
                    sta $0314
                    lda #>raster0
                    sta $0315
                    lda #RASTER0POS
                    sta $d012
                    jmp $ea81


The variables
-------------

General variables:

    score:          dc.b 0,0,0
    hiscore:        dc.b 0,0,0
    lives:          dc.b 3
    level:          dc.b 1
    time:           dc.b $99
    timedl:         dc.b 0
    firedelay:      dc.b 0
    killactive:     dc.b 0
    killmeter:      dc.b 0
    killmeterd:     dc.b 0
    killlimit:      dc.b 0

Actor variable arrays:

    actxl:          ds.b 8,0
    actxh:          ds.b 8,0
    acty:           ds.b 8,0
    actf:           ds.b 8,0
    actfd:          ds.b 8,0
    actd:           ds.b 8,0
    actsx:          ds.b 8,0
    actsy:          ds.b 8,0
    actyd:          ds.b 8,0
    actj:           ds.b 8,0
    actt:           ds.b 8,0
    actimm:         ds.b 8,0
    acthp:          ds.b 8,0

Tables for properties of different actor types:

X- and Y-hotspots (centers within the sprite):

    acthotx:        dc.b 12,12,12,12,24,24
    acthoty:        dc.b 40,40,40,40,40,40

X- and Y-magnification:

    actmagx:        dc.b 0,0,0,0,1,1
    actmagy:        dc.b 1,1,1,1,1,1

X- and Y-sizes:

    actsizex:       dc.b 12,12,12,12,30,48
    actsizey:       dc.b 42,42,42,42,32,42

Colors:

    actcolor:       dc.b 11,9,2,4,12,7

Base frames facing left and right:

    actbaseframer:  dc.b 128,144,144,144,152,156
    actbaseframel:  dc.b 136,148,148,148,154,156

Addresses of move routines:

    actroutlo:      dc.b <plr,<man,<man,<man,<mc,<expl
    actrouthi:      dc.b >plr,>man,>man,>man,>mc,>expl

Shooting Y-coord modification:

    actshootymod:   dc.b 20,20,20,20,20,0

Initial hitpoints:

    actmaxhp:       dc.b 1,1,2,3,5,0

Score for killing an enemy:

    actscorelo:     dc.b $00,$50,$50,$00,$00,$00
    actscorehi:     dc.b $00,$02,$04,$06,$10,$00

Bullet variable arrays:

    bulletxl:       ds.b 16,0
    bulletxh:       ds.b 16,0
    bullety:        ds.b 16,0
    bulletd:        ds.b 16,0
    bullett:        ds.b 16,0
    bulletlo:       ds.b 16,0
    bullethi:       ds.b 16,0
    bulletunder:     ds.b 16,0


Included binary data
--------------------

The sprites:

                    org SPRITES
                    incbin efny.spr

The chars: (the char-collision data saved by BGEDIT is unused)

                    org CHARS-$100
                    incbin efny.chr

The music, made with SadoTracker:

                    org MUSIC
                    incbin music.bin

The background map data (map-header saved by BGEDIT is unused)

                    org MAP-2
                    incbin efny.map

The blocks (block-color data saved by BGEDIT is unused)

                    org BLOCKS-$80
    blocks:         incbin efny.blk

The title bitmap picture:

                    org PICTURE
                    incbin plissken.pic


So, there we have reached the end of the Escape From New York
sourcecode, and almost the end of this rant. But for a closing I'll
explain how the music was extracted, and the commands in the makefile.

The music was saved with the pack/relocate option of SadoTracker on a
D64 image (EFNYMUS.D64), starting from address $4000. Then, that .PRG
file was extracted from the disk image (don't remember what utility I
used back then, today I would use D642PRG in my commandline-utility
collection). The EFNY+PLAYER.PRG file was then converted to a raw
binary file MUSIC.BIN (without start address) with the PRG2BIN
utility.

The makefile commands:

- EFNY.PRG depends on the source code, on the IFF/LBM title picture
  and the music binary:

    `efny.prg: efny.s efny.lbm music.bin`

- Execute the BENTON64 picture conversion utility, save the picture as
  a raw binary file PLISSKEN.PIC with bitmap data (8kb) followed by
  screen data (1kb) and color memory data (1kb)

    `benton64 efny.lbm plissken.pic -r`

- Assemble the source code with DASM, output file is EFNY.PRG. Use
  verbose mode and maximum of 3 passes.

    `dasm efny.s -oefny.prg -v3 -p3`

- Compress the output file with PUCRUNCH (get it at
  http://www.cs.tut.fi/~albert/Dev/pucrunch/) with execution start
  address set at 2048.

    `pucrunch -x2048 efny.prg efny.prg`

If you are interested, you can examine the sprite file EFNY.SPR with
SPREDIT and the background data with BGEDIT (fastest is to press F9 to
"load all leveldata" and type EFNY)

End of rant.
