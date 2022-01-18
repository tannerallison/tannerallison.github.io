---
layout: post
title: A Task Delayed
category: screeps
---

I can sometimes be slow on the uptake for some things, especially when it comes to programming. I recently had a moment of clarity in designing my Screeps AI that may seem like a "well duh" kind of thing to most who play the game, but I just didn't grasp the concept, even though I've been playing off and on for years, and that is the idea of setting up actions for the screeps to perform 'aynchronous' to execution of the action itself.

The Screeps tick cycle seems so quick that it is hard to wrap my mind around all the stuff that is happening every second. When `main.loop` is called every ~1 second, your entire codebase is run, from scratch. If you instantiate a class that collects all the creeps of a certain role in the main.loop, it gets instantiated from scratch, every second. My mind just rejects that. How inefficient! It can't possibly be doing that every time?!

Because it was so unfathomable to me, I've operated up til now with a 'synchronous' type approach to my screeps codebase. It was easier to think that what was happening was that the game would call upon each creep to decide what it would do and that creep would then do it. My handlers function as decision trees where the world state would determine what the creep would do and then the action is taken, right then and there.

But that's not actually what is happening. Each game tick, the game calls your `main.loop` function and it's up to you to decide what to do with it. Now, you can, as I have done, include a loop within your main.loop function that cycles through the creeps and executes an action on each one in turn. But it's also possible to have higher order processes make the decisions and assign tasks to the creeps, then follow that up with a simple runner that just executes the assigned action for each creep. 

This followup cycle is what really made it click for me. If I can assign a task, rather than execute it immediately, I can have other high-level processes potentially hijack that task if something more important is found. Each creep or decision maker doesn't have to be aware of everything happening around it, it just have to decide the best decision for what it is responsible for.
