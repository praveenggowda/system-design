# Item Reordering — Problem Statement

Design the item reordering feature on a monday.com board.

A user has a board with 1,000 items. They drag Item C and drop it between Item A and Item B.

Walk through everything that happens: user interface, API, backend, database, and real-time updates to other connected users.

## Constraints

- Average: 5,000 reorder events per second globally
- Peak: 20,000 reorder events per second
- A single board can have up to 100,000 items
- Up to 50 concurrent users viewing and editing the same board
- Two users can reorder the same item simultaneously — one must win
- Real-time update must reach all connected users within milliseconds
- The item order must never be corrupted

## Core Problems

1. How to represent item order efficiently (avoid renumbering entire table)
2. How to handle concurrent reorders on the same item
3. How to push the update to all connected users in real time
