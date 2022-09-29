# **A Bitset Entity-Component-System in C99**

# What is an ECS
If you're reading this, you probably know what an Entity-Component-System (ECS) is already. If you need a really giant overview of this concept, check out this [ECS FAQ](https://github.com/SanderMertens/ecs-faq). However, here's a quick refresher.

Implementations differ, but the modern, generalized concept is a bunch of arrays storing several instances of the same data, say, 500 floats in a row. An **entity**, usually just an integer of some sort, indexes into these data arrays, otherwise known as **components**, in some fashion. There may or may not be a concept of a **tag**, which is almost the same as a component except without any associated data (a tag's presence essentially acts as a boolean flag). **Systems** are processing functions, usually accompanied by some sort of **query** that selects the entities owning certain components that you are actually interested in operating on within a particular system. There may be a **world**, which acts as a container that holds component arrays and entity data. A world is technically optional, but this concept arises due to issues pertaining to entity deletion.

# Why Make Another One?
There are plenty of existing ECS solutions around the web (probably one of the best being [Flecs](https://github.com/SanderMertens/flecs)) Many of them are incredible feats of engineering! Many of them are also very complicated, and lean very far from the *actual* goal of what your average casual game developer likely wants: a conceptually simple and decently performant way to filter for the particular game objects they would like to work with at any given time. This really should not take a multi-thousand line codebase to achieve, and we should be able to keep our implementation in our heads. We shouldn't have to worry about deferred updates or multithreading or relationship graphs or whatever. 

I want to create new objects, modify the things they do and don't have, modify the values that those things have, and delete them sometimes. I want this to be easy and dynamic. 

Let's keep it simple, and design something we won't be afraid to use.

# The Approach
There are many varieties or ECS designs, and you can research some of these keywords: Archetype, Sparse Set, Bitset, Reactive. We're going to focus on "Bitset," as it's easy to implement and I've never seen any real writeups behind this design anywhere. However, I know of three ECS implementations using this approach in some form:

- [ECX](https://github.com/eliasku/ecx)
- [Specs](https://github.com/amethyst/specs)
- [EntityX](https://github.com/alecthomas/entityx)

The general idea behind this bitset design is fairly simple but that does not mean I can explain everything we're doing in 20 words unfortunately. Let's break it down:

# Entities
Entities will be represented as an index into a particular component array slot (or a component's bitset, which we'll get to), combined with a **generation**. Both the entity ID/handle and generation will be stored in a single unsigned 64-bit integer. The ID portion will be stored in the lower bits of the integer, and the generation in the upper bits of the integer. How many bits are consumed by the ID and how many by the generation are up to you, but general limitations of the bitset ECS design will likely lead you to have a rather small ID limit and a very massive generation limit.

A generation is a value that represents a reused entity as a conceptually new object. The generation concept arises naturally from a pretty major problem; pretend we store Entity_A in a component to use for later, maybe as a pathfinding target for another entity. If a random system kills/deletes/removes/recycles Entity_A and it becomes Entity_A++ from a global standpoint, and we attempt to use the stored Entity_A...what happens? Entity_A is dead; it's been replaced with Entity_A++. We need a way to distinguish between the two, and that's the problem the generation solves. 

Believe it or not, this is the only reason you need a generation at least as far as I can tell. If your game design restrictions enabled you to never delete an entity or at least never store them past immediate usage, you could skip this part of the implementation.

Another problem caused by the generation; we need a permanent place to store the current entity generation for each entity. The easiest way to do this is simply store all entity IDs in a sort of lookup table, and increment the generation of the particular entity in the lookup table when an entity is recycled.

For purposes we'll get into later, we'll want a value representing the MAX_ENTITIES of our ECS, and this value should be a power of two that's at least 32 elements long. Nice starting counts are 2048, 4096, and 8192 entities. You'll notice we're not talking about values of 1000000000000 or whatever like a significant amount of other ECS implementations enjoy talking about. Practical memory limitations related to our bitset ECS design won't allow this and you're unlikely to need that many entities, ever. 

Additionally, entities will be supported by a single **bitset**, essentially a very compact list of boolean flags. The bitset will consist of (MAX_ENTITIES / 32) worth of unsigned 32-bit integers, which will be the backing storage for these bools. This bitset will indicate whether an entity is alive or dead, or alternatively active or inactive. Note that this bitset is NOT storing which components that each entity has, although other bitset ECSs sometimes use this approach. We only store the active/inactive state here.

We're still not done here. It's convenient to store a **freelist** of entities, essentially tracking which IDs are available to hand out when a new entity is requested. You could simply scan through the active/inactive bitset to determine an entity to hand out, but this could potentially get slow as you exhaust more and more entities. I've opted to use the freelist approach advocated for by other ECS articles.

# Components
Components will be one-time-allocated data arrays consisting of MAX_ENTITIES worth of data elements. This will be the case for every single component (not tags though). Each component will have an accompanying bitset, an array with (MAX_ENTITIES / 32) worth of unsigned 32-bit integers. These bitsets will represent whether or not an entity has a particular component...but it's more appropriate to think of the component as having a particular entity.

Someone will point out that this wastes a lot of memory, and that the data we will be iterating over in the future will not be tightly packed together. You are correct. However, our focus is not maximal performance or minimum memory usage; it's ease of implementation while gaining the benefits of the ECS design pattern. Additionally, I've run some math on the memory usage before. Even in very pessimistic cases (but like, actually reality based, so again no 10000000000000 element boids simulation), you're using maybe 150 megabytes of RAM total for the entire ECS state, with a lot of it unused. That's like half of a Firefox tab, so I'm not really going to cry about it. If you absolutely need to minimize memory usage, or you need really high entity counts with an absolutely metric ton of unique non-tag data-carrying components, this is not the implementation for you.

There's not really to much to components; they're just arrays of data that you can index into via the ID bits of an entity. The main catch here comes from the entity side; we have to make sure that entities that have been stored perform a check against the global entity generation first, to make sure that entity is permitted to access the component storage.

# Tags
Tags are special cased components. They're often used as on/off flags to indicate functionality that an entity may have (Movable, Rotatable, CanShoot, etc.). In our bitset ECS, we'll simply treat a tag as a component with empty data storage and only the bitset available to use.

# Systems
Systems can do anything; they're just data-processing functions, so you might see operations in them like modifying component data via an entity index, creating new entities, deleting entities, interfacing with other functionality of a library like SDL, whatever. They're usually going to be paired with a query, but that's not strictly necessary.

# Queries
Queries are a way to search for the particular entity handles that you are interested in processing at any given time. If we wanted to match all Active Entities with the components Position and FireDamageReduction, and the tag Enemy, but excluding Bosses, a query would give us all entities that match these search parameters.

In our ECS, the entity active/inactive bitset combined with the component array bitsets will be used to generate queries. Starting with the entity bitset, we'll perform bitwise and/or/not operations against every component type that we may/may not be interested in. After we've performed all of our query operations, we'll iterate over the resulting bitset which will conveniently contain entities matching our query terms, extracting entity IDs along the way. 

A bitwise and of the entity bitset against Position and Rotatable would only collect active entities with BOTH of these components.

A bitwise or of the entity bitset against Enemy and Player would collect any active entity that has at least one of these components. Note that bitwise or queries require greater care to work with in your systems; in a bitset ECS, you'll need to perform a manual check to determine which one (or both) of these components the entity actually has. You know it has at least one or the other, however.

A bitwise not of the entity bitset against IsTable would exclude all active entities that are tables in the entire game. Note that bitwise nots are *order dependent*, so with a naive implementation and no automated reordering of your query operations you may get results that you aren't expecting. I won't be putting any special handling for query operations in this implementation, so all nots should be added AFTER bitwise ands and ors.

Using bitsets to generate these queries is excellent for several reasons, most notably, it's really fast to generate them. Bitwise operations are very fast at a hardware level, and they are likely to be [vectorized](https://en.wikipedia.org/wiki/Automatic_vectorization) without any work on your part. Determining whether 128 (or more!) entities match your search criteria per operation is pretty neat to say the least.

You might now understand why we organized our bitsets as belonging to the component arrays rather than to the entities themselves; easy, fast, and cheap generation of an iteration list of entities matching our search criteria is the reason why. Even in extreme cases with large amounts of query operations and hundreds of queries, it's unlikely for your total query time to take more than half a millisecond combined. Well, in an optimized release build, anyway. It's not the fastest query solution, but it's still pretty good for the amount of effort we're putting in (read: little).

In our ECS, queries are not cached and are generated on demand. They do not perform "live-updates," so any entity creation or deletion will not be visible until another query is executed.

# World
The world is a data container, consisting of the entity master list thing we were hinting at before along with all component arrays that exist, and their bitsets. The world concept would ideally not be needed. We almost have everything we want already; we keep some master list of entity "references" to keep track of entity generations and available entities consumers can use, we create some component arrays and their bitsets, and then some functions that take pointers to the entity master list and the component arrays we want to work with. We execute a query in the system to generate a list to iterate over so we can safely extract data out of the arrays we've passed into the function.

This works, up until you delete an entity. Deleting an entity requires removing its active state on **every component that exists**. Not doing so is just a straight error, because queries and direct checks against the array will indicate the entity still exists, even though it doesn't.

You might think you could keep track of this manually, but you can't. Systems can dynamically add components or remove them from an entity individually at any time. Do you think you can manually delete an entity from its appropriate component arrays after 100 branching transformation functions? Probably not, so we have to shotgun the entire thing and guarantee that every component disables the correct entity on deletion. We also have to increment the entity's generation here.

It's the kind of combined action that really should be taken care of by a larger container, so that's what the world is there for.

# The Summary So Far, Because That Was a Lot of Words
Entities are an unsigned 64-bit integer, with the 64-bits split between a generation that signifies the "uniqueness" of an entity, and a handle which indexes into component data arrays and component bitsets. We'll have a bitset that indicates the active and inactive state of these entities. There will be a "global" storage for these entities so that we can actually keep track of the generation and increment it. We'll maintain a freelist of entities so that we can hand out unused entities very quickly.

Components are data stores accessed by an entity handle, combined with a bitset indicating whether an entity currently has the component. They always hold MAX_ENTITIES worth of their component type, all pre-allocated. If accessing a component via a stored entity, a check against the stored entity's generation and the global version of the entity's generation must be performed. 

Tags are special-cased components; they have no data aside from the component bitset. They act as boolean flags.

Systems are data-processing functions, typically linked to a query, and often manipulate entity data or entities themselves.

Queries select the entities you are interested in working on at a given time based on your component search criteria. You can require certain components to exist, or not exist, using filtering operations on bitsets. Queries do not update "live," so entity creation or deletion will not be visible until another query is created. We use queries to generate a list of entities to iterate over that match our search criteria.

The world is a container that synchronizes and stores entity and component state. The primary reason for its existence is that entity deletion requires you to know about the existence of all components in order to appropriately remove said entity from all component bitsets. It's otherwise just a convenience around independent access to entity state and component state.
