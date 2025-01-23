---
layout: default
---

## Part 2: Wall running

In this 2 part blog I will explain the theory and implementation of a basic character controller and add the advanced movement mechanic along side it. Wall running, This will spice up any game by giving the player a lot more freedom to explore your game world.

Part 1: [Character controller](./wall-running-technical-breakdown-part-1.html) \
Part 2: [Wall running](./wall-running-technical-breakdown-part-2.html)

In this second part we will talk about how you can take your character controller and add the wall run mechanic to it. I will first take a look behind the theory behind it and afterwards show you an implementation of it.\
At the end of this part you will have on top of your character controller, one that can at runtime detect "walls" and walk on them.
On top of that you as the developer can also decide what should be the parameters to initialize and end a wall run.

### Difference between pre-calculated and runtime wall runtime

There are 2 approaches to detecting if a wall is suitable to be ran on. We are gonna take a quick look why I chose the method I chose and the advantages and disadvantages of both.

Pre-calculating wall detection means that before loading into a level. Each wall in the level contains data related to it being able to be ran on.  
runtime wall detection means that you can give everything in the level a collider and then it works. A plug and play approach.

pre-calculated\
Pros:
* performance efficiency
* better for larger sized levels
* much more stability within your world.
* more predictable paths the user has to take.

Cons:
* more storage cost to save map data
* increased development time
* no live updating scenes.
* not a "plug and play" type system since it relies on setting up the environment correctly.

runtime\
Pros:
* can increase the players fun due to new usages being found
* system supports dynamic terrain such as broken buildings.
* does have a "plug and play" type system that isn't reliant on the environment setup.

Cons:
* less predictable what the user will do with it and can lead to unwanted gameplay outcomes.
* higher performance cost
* a lot more prone to bugs.
* players can take paths you didn't intend to be possible and exploit.

### wall running

Wall running is a niche movement mechanic often seen in games like Mirror's Edge Catalyst, Titanfall 2, Call of Duty: Black Ops III and ProtoType 2.
In Titanfall series it is a staple peace of gameplay. There is an entire genre called "Movement-Shooters" which follows roughly the same idea.

I'm here to recreate a extremely dumbed down version of the Titanfall movement system.

##### Theory

Wall running is a feature in games where the player can run on walls.
These walls are 99% of the time perfectly upright.



##### Implementation


#### sticking and unsticking

Sticking and unsticking to the wall to start and stop wall running is something that should be intuitive to the player.
There are a few suggestions I have to handle the design of the player sticking to the wall and unsticking.

sticking:
1. Only start wall run a short delay after jumping off the floor. to make you not wall run super low on the wall on accident.
2. Only when running along with the wall. Compare the face normal and player moving direction. This stops you from going face first in a wall and starting the wall run anyway.
3. When player is looking the right direction. You don't want to start wall running backwards.
4. When having enough speed to start it. You're making a wall run, not a wall walk (duh).

unsticking
1. If player looks to far away from the forward running direction.
2. If the player presses jump.
3. If player is running too long in one go without changing walls.
4. When having not enough to continue wall running.

Sources:

[Character Controllers - NVIDIA](https://docs.nvidia.com/gameworks/content/gameworkslibrary/physx/guide/Manual/CharacterControllers.html)\
[Moving Characters in Games – Kinematic Character Controller in Unity - Nick Maltbie](https://youtu.be/s-99Z_W8bcQ?si=ylJKyuFEmlmsqH07)\
[Collide And Slide - *Actually Decent* Character Collision From Scratch - Poke Dev](https://youtu.be/YR6Q7dUz2uk?si=HxrokkEFIoWsMxGK)\
[Improving the Numerical Robustness of Sphere Swept Collision Detection - Jeff Linahan](https://arxiv.org/pdf/1211.0059)\
[Improved Collision detection and Response - Kasper Fauerby](http://www.peroxide.dk/papers/collision/collision.pdf)\
[Why Is Titanfall 2’s Movement System So Good? - Callum Gibson](https://claritypotion.com/2022/07/11/titanfall-movement-system-so-good/)\
[Designing Unforgettable Titanfall Single Player Levels with Action Blocks - Christopher Dionne](https://www.gdcvault.com/play/1025105/Designing-Unforgettable-Titanfall-Single-Player)\
[How Titanfall 2 Made Movement the Star of the Show - Art of the Level - IGN](https://youtu.be/jajgleIR9tI?si=IzHN2Yixv5PZF8n0)\
[Titanfall 2 how design informs speed - Abhishekiyer](https://medium.com/@abhishekiyer_25378/titanfall-2-how-design-informs-speed-f14998d7f470)