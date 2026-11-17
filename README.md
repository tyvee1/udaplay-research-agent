# UdaPlay: an AI research agent for video games

Submission for the UdaPlay project of Udacity's *Building Agents* course, by Ty.

UdaPlay answers questions about video games. It looks in a local database of 15 games first,
judges whether what it found is enough, and searches the web only when it is not. Every answer
states its confidence and cites its sources: a game file, or the URL of a web page.

## What is in this repository

| Path | What it is |
| --- | --- |
| `project/starter/Udaplay_01_starter_project.ipynb` | Part 1: loads the game JSON files, embeds them into a persistent ChromaDB collection, and shows semantic search. Executed, with outputs. |
| `project/starter/Udaplay_02_starter_project.ipynb` | Part 2: the three tools, the agent, and its run on 8 questions. Executed, with outputs. |
| `project/starter/chromadb/` | The vector database Part 1 built (collection `udaplay`) and Part 2 loads. |
| `project/starter/lib/` | The course's library, unchanged (the notebooks import from it). |
| `project/starter/games/` | The course's 15 game files, unchanged. |
| `project/starter/requirements.txt` | The course's dependency list, unchanged. |
| `.env.example` | The names of the three variables the notebooks need; no values. |

The first commit imports the course starter unchanged, so `git diff HEAD~1 -- project/starter`
shows exactly what this submission added.

## Where each requirement can be seen

| Requirement | Where to look |
| --- | --- |
| Load, process and format the game JSON files | Part 1, `In [8]` (the text that gets embedded) and `In [9]`: "15 documents for 15 JSON files" |
| Persistent vector database with embeddings | Part 1, `In [5]`–`In [7]`: `chromadb.PersistentClient(path="chromadb")`, collection `udaplay`, OpenAI embeddings |
| Reusable vector store manager | Part 1, `In [7]`: `GameStoreManager` builds, opens, counts and searches the collection |
| Semantic search demonstrated | Part 1, `In [10]`: 4 queries that do not use any title, each with ranked results and distances |
| Retrieval tool | Part 2, `In [6]`: `retrieve_game` |
| Evaluation tool | Part 2, `In [7]`: `evaluate_retrieval`, an LLM judge returning `useful`, `confidence` and an explanation |
| Web search tool | Part 2, `In [8]`: `game_web_search`, using Tavily |
| Internal first, then evaluate, then web only if needed | Every question's output: `[2 retrieve_game]`, then `[3 evaluate_retrieval]` with its verdict and a `decision:` line, then `[4 game_web_search]` only when the verdict is not useful or its confidence is low |
| State machine | Part 2, `In [9]`: built on the course's `StateMachine`; the `[StateMachine] Executing step:` lines show it running |
| Conversation state | Each finished run is stored per session in the course's `ShortTermMemory`; question 1 asks "When was Pokémon Gold and Silver released?" and the output shows it understood as "When was the video games Pokémon Gold and Silver released?" |
| Reasoning, tool usage and final answer for each query | The step lines, the judge's explanation under `[3 evaluate_retrieval]`, and the report block (`Answer`, `Confidence`, `Process`, `Sources`, `JSON`) |
| Citations | `Sources` in every report; web answers cite URLs. The model can only cite the evidence it was given (each piece has a key such as `G009` or `W2`), and the code turns keys back into files and URLs |
| Combining sources | question 4 cites 2 sources: https://www.facebook.com/fb-answers/mortal-kombat-x-playstation-5-release, https://www.facebook.com/fb-answers/mortal-kombat-x-playstation-5-release-status |
| Confidence levels | `Confidence` in every report, and the judge's own confidence under `[3 evaluate_retrieval]` |

## The run in the notebook

Chat model: `gpt-4o-mini`. Of the 8 questions in the main session, 6 were answered
from the local database and 2 by web search.

