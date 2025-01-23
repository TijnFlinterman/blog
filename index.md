---
layout: default
---

## Part 1: Character controller

In this 2 part blog I will explain the theory and implementation of a basic character controller and add the advanced movement mechanic along side it. Wall running, This will spice up any game by giving the player a lot more freedom to explore your game world.

Part 1: [Character controller](./index.html) \
Part 2: [Wall running](./another-page.html)

In this first part we talk about the difference between kinematic and dynamic rigidbodies for character controllers. Show you a few examples from different media and show you my approach on the topic.\
At the end of this part you will have a kinematic character controller with a first person camera that sweeps the area in the desired player input direction and handles the collision detection and resolves it. This same system will be easily applicable for gravity as well.

If you are using a game engine like Unity then you can skip this first part and go directly to Part 2: [wall running](./another-page.html).


### Difference between kinematic and dynamic

Typically in games, character controller are either kinematic or dynamic.

#### dynamic
The difference is that dynamic character controllers move their characters using forces resolved purely by the physics engine. \
Examples of this are seen in games like:
- Gang Beasts
- Human: Fall Flat
- Boneworks
- Totally Accurate Battle Simulator 

These games often make it their USP (unique selling point) that their game characters are physics driven.
Games like "Gang Beasts" and "Human: Fall Flat" use that slightly unpredictable chaos of using forces to drive the player to create funny interactions with the environment and other players.

#### kinematic
Kinematic character controllers move their players directly without passing input through the physics engine first. the player directly effects the character. kinematic character controllers aren't moved by forces. physics is sort of faked in these types of games.

Examples of this are seen in games like:
- Call of Duty
- Titanfall
- Grand Theft Auto 5
- Super Mario 64

This is the more common option since this type of character controller is a whole lot more predictable and often more in line of what the player expects how the character should move.

### Basic kinematic charcter controller

Since kinematic rigidbodies phase through walls when colliding since the normal collision resolve of the physics engine don't apply to to kinematic bodies we need to create a fix for this.\
When creating the physics object you will probably give it a "capsule" shape. this is roughly the shape of a human in terms for proportions.\
Then when you have created your capsule shape and locked the capsule from rotating along the X and Y axis then it is time to handle input.
```c++
// This is the code you need for an engine where Y is the up axis.
glm::vec3 forward = glm::normalize(glm::vec3(cos(cameraYaw), 0.0f, sin(cameraYaw)));
glm::vec3 right = glm::normalize(glm::cross(forward, glm::vec3(0.0f, 1.0f, 0.0f)));

// This is the code you need for an engine where Z is the up axis.
glm::vec3 forward = glm::normalize(glm::vec3(cos(cameraYaw), sin(cameraYaw), 0.0f));
glm::vec3 right = glm::normalize(glm::cross(forward, glm::vec3(0.0f, 0.0f, 1.0f)));

// The rest of the code is unaffected related if Y or Z is up
glm::vec3 movementInput(0.0f);
if (Engine.Input().GetKeyboardKey(Input::KeyboardKey::W))
	movementInput += forward;
if (Engine.Input().GetKeyboardKey(Input::KeyboardKey::S))
	movementInput -= forward;
if (Engine.Input().GetKeyboardKey(Input::KeyboardKey::A))
	movementInput -= right;
if (Engine.Input().GetKeyboardKey(Input::KeyboardKey::D))
	movementInput += right;

// You need to normalize the vector length since if you don't do this,
// the player will move faster diagonal then when only pressing 1 movement key.
if (glm::length(movementInput) > 0.0f)
	movementInput = glm::normalize(movementInput); 
```
Since we are making an FPS controller when holding W, it should be relative to the camera's direction.
So you need to get the camera yaw to see what direction the player is facing.





#### Header 4

*   This is an unordered list following a header.
*   This is an unordered list following a header.
*   This is an unordered list following a header.

##### Header 5

1.  This is an ordered list following a header.
2.  This is an ordered list following a header.
3.  This is an ordered list following a header.

###### Header 6

| head1        | head two          | three |
|:-------------|:------------------|:------|
| ok           | good swedish fish | nice  |
| out of stock | good and plenty   | nice  |
| ok           | good `oreos`      | hmm   |
| ok           | good `zoute` drop | yumm  |

### There's a horizontal rule below this.

* * *

### Here is an unordered list:

*   Item foo
*   Item bar
*   Item baz
*   Item zip

### And an ordered list:

1.  Item one
1.  Item two
1.  Item three
1.  Item four

### And a nested list:

- level 1 item
  - level 2 item
  - level 2 item
    - level 3 item
    - level 3 item
- level 1 item
  - level 2 item
  - level 2 item
  - level 2 item
- level 1 item
  - level 2 item
  - level 2 item
- level 1 item

### Small image

![Octocat](https://github.githubassets.com/images/icons/emoji/octocat.png)

### Large image

![Branching](https://guides.github.com/activities/hello-world/branching.png)


### Definition lists can be used with HTML syntax.

<dl>
<dt>Name</dt>
<dd>Godzilla</dd>
<dt>Born</dt>
<dd>1952</dd>
<dt>Birthplace</dt>
<dd>Japan</dd>
<dt>Color</dt>
<dd>Green</dd>
</dl>

```
Long, single-line code blocks should not wrap. They should horizontally scroll if they are too long. This line should be long enough to demonstrate this.
```

```
The final element.
```

[Part 2: wall running](./another-page.html).

Sources:

http://www.peroxide.dk/tuts_c.shtml