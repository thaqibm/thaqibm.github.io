---
layout: page
title: AGE Game Engine
description: An Entity Component System based game engine written in C++17
img: assets/img/age.png
importance: 1
category: Fun
github: https://github.com/thaqibm/age
---

# Game Engine Overview 
The game executes the update method each cycle and these are the following things that happen:

{% raw %}
```cpp
/**
* Update cycle:
* - Detect Collisions (Collision System)
* - Get input (Controller)
* - Pass collision data to all systems (system_base)
* -Pass input to all systems (system_base)
* - Call update on each system
* - Render (Render System)
**/
```
{% endraw %}

## Entity
An entity is just a index of a game object. 
Each EntityID is just a unique 32 bit integer. The maximum number of entities that the game can have
must be determined at *compile time*.
```cpp
using EntityID = std::uint32_t;
static const constexpr std::size_t MAX_ENTT = 5000;
```

## Component
A Component is any piece of pure data. A user can attach any component to an entity and its behavior
will change based on the data added. For example the some physics based components are:

{% raw %}
```cpp
// Components/default/default.h
struct Vec2d{ float x; float y; };
struct Position{
Vec2d p;
float z; // height
float len;
float width; };
struct Velocity{ Vec2d v; };
struct Acceleration{ Vec2d a; };
```
{% endraw %}

These are just pure pieces of POD data the user can attach to any entity and their behavior / movement
will change based on that. Attaching / Detaching components to an entity is done with templates and
to make it easier to save predefined data I’ve created a prefab template class.

{% raw %}
```cpp
template<typename T> void attachComponent(EntityID id, T component)
template<typename T> void detachComponent(EntityID id)
// Examples:
GAME1.attachComponent<PlayerTag>(player {GAME1.createEntityFromPrefab(bullet), 0});
GAME1.attachComponent<AGE_COMPONENTS::Velocity>(er, {{0,0}});
GAME1.attachComponent<AGE_COMPONENTS::Solid_tag>(er, {});
```
{% endraw %}

### Prefabs
This is a feature to make it easier to add stored entities with predefined data. So the user does not need
to type out attach every time. For example the bullet prefab in the example above is defined as:

{% raw %}
```cpp
// Prefab.h
Prefab<AGE_COMPONENTS::Position, AGE_COMPONENTS::Drawable, AGE_COMPONENTS::Velocity, AGE_COMPONENTS::acceleration, 
    AGE_COMPONENTS::BoxCollider, AGE_COMPONENTS::Mass> bullet(
            "bullet",
            {{-1,-1},0, 0,0},
            {{{’o’}}},
            {{0,0}},
            {{0,0}},
            {1,1},
            {1.0f}
            );
```
{% endraw %}

## System 
A System is the core logic and it handles mutating game state / movement of entities and the core game
logic.
Any system the user defines must have an update method and they can decide if they need an end()
method which is called for each system at the end of the game. To enforce this on user defined systems
the inheritance was the best strategy. Any system must be derived from system base class. It also
provides interfaces the user can access to get the current Input, Collision data for their system to react.
A system can only run and modify entities that have a specified signature. That is they must have a
specific list of components added to them. This information is represented in a compile time bitset since
every time a user attaches a component they call a template function that is what makes it possible to
make these bit sets at compile time.

Example physics system:
{% raw %}
```cpp
bool update() override{
    float dt = DeltaTime(); // provided by System_base
    for(const auto& et : Entities){ // contains all entities with required signature
        auto& pos = ComponentData.getComponentData<AGE_COMPONENTS::Position>(et);
        auto& vel = ComponentData.getComponentData<AGE_COMPONENTS::Velocity>(et);
        auto& acc = ComponentData.getComponentData<AGE_COMPONENTS::Acceleration>(et);
        auto v0 = vel.v;
        vel.v += acc.a*(dt);
        pos.p += (vel.v + v0)*0.5*dt;
    }
    return true;
}
```
{% endraw %}

More on [AGE Github](https://github.com/thaqibm/age)

