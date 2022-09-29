# **A Bitset Entity-Component-System in C99**

# What
If you're reading this, you probably know what an Entity-Component-System (ECS) is already. If you need a really giant overview of this concept, check out this [ECS FAQ](https://github.com/SanderMertens/ecs-faq) However, here's a quick refresher.

Implementations differ, but the modern, generalized concept is a bunch of arrays storing contiguous or at least sequential data of the same type, say, 500 floats in a row. An **entity**, usually just an integer of some sort, indexes into these data arrays, otherwise known as **components**, in some fashion. **Systems** are processing functions, usually accompanied by some sort of **query** that selects the entities owning certain components that you are actually interested in operating on within a particular system.

# Why
There are plenty of existing ECS solutions around the web (probably one of the best being [Flecs](https://github.com/SanderMertens/flecs)) Many of them are incredible feats of engineering! Many of them are also very complicated, and lean very far from the *actual* goal of what your average game developer likely wants: a conceptually simple and decently performant way to filter for the particular game objects they would like to work with at any given time. This really should not take a multi-thousand line codebase to achieve, and we're going to prove that in this article.

# The Approach
There are many varieties or ECS designs, and you can research some of these keywords: Archetype, Sparse Set, Bitset, Reactive. We're going to focus on "Bitset," as it's easy to implement and I've never seen any real writeups behind this design anywhere. However, I know of three ECS implementations using this approach in some form:

-[ECX](https://github.com/eliasku/ecx)
-[Specs](https://github.com/amethyst/specs)
-[EntityX](https://github.com/alecthomas/entityx)