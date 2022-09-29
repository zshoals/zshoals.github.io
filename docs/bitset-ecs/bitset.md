# **A Bitset Entity-Component-System in C99**

# What is an ECS
If you're reading this, you probably know what an Entity-Component-System (ECS) is already. If you need a really giant overview of this concept, check out this [ECS FAQ](https://github.com/SanderMertens/ecs-faq) However, here's a quick refresher.

Implementations differ, but the modern, generalized concept is a bunch of arrays storing contiguous or at least sequential data of the same type, say, 500 floats in a row. An **entity**, usually just an integer of some sort, indexes into these data arrays, otherwise known as **components**, in some fashion. There may or may not be a concept of a **tag**, which is almost the same as a component except without any associated data (a tag's presence essentially acts as a boolean flag). **Systems** are processing functions, usually accompanied by some sort of **query** that selects the entities owning certain components that you are actually interested in operating on within a particular system.

# Why Make Another One?
There are plenty of existing ECS solutions around the web (probably one of the best being [Flecs](https://github.com/SanderMertens/flecs)) Many of them are incredible feats of engineering! Many of them are also very complicated, and lean very far from the *actual* goal of what your average casual game developer likely wants: a conceptually simple and decently performant way to filter for the particular game objects they would like to work with at any given time. This really should not take a multi-thousand line codebase to achieve, and we should be able to keep our implementation in our heads. We shouldn't have to worry about "deferred updates" or multithreading or relationship graphs or whatever.

Let's keep it simple, and design something we won't be afraid to use.

# The Approach
There are many varieties or ECS designs, and you can research some of these keywords: Archetype, Sparse Set, Bitset, Reactive. We're going to focus on "Bitset," as it's easy to implement and I've never seen any real writeups behind this design anywhere. However, I know of three ECS implementations using this approach in some form:

- [ECX](https://github.com/eliasku/ecx)
- [Specs](https://github.com/amethyst/specs)
- [EntityX](https://github.com/alecthomas/entityx)

The general idea behind this bitset design is fairly simple but it can initially be a lot to take in. Let's break it down:

Entities will be represented as an index into a particular component array slot, combined with a **generation**. Both the entity ID/handle and generation will be stored in a single unsigned 64-bit integer. The ID portion will be stored in the lower bits of the integer, and the generation in the upper bits of the integer. How many bits are consumed by the ID and how many by the generation are up to you, but general limitations of the bitset ECS design will likely lead you to have a rather small ID limit and a very massive generation limit.

Additionally, entities will be supported by a single **bitset**, essentially a very compact list of boolean flags. The bitset will consist of (MAX_ENTITIES / 32) worth of unsigned 32-bit integers, which will be the backing storage for these bools. This bitset will indicate whether an entity is alive or dead, or alternatively active or inactive. Note that this bitset is NOT storing which components that each entity has, although other bitset ECSs sometimes use this approach. 

Components will be one-time-allocated data arrays consisting of MAX_ENTITIES worth of data elements. This will be the case for every single component. Each component will have an accompanying bitset, an array with (MAX_ENTITIES / 32) worth of unsigned 32-bit integers. These bitsets will represent whether or not an entity has a particular component...but it's more appropriate to think of the component having a particular entity.

Systems can do anything; they're just processing functions, so you might see operations like modifying component data via an entity index, creating new entities, deleting entities, interfacing with other functionality of a library like SDL, whatever. They're just functions. A query will be a series of bitwise ands, ors, and nots, .

//TODO: MOVE ME
A generation is a value that represents a reused entity as a conceptually new object. The generation concept arises naturally from a pretty major problem; pretend we store Entity_A in a component to use for later, maybe as a pathfinding target for another entity. If a random system kills/deletes/removes/recycles Entity_A and it becomes Entity_A++, and we attempt to use the stored Entity_A...what happens? Entity_A is dead; it's been replaced with Entity_A++. We need a way to distinguish between the two, and that's the problem the generation solves. 

Believe it or not, this is the only reason you need a generation at least as far as I can tell. If your game design restrictions



