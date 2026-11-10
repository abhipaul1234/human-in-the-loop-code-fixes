# Mistakes in AI-written code: logic, data and reliability fixes

I built several small projects with AI assistance plus my own work. This repository documents mistakes the AI made in the code, what was wrong with each one, and how it was fixed.

Each entry shows the AI-written pattern, the fix, a status, and a one-line lesson. The snippets are shortened and generalised so the idea is easy to see; names, values and layout differ from the real projects.

Status values:

- **Fixed (untested)**: the fix is written and was checked by reading the code. The projects were not built, run or tested after the change.
- **Partially fixed**: part of the problem is fixed; the entry says what remains.
- **Already fixed in later migrations**: the original code is a database migration that a later migration replaced.

## Projects

| Project | What it is | High | Moderate | Fixes |
|---|---|---:|---:|---|
| Blackout | Multiplayer web board game with a server on an edge runtime | 0 | 1 | [blackout/fixes.md](blackout/fixes.md) |
| Campus Bus Tracking | Flask + Socket.IO live bus tracker with a map | 2 | 2 | [campus-bus/fixes.md](campus-bus/fixes.md) |
| Lumina Academic | Flask study tracker with AI-written insights | 2 | 2 | [lumina/fixes.md](lumina/fixes.md) |
| Malware Analysis Dashboard | Next.js dashboard plus Python scanning scripts | 2 | 4 | [malware/fixes.md](malware/fixes.md) |
| multitype | Typing-battle web app on a hosted Postgres backend | 1 | 3 | [multitype/fixes.md](multitype/fixes.md) |
| quizzy2.0 | Browser quiz app that parses AI-written quizzes | 3 | 3 | [quizzy/fixes.md](quizzy/fixes.md) |
| Source-label confounding code | Research code for a phishing-detection paper | 3 | 4 | [source-label/fixes.md](source-label/fixes.md) |
| UniGrid | Next.js + PostgreSQL university management app | 0 | 1 | [unigrid/fixes.md](unigrid/fixes.md) |
| **Total** | | **13** | **20** | |

## How each `fixes.md` is organised

Entries are grouped into **Logic & data**, **UI / UX** and **Reliability & quality**. Each entry has: title, why it is a problem, the AI-written code, the fixed code, status, and a lesson.
