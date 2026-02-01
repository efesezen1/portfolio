---
title: "CoSudoku"
description: "A multiplayer Sudoku game"
pubDate: "Feb 08 2026"
heroImage: "/blog-placeholder-cosudoku.png"
---

# Developing Cosudoku: My Journey and Tech Stack

In this post, I want to talk about the development process of **Cosudoku**, the tools I used, and some specific technical details I want to document for my future self.

## Why Cosudoku?

Sudoku entered my life about five months ago. Initially, I was playing on a popular mobile app. However, I quickly became frustrated with the monetization model: hints cost money, and if you dared to hit a winning streak of 4-5 games, you were hit with unavoidable ads. Removing ads required a subscription, and as a casual player, the constant interruptions were a dealbreaker.

![Screenshot of a standard Sudoku game or the distracting UI of mobile apps](/blog/sudoku/sudoku-ads.gif)

Despite the annoying ads, I didn't want to stop playing. I found myself encouraging my friends to play too. We started trying to solve the same puzzles simultaneously while hanging out, which turned into a fun, informal competition.

That was the "Aha!" moment: **Why not turn this into a dedicated competitive platform?**

## The Vision

I wanted to create a Sudoku experience that felt more social and competitive. To achieve this, I needed a few core components:

1. **Puzzle Generator:** A robust way to create boards of varying difficulty.
2. **Single Source of Truth (DB):** A database to keep track of game states.
3. **Matchmaking/Queue:** A system to match players based on difficulty levels.
4. **Authorization:** A simple yet secure way to handle user identities.
5. **Real-time Synchronization:** This was the most critical part. The lobby and the game board needed to stay perfectly in sync across multiple clients.

![UI Screenshot of the Cosudoku Lobby or Matchmaking screen](/blog/sudoku/game-lobby.png)

## Choosing the Tech Stack: The Move to Convex

In a typical position, one might look at REST for control and data flow. However, because real-time state is paramount for a competitive game, unidirectional data flow (like standard REST) isn't sufficient. You need **Websockets**.

Initially, I considered using Firebase/Supabase for real-time needs. But after seeing the "emerging tech" discussions by developers like _Theo (T3.gg)_ on YouTube, I decided to dive into **Convex**.

### Why Convex?

Let me explain why I fell in love with it. Convex is basically a "backend-as-a-service" that treats your database as code.

Traditionally, if you want to change your DB schema, you go to a dashboard, write SQL, or run migrations. In Convex, there is no need to leave your editor. **Everything is abstracted as code.**

### Defining the Schema

When you initialize a project, your entire database structure is defined in a `schema.ts` file.

Here is a snippet of how I defined the user structure. Notice how readable and type-safe the validation is:

```js
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  users: defineTable({
    externalId: v.string(), // ID from the auth provider
    name: v.string(),
    displayTag: v.string(),
    status: v.union(v.literal("online"), v.literal("offline")),
    image_url: v.string(),
    has_custom_board: v.boolean(),
  }).index("by_externalId", ["externalId"]),
});
```

![Image: Screenshot of the schema.ts file in the editor](/blog/sudoku/typesafe-table.png)

### Development Velocity

This approach changes everything. It means faster development cycles and no "context switching" between your frontend logic and your database dashboard.

1. **Type Safety:** Everything from the server to the client is automatically typed.
2. **Simplified Logic:** You don't need to worry about complex state management tools like Redux. Convex handles the reactivity.
3. **Unified Routing:** Instead of wrestling with API routes, you simply write queries and mutations that feel like standard TypeScript functions.

For example, starting a solo game is as simple as calling a mutation from a React component:

```js
const createSoloGame = useMutation(api.games.createSoloGameForCurrentUser);

// Inside your component:
const handleStartGame = async () => {
  const gameId = await createSoloGame({ difficulty: selectedDifficulty });
  // Navigate to game...
};
```

Check how we call function based on file routing:

![Image: Screenshot of the useMutation hook and game creation logic](/public/blog/sudoku/route-based-typing.png)