| # | Question | Answered from | Confidence | Sources |
| --- | --- | --- | --- | --- |
| 1 | When was Pokémon Gold and Silver released? *(understood as: When was the video games Pokémon Gold and Silver released?)* | local database | high | `games/006.json` |
| 2 | What platform was it released on? *(understood as: What platform was Pokémon Gold and Silver released on?)* | local database | high | `games/006.json` |
| 3 | Which one was the first 3D platformer Mario game? *(understood as: Which video game was the first 3D platformer in the Mario series?)* | local database | high | `games/009.json` |
| 4 | Was Mortal Kombat X released for PlayStation 5? *(understood as: Was the video game Mortal Kombat X released for the PlayStation 5?)* | web search | high | https://www.facebook.com/fb-answers/mortal-kombat-x-playstation-5-release, https://www.facebook.com/fb-answers/mortal-kombat-x-playstation-5-release-status |
| 5 | Who published Grand Theft Auto: San Andreas? *(understood as: Who published the video game Grand Theft Auto: San Andreas?)* | local database | high | `games/002.json` |
| 6 | What is Rockstar Games working on right now? *(understood as: What is Rockstar Games currently working on?)* | web search | high | https://www.bbc.com/news/articles/c9q20djz4wno, https://www.youtube.com/watch?v=wXGNFMErHgs, https://www.youtube.com/watch?v=hNmOP6_njhw |
| 7 | What genre is Halo Infinite, and what is it about? *(understood as: What genre is the video game Halo Infinite, and what is the storyline or premise of Halo Infinite?)* | local database | high | `games/015.json` |
| 8 | Which came out first, Super Mario 64 or Gran Turismo, and on which platforms? *(understood as: Which video game came out first, Super Mario 64 or Gran Turismo, and on which platforms were these video games released?)* | local database | high | `games/009.json`, `games/001.json` |

Answers and web results vary from run to run; the process does not.

## Stand-out features

* **Long-term memory** (the Stand Out section of Part 2): web findings are kept in a second
  collection, `udaplay_web_memory`, on disk in `chromadb/`. In the notebook, the agent
  saved what it learned about Mortal Kombat X, and a new
  agent in a new session answered the same question from that memory without searching the web.
  Only the agent's answer and the URLs it cited are kept, never raw page text, and entries
  expire after 30 days.
* **Structured output**: every answer is also printed as JSON.
* **Tools as pre-defined nodes of the state machine**: the agent's order is fixed in code (see below).

## Design decisions

* **The order is enforced by the state machine, not left to the model.** A model choosing its own
  tool calls can skip retrieval or evaluation on any given run; here every question goes through
  retrieve → evaluate → (web) → answer, and the fallback decision is made in code from the judge's
  verdict.
* **Low confidence counts as not found.** The project asks for the web "if the information is not
  found or confidence is low", so a useful-but-low-confidence verdict also goes to the web.
* **The judge fails closed.** If the evaluation cannot be made (a refusal, an invalid reply, an
  API error), the local results count as insufficient and the agent searches the web.
* **Citations are checked, not trusted.** A cited game file must be one that was retrieved, and a
  cited URL must be one the search returned; anything else is removed and reported under
  `Sources` with a `!`.
* **Follow-up questions are rewritten using the session**, so "What platform was it released on?"
  is retrieved as a complete question.
* **Part 1 rebuilds the collection on each run**, so re-running it never re-embeds duplicates.
* **The database ships** (`chromadb/` is not git-ignored here) so Part 2's outputs can be checked
  against the exact data it used.

## Run it yourself

Python 3.11 or later, from the repository root:

```
cd project/starter
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt notebook
cp ../../.env.example .env      # then put your keys in .env
jupyter notebook                # run Part 1, then Part 2
```

To check the shipped database without any key or install (from the repository root):

```
python3 -c 'import sqlite3; c = sqlite3.connect("project/starter/chromadb/chroma.sqlite3"); print(c.execute("SELECT COUNT(*) FROM embeddings e JOIN segments s ON e.segment_id = s.id JOIN collections c ON s.collection = c.id WHERE c.name = ?", ("udaplay",)).fetchone()[0])'
```

It prints `15`.
