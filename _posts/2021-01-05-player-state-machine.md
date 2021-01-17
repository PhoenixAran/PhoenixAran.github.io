---
---
When coding the player entity for the _Oracle of Love_ engine, I ran into a wall trying to figure out the best way to handle 
all the different player states. If you never played _Links Awakening_ or any of the _Oracle_ games, here is a high level list of what the player can do.

1. Walk
2. Jump
3. Swim
4. Glide (cape)
5. Use Equipped Item
6. Carry Objects
7. Throw Objects
8. Push Objects

If you ever heard of the state machine design pattern, you would probably assume that it would be perfect for this situation. 

![Basic StateMachine](/assets/images/state_flowchart.png)  
*If you never heard of a state machine, I recommend reading this [article from Game Programming Patterns](https://gameprogrammingpatterns.com/state.html){:target="_blank"}* 

However, a tradtional state machine did not work out in actual practice. Here are some of the short comings of the traditional state machine approach I encountered.

### Concurrent States
The amount of _states_ that exist for the player in the Gameboy Zelda games is actually pretty complex. 
I would even go as far as to say that is is more complex than recreating the player code in _Link to the Past_. 
The difference between the _Links Awakening/Oracle_ player and the _Link to the Past_ player? The ability to jump.

Jumping introduces alot of complexity when programming a player controller. If you were to keep using a traditional state machine approach, you would have to _double_
the amount of state objects to implement. _Sword Swing_ would become _Sword Swing_ and _Jump Sword Swing_.

Another solution would be to make the items have their own _update_ method so the item can check if the player is in the _Jump_ state if it needs to. This introduces other kinds of complexity such as trying to implement logic like *the player can move, but not change animation direction while charging the sword.* You could implement a _Sword Loading_ state, but then what happens when you need to let the player _jump_ while charging the sword? Would you create a _Jump Sword Load_ state, or would you stuff the logic into the item code? 

### Choosing Animations
Choosing animations is a nightmare. To refer back to the problem above for the _sword loading_ state, where exactly do you choose the animation? For simpler games a state object would have a virtual **State::onBegin()** method you can override and have it done there. But with so many concurrent states and combinations of logic like _you can move but you can't change the animation direction_, it becomes a huge head ache.  

This is not to say it can't be done. Im pretty certain it could be if you're smart (unlike myself), but I could not figure it out. 

## Alternative Approach to Using State Machines
Before I go into this, I would like to credit [this Zelda recreation project](https://github.com/trigger-segfault/ZeldaOracle) for the original implementation. I borrowed some of their player code and adapted it to my engine, so I think I have an okay understanding of how it works.    

## Setting up the State Machines  
The basic idea is to split up your state machines. Instead of having a state machine to control the player as whole, create state machines for different aspects of the player.

~~~ 
class Player extends MapEntity

// To handle movement and pick movement animation
var playerMovementController
// State Machines for the player
var environmentStateMachine
var controlStateMachine
var weaponStateMachine
var conditionStateMachines = []
// Player State Parameters that get mutated by each active state
var stateParameters = PlayerStateParameters()

func init():
    environmentStateMachine = StateMachine(self)
    playerMovementController = PlayerMovementController(self)
    ...
</pre>
~~~
_Player entity pseudo code_

The diagram below will outline the what goes on in **Player::update()** each frame.

![Basic StateMachine](/assets/images/player_state_update_high_level.png)  
_Update process for the player entity_  

#### Request Natural State and Updating Movement Values
When we request the player's natural state, we are asking the following questions:
* Is the player in the air?
* Is the player in high grass
* Is the player in water
* ... etc.
Once we find out where the player is in the environment, we retrieve the corresponding environent state object. Environment state objects just adjust player movement
values when it is active. For example, if a player is on ice, the _Ice Environment State_ will just make the player's movement more slippery.

#### Get Move Controls
