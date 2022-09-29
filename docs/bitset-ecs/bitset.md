# **A Bitset Entity-Component-System in C99**

# What is an ECS
If you're reading this, you probably know what an Entity-Component-System (ECS) is already. If you need a really giant overview of this concept, check out this [ECS FAQ](https://github.com/SanderMertens/ecs-faq) However, here's a quick refresher.

Implementations differ, but the modern, generalized concept is a bunch of arrays storing contiguous or at least sequential data of the same type, say, 500 floats in a row. An **entity**, usually just an integer of some sort, indexes into these data arrays, otherwise known as **components**, in some fashion. There may or may not be a concept of a **tag**, which is almost the same as a component except without any associated data (a tag's presence essentially acts as a boolean flag). **Systems** are processing functions, usually accompanied by some sort of **query** that selects the entities owning certain components that you are actually interested in operating on within a particular system. There may be a **world**, which acts as a container that holds component arrays and entity data. A world is technically optional, but this concept arises due to issues pertaining to entity deletion.

# Why Make Another One?
There are plenty of existing ECS solutions around the web (probably one of the best being [Flecs](https://github.com/SanderMertens/flecs)) Many of them are incredible feats of engineering! Many of them are also very complicated, and lean very far from the *actual* goal of what your average casual game developer likely wants: a conceptually simple and decently performant way to filter for the particular game objects they would like to work with at any given time. This really should not take a multi-thousand line codebase to achieve, and we should be able to keep our implementation in our heads. We shouldn't have to worry about "deferred updates" or multithreading or relationship graphs or whatever.

Let's keep it simple, and design something we won't be afraid to use.

# The Approach
There are many varieties or ECS designs, and you can research some of these keywords: Archetype, Sparse Set, Bitset, Reactive. We're going to focus on "Bitset," as it's easy to implement and I've never seen any real writeups behind this design anywhere. However, I know of three ECS implementations using this approach in some form:

- [ECX](https://github.com/eliasku/ecx)
- [Specs](https://github.com/amethyst/specs)
- [EntityX](https://github.com/alecthomas/entityx)

The general idea behind this bitset design is fairly simple but that does not mean I can explain everything we're doing in 20 words unfortunately. Let's break it down:

# Entities
Entities will be represented as an index into a particular component array slot, combined with a **generation**. Both the entity ID/handle and generation will be stored in a single unsigned 64-bit integer. The ID portion will be stored in the lower bits of the integer, and the generation in the upper bits of the integer. How many bits are consumed by the ID and how many by the generation are up to you, but general limitations of the bitset ECS design will likely lead you to have a rather small ID limit and a very massive generation limit.

A generation is a value that represents a reused entity as a conceptually new object. The generation concept arises naturally from a pretty major problem; pretend we store Entity_A in a component to use for later, maybe as a pathfinding target for another entity. If a random system kills/deletes/removes/recycles Entity_A and it becomes Entity_A++ from a global standpoint, and we attempt to use the stored Entity_A...what happens? Entity_A is dead; it's been replaced with Entity_A++. We need a way to distinguish between the two, and that's the problem the generation solves. 

Believe it or not, this is the only reason you need a generation at least as far as I can tell. If your game design restrictions enabled you to never delete an entity, you could skip this part of the implementation.

For purposes we'll get into later, we'll want a value representing the MAX_ENTITIES of our ECS, and this value should be a power of two that's at least 32 elements long. Nice starting counts are 2048, 4096, and 8192 entities. You'll notice we're not talking about values of 1000000000000 or whatever like a significant amount of other ECS implementations enjoy talking about. Practical memory limitations related to our bitset ECS design won't allow this and you're unlikely to need that many entities, ever. 

Additionally, entities will be supported by a single **bitset**, essentially a very compact list of boolean flags. The bitset will consist of (MAX_ENTITIES / 32) worth of unsigned 32-bit integers, which will be the backing storage for these bools. This bitset will indicate whether an entity is alive or dead, or alternatively active or inactive. Note that this bitset is NOT storing which components that each entity has, although other bitset ECSs sometimes use this approach. We only store the active/inactive state here.

# Components
Components will be one-time-allocated data arrays consisting of MAX_ENTITIES worth of data elements. This will be the case for every single component. Each component will have an accompanying bitset, an array with (MAX_ENTITIES / 32) worth of unsigned 32-bit integers. These bitsets will represent whether or not an entity has a particular component...but it's more appropriate to think of the component as having a particular entity.

There's not really to much to components; they're just arrays of data that you can index into via the ID bits of an entity.

# Tags
Tags are special cased components. They're often used as on/off flags to indicate functionality that an entity may have (Movable, Rotatable, CanShoot, etc.). In our bitset ECS, we'll simply treat a tag as a component with empty storage and only the bitset.

# Systems
Systems can do anything; they're just processing functions, so you might see operations in them like modifying component data via an entity index, creating new entities, deleting entities, interfacing with other functionality of a library like SDL, whatever.

# Queries
Queries are a way to search for the particular entity handles that you are interested in processing at any given time. If we wanted to match all Active Entities with the components Position and Rotatable, and the tag Enemy, but excluding Bosses, a query would give us all entities that match the search parameters.

In our ECS, the entity active/inactive bitset, and the component array bitsets will be used to generate queries. Starting with the entity bitset, we'll perform bitwise and/or/not operations against every component type that we may/may not be interested in. 

A bitwise and of the entity bitset against Position and Rotatable would only collect entities with BOTH of these components.

A bitwise or of the entity bitset against Enemy and Player would collect any entity that has at least one of these components. Note that bitwise or queries require greater care to work with in your systems; in a bitset ECS, you'll need to perform a manual check to determine which one (or both) of these components the entity actually has. You know it has at least one or the other, however.

A bitwise not of the entity bitset against IsTable would exclude all entities that are tables in the entire game. With a naive setup, note that bitwise nots are *order dependent*. I won't be putting any special handling for queries in this implementation, so all exclusion queries should be added AFTER bitwise ands and ors.

You might now understand why we organized our bitsets as belonging to the component arrays rather than to the entities themselves; easy, fast, and cheap querying is the reason why.

Using bitsets to generate these queries is excellent for several reasons, most notably, it's really fast to generate them. Bitwise operations are very fast at a hardware level, and they are likely to be [vectorized](https://en.wikipedia.org/wiki/Automatic_vectorization) without any work on your part. Determining whether 128 (or more!) entities match your search criteria per operation is pretty neat to say the least.

