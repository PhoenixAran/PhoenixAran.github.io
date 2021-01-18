---
---
In this post I'll be documenting how I implemented the player controller in [Oracle of Love](https://github.com/PhoenixAran/LoveOracle). This is more to cement my own understanding of it so it will probably be confusing and abstract to other readers. You have been warned before you read this wall of text.

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
The difference between the two? The ability to jump.

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
class Player extends MapEntity {
    // To handle movement and pick movement animation
    var playerMovementController
    // State Machines for the player
    var environmentStateMachine
    var controlStateMachine
    var weaponStateMachine
    var conditionStateMachines = []
    // Player State Parameters that get mutated by each active state
    var stateParameters = PlayerStateParameters()

    function init() {
        environmentStateMachine = StateMachine(self)
        playerMovementController = PlayerMovementController(self)
        ...

    }
}
~~~
_Player entity pseudo code_

The diagram below will outline the what goes on in **Player::update()** each frame. I will also provide explanations for what each step in the flowchart does.

![Basic StateMachine](/assets/images/player_state_update_high_level.png)  
_Update process for the player entity_  

### Request Natural State and Updating Movement Values
When we request the player's natural state, we are asking the following questions:
* Is the player in the air?
* Is the player in high grass
* Is the player in water
* ... etc.  
  
Once we find out where the player is in the environment, we retrieve the corresponding environment state object. Environment state objects just adjust player movement
values when it is active. For example, if a player is on ice, the _Ice Environment State_ will just make the player's movement more slippery.


### Integrate State Parameters
The player entity will have one instance of the _Player State Parameter_ struct. 

~~~
class PlayerStateParameters {

    var canJump = true;
    var canWarp = true;
    var canLedgeJump = true;
    var canControlOnGround = true;
    var canControlInAir = true;
    var canPush = true;
    var canUseWeapons = true;
    var canStrafe = true;
    var defaultAnimationWhileStill = true;

    var alwaysFaceUp = false;
    var alwaysFaceDown = false;
    var alwaysFaceLeft = false;
    var alwaysFaceRight = false;

    var animations = {
        'swing' = nil,
        'swingNoLunge' = nil,
        'swingBig' = nil,
        'spin' = nil,
        'stab' =  nil,
        'aim' = nil,
        'throw' - nil,
        'default' = nil,
        'move' = nil,
        'carry' = nil,
        'count' = nil
    };
}
~~~

Each state will have their own configuration for State Parameters. When we _integrate_ state parameters, we run a routine inside the Player entity
that looks like this:

~~~
class Player extends MapEntity {

    function integrateStateParameters() {
        self.stateParameters = PlayerStateParameters();
        // declare your default state parameter values
        self.stateParameters.animations.default = 'idle';
        ...
        foreach(var stateMachine in self.conditionStateMachines) {
            if (stateMachine.isActive())
                self.stateParameters.integrateStateParmeters(stateMachine.getStateParameters());
        }
        self.stateParameters.integrateStateParameters(self.environmentStateMachine.getStateParameters());
        self.stateParameters.integrateStateParameters(self.controlStateMachine.getStateParameters());
        self.stateParameters.integrateStateParameters(self.weaponStateMachine.getStateParameters());
    }
}
~~~

**PlayerStateParameters::integrateStateParameters** is a function that combines state parameter values from another instance of PlayerStateParameter struct.  
It will run compare itself with the other instance, and set it's own flags depending on the values in the other struct. For example, if we integrate a PlayerStateParameter struct with the flag _canMove_ as false, it will set it's own flag _canMove_ to false as well.  It will also override it's own animation declarations with the instance it is integrating with.  

~~~
class PlayerStateParameters {

    function prioritizeFalse(a : bool, b : bool) {
        if (!a) { return false; }
        if (!b) { return false; }
        return true;
    }

    function integrateStateParameters(other : PlayerStateParameters) {
        // for player actions, we want to prioritize restrictions
        self.canJump = prioritizeFalse(self.canJump, other.canJump);
        self.canWarp = prioritizeFalse(self.canWarp, other.canWarp);
        ...

        // prefer true for clamping animation directions
        self.alwaysFaceUp = self.alwaysFaceUp || other.alwaysFaceUp;
        self.alwaysFaceDown = self.alwaysFaceDown || other.alwaysFaceDown;
        
        // prefer the other animations if they are not null
        foreach(keyValuePair in other.animations) {
            var key = keyValuePair.key;
            self.animations[key] = keyValuePair.value or self.animations[key];
        }
    }
}
~~~

This part is a little confusing but to sum it up, integrating state parameters is determining what the player is allowed to do by combining the constrictions defined by
each active Player state. So if the active weapon state says that "the player cannot move when it is in use", that rule will set the flag _canMove_ to false which the 
PlayerMotionController will then have to respect.


### Get Move Controls
This is when the movement controller will intrepret movement controls from the player based on the Player's PlayerStateParameter struct instance. If 
player is allowed to move, the movement controller will set motion values in the physics component to declare how the player will move.  

### Choose Animations
Pick what animation to play based on what the player is currently doing and which direction they are facing. 

### Check pressed buttons callbacks
One pretty cool thing about this player controller implementation is that it allows for dynamic callbacks to be used for each individual buttons. Here is what it looks
like when we button callbacks are assigned.

~~~
addPressInteraction('x', function(player) {
    // when the player requests the natural state, it will be in the air
    // therefore making the JumpState the active Environment State
    movementController.jump();   
});

addPressInteraction('x', function(player) {
    // attempt to read sign / talk to NPC
    player.interact();
});

addPressInteraction('b', function(player) {
    player.actionUseWeapon('b');
});
~~~

These callbacks get looped through depending if their coresponding button is pressed. 

### Move Player
Move your player within your physics system.

### Update Equipped Items
Items in my engine are their own entities whose update and draw methods get called by their owner entity. Update them in this step.

### Update Other Components
Update the other components like the sprite component, or combat component.

## The Result
After implementing the controller this way, the result was slightly less confusing code. But the benefit was that the ugly code is only in a couple of places instead of inside multiple monolithic state objects.  