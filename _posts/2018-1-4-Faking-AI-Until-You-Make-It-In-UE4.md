---
layout: post
title: Faking AI Until You Make it in UE4
image: https://cdn2.unrealengine.com/Unreal+Engine%2FUE-Logo-988x988-1dee3bc7f6714edf3c21ee71826ebab54ae02077.png
---

If you are a follower of Troll Purse, then you well know that our current game [Eight Hours](http://trollpurse.com/eighthours.html) is built using [Epic Game's](https://www.epicgames.com) [Unreal Engine 4](https://www.unrealengine.com/). Today, we will discuss doors. Yes, doors and how our AI opens them - or rather, doesn't.

![Eight Hours Banner](http://media.indiedb.com/images/games/1/61/60992/AppLogo_1176x662.png "Eight Hours Logo")

# Quick Overview

Eight Hours is a horror game about a paranormal investigator called in to investigate a house where the owners mysteriously and rapidly vacated the premises. Rather than just using the standard puzzle adventure with a horror theme, Eight Hours includes action involving ghosts. There is an entity that will chase you - when caught, the game is over.

The special qualities of the entity are as follows. The entity cannot exist in lighted areas, will spawn at an increased rate as more doors are opened and lights are on, and is invisible.

The first two qualities are those triggered and effected by how the player plays the game. 

Rather than the trope of a defenseless protagonist who cannot fight the powers that be, the player is able to fight back. A way the player can fight back is to banish the entity using the lights in the house. Other ways are to reduce risk of summoning the entity by keeping the lights off and the doors closed. Finally, the player can strategically place a limited number of sensors that will warn the player of the entity's presence within an area.

Other than the player deciding how to fight or hide from the entity, the entity can exist without being visible.

The entity is a ghost - a supernatural being that exists on another plane of existence. In the game, the ghost is invisible. There are the afore mentioned ways to detect the ghost; however, to aid in gameplay frustration, the ghost also has four other tells. These tells are casting a shadow within a lighted area before being banished, closing random doors in the house, shutting off random lights in the house, and opening doors it passes through.

The final item in the list, opening doors the ghost passes through, is what will be discussed in this blog post.

# Faking Door Interactions with AI

Unreal Engine 4 has an excellent [series on building game ready AI](https://www.youtube.com/watch?v=PgxuaTSkyu4) - it is awesome and helpful. Epic Games also has some [great documentation on AI pathing]( https://docs.unrealengine.com/latest/INT/Resources/ContentExamples/NavMesh/1_1/). Troll Purse considered both of these features and though, "Wow, this is really deep and useful stuff for AI; however, our AI just chases the player - do we need it to be that 'smart'?"

The answer was a resounding "No". [Mostly inspired by this blog article about "smart" AI](http://askagamedev.tumblr.com/post/76972636953/game-development-myths-players-want-smart).

So, what did the Troll Purse do? Well, it was all a happy little accident. See, ghosts are (as the mythos goes) ethereal. They cannot interact with the physical realm (our world) unless willfully using energy to engage with objects. This meant that our collision layers have a specific `GHOST` layer, ignored by everything but the floors (so the ghost can walk and path - really don't need it flying). This allowed the ghost to pass through everything but the walls of the house (as we wanted some rules for gameplay purposes). 

This meant doors weren't a problem. The ghost would just pass through. But how did Troll Purse developers make the ghost open doors as it approached? Well, rather than using [EQS](https://docs.unrealengine.com/latest/INT/Engine/AI/EnvironmentQuerySystem/), [Behaviour Trees](https://docs.unrealengine.com/latest/INT/Engine/AI/BehaviorTrees/QuickStart/), and [Tasks](https://docs.unrealengine.com/latest/INT/Engine/AI/BehaviorTrees/QuickStart/12/index.html), we just added a proxy detection for the `GHOST` layer to the door. When overlapped, the door opens itself. So, it isn't that the ghost is opening the door, rather, it is the presence of the ghost that opens the door. What this does is that, sans the animation, it seems as if the ghost is opening the door.

Perhaps this simple implementation is something that one would want to do in a quick prototype. The steps were as follows:

1. Setup a [collision layer](https://www.unrealengine.com/en-US/blog/collision-filtering) for the doors and the AI Character
2. Configure the Door and AI Character to [Overlap](https://docs.unrealengine.com/latest/INT/Engine/Physics/Collision/Overview/), instead of Block
3. As the AI Character approaches, or even passes through the door, fire an [Overlap Begin](https://docs.unrealengine.com/latest/INT/Engine/Blueprints/UserGuide/Events/) event on the door to call the open subroutine.

> Easy Peasy Lemon Squeezy - *Sqezy*

# That’s All Folks

Yup, Unreal Engine 4 has some awesome AI features for bigger games that we would love to use. However, one cannot forget to [KISS](https://en.wikipedia.org/wiki/KISS_principle). Sometimes, faking AI is better than making AI.