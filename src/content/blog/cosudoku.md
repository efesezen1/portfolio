---
title: "CoSudoku"
description: "A multiplayer Sudoku game"
pubDate: "Feb 08 2026"
heroImage: "/blog-placeholder-cosudoku.png"
---

# How I Built **cosudoku** – A Full-Stack Competitive Sudoku App

I just shipped **cosudoku**, a full-stack Sudoku app designed for both solo zen sessions and high-stakes competitive play. It features global matchmaking, private friend invites, and real-time move synchronization.

The goal was to build something that feels snappy and reactive, whether you're solving a puzzle alone or racing against an opponent. Here is a look behind the scenes at how I built it.

---

## Architecture and Tech Stack

Building a real-time game requires a stack that minimizes latency. I chose **React Native** for the frontend to ensure a native feel on mobile, and **Convex** for the backend because its reactive nature is perfect for games.

The frontend calls Convex server functions, which manage the database and sync state across clients instantly. Here’s the core stack:

- **React Native / Expo:** For a cross-platform mobile experience.
- **Convex:** The backend-as-a-service handling the DB, real-time sync, and scheduling.
- **Clerk:** Identity management and secure authentication.
- **Sentry:** Error tracking and session replay to catch production bugs.
- **sudoku-gen:** A lightweight library to generate valid Sudoku puzzles on the fly.
- **i18next:** Localization support for 6 different languages.

---

## The Core Logic: Multiplayer Game Flow

The heart of **cosudoku** is the real-time interaction between two players. Unlike a turn-based game, Sudoku is a race. Every valid move updates your board and score, which must be reflected on your opponent's screen immediately so they know exactly how far ahead (or behind) they are.

Here is how the data flows from the moment two players look for a match to the final victory:

```text
  Player 1                  Convex Backend                  Player 2
  --------                  --------------                  --------
      |                          |                             |
      |-- findMatch() -------->>|                             |
      |                          |-- add to queue              |
      |                          |                             |
      |                          |<<-------- findMatch() ------|
      |                          |-- match found!              |
      |                          |-- createMultiPlayerGame()   |
      |                          |-- schedule startGame()      |
      |                          |                             |
      |<<-- game created -------|------->> game created ------>|
      |                          |                             |
      |   (3-second countdown)   |     (3-second countdown)    |
      |                          |                             |
      |-- move() ------------->>|                             |
      |                          |-- validate & score          |
      |<<-- updated board ------|------->> updated board ----->|
      |                          |                             |
      |                          |<<-------------- move() -----|
      |                          |-- validate & score          |
      |<<-- updated board ------|------->> updated board ----->|
      |                          |                             |
      |                          |  (repeat until solved)      |
      |                          |                             |
      |<<-- game complete ------|------->> game complete ----->|
      |                          |-- archive to history        |

```

---

## Social Play: Friend Invite Flow

Playing against strangers is fun, but challenging a friend is better. I built a dedicated flow for invites that includes a search feature and a "Waiting Room." To prevent the database from filling up with "ghost" invites, I used Convex’s **Scheduler** to automatically expire requests after a few minutes.

```text
  Sender                    Convex Backend                  Receiver
  ------                    --------------                  --------
      |                          |                             |
      |-- searchUsers() ------>>|                             |
      |<<-- results ------------|                             |
      |                          |                             |
      |-- sendInvite() ------->>|                             |
      |                          |-- insert game_invites       |
      |                          |-- schedule removeInvite()   |
      |                          |   (auto-expire timer)       |
      |                          |                             |
      |  (polls getInviteStatus) |  (sees in getPendingInvites)|
      |                          |                             |
      |                          |<<------- acceptInvite() ----|
      |                          |-- create multiplayer game   |
      |                          |-- schedule startGame()      |
      |                          |                             |
      |<<-- invite accepted ----|------->> redirect to game -->|

```

---

## Defining the Game Schema

Using Convex’s `defineSchema`, I ensured the data was type-safe from the start. This is crucial when you're passing board states back and forth.

| Table                   | Purpose                                              |
| ----------------------- | ---------------------------------------------------- |
| **`users`**             | Profiles linked via Clerk's `externalId`.            |
| **`solo_games`**        | Active and archived solo game states.                |
| **`multiplayer_games`** | Real-time state for 2-player competitive matches.    |
| **`matchmaking_queue`** | A waiting room for players seeking random opponents. |
| **`game_invites`**      | Tracks friend-to-friend game requests.               |

---

## Conclusion

Building **cosudoku** was a masterclass in leveraging managed services. By combining React Native’s flexibility with Convex’s real-time power, I was able to build a competitive game environment that scales without managing complex WebSockets or custom server infrastructure.
