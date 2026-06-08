# Spec: Tool Functions

**File:** `tools.py`
**Status:** `get_seasonal_conditions` — Pre-implemented, read through. `lookup_plant` — complete spec fields before implementing.

---

## Purpose

These two functions are the tools the agent can call. They retrieve structured data from the local plant database and seasonal data files and return it to the agent loop, which passes it to the LLM as context for generating a response.

---

## Function 1: `lookup_plant()`

### Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `plant_name` | `str` | The plant name as entered by the user or chosen by the LLM — may be any casing, common name, scientific name, or alias |

**Output:** `dict`

When the plant is **found**, return:
```python
{"found": True, "plant": <the full plant dict from _plant_db>}
```

When the plant is **not found**, return:
```python
{"found": False, "name": <normalized input>, "message": <helpful string>}
```

---

### Design Decisions

*Complete the two blank fields below before writing code. The others are pre-filled for you.*

---

#### Input normalization

Strip leading/trailing whitespace and convert to lowercase before any comparison.

```python
normalized = plant_name.strip().lower()
```

---

#### Search order

Search in this order: direct key → display name → aliases. Keys are the fastest
lookup (O(1) dict access), so check those first. Display names are the next most
likely match for clean user input. Aliases are the broadest net, so they go last.

```
1. Direct key match: normalized in _plant_db
2. Display name match: plant["display_name"].lower() == normalized
3. Alias match: normalized in [alias.lower() for alias in plant["aliases"]]
```

---

#### Alias matching approach

*Aliases are stored as a list of strings. How will you check if the normalized input matches any alias in the list? Write your approach in pseudocode or plain English.*

```
So the way im thinking about it is the aliases are just a list of strings for each plant, so to check if what the user typed matches one of them i'd lowercase every alias in the list and then see if my normalized input is in that list, basically something like normalized in [alias.lower() for alias in plant["aliases"]]. The big thing here is that i normalize both sides the exact same way so the casing or any extra spaces dont end up making it miss when it shouldnt, and i want to use exact equality instead of substring matching becuase otherwise something vague like "ivy" would accidentally grab "devil's ivy" and give back the wrong plant. One thing i realized is that this is technically a linear scan since im looping over every plant and every alias, so if the database ever got really big with thousands of plants i'd probably just build one big dictionary at the start that maps every name and alias straight to its slug, that way each lookup is one fast hit instead of looping through everything every single time.
```

---

#### Not-found message

*When a plant isn't found, the agent will read your message and use it to decide what to tell the user. Write the exact string you'll return — make it useful to the agent, not just to a human reading logs.*

```
For the not found message the main thing i kept reminding myself is that the agent is the one actually reading this string, not the user, so it cant just say "plant not found" or the agent is basically gonna dead end the whole conversation. What i want instead is for the message to tell the agent that this plant isnt in our database of 15 houseplants, but also make it clear that its still totally fine to give general care advice from what it already knows, as long as it stays honest with the user that this specific plant isnt in the curated data so the answer is more general. I also want it to nudge the user to try a different name or pick something off the sidebar list incase it was just a naming mismatch. So the actual string i return looks something like: "No exact match for '{normalized}' was found in the plant care database, which covers 15 common houseplants. This plant may go by a different name here, or it may not be in the database. You can still offer general care guidance from your own knowledge, but tell the user this specific plant isn't in the curated database so your advice is general rather than from verified care data. If it might be a naming mismatch, suggest they try the plant's common or scientific name, or pick from the plant list shown in the sidebar."
```

---

#### Implementation Notes

*Fill this in after implementing and running the app.*

**Test: does `"devil's ivy"` return the pothos entry?**
```
Yes it works, when i ran it with "devil's ivy" it came back with found True and gave me the whole Pothos entry, so the alias matching is doing exactly what i wanted since "devil's ivy" isnt the key or the display name, its just sitting in the aliases list under pothos and it still found it.
```

**Test: does `"SNAKE PLANT"` return the snake plant entry?**
```
Yes, typing "SNAKE PLANT" in all caps still returned the snake plant entry no problem, which is basically the whole point of lowercasing both sides before i compare anything, the casing doesnt matter at all and it ended up matching on the display name.
```

**One edge case you discovered while implementing:**
```
The edge case i ran into is that the scientific name isnt actually part of the search order, so if someone types something like "Epipremnum aureum" instead of pothos it comes back as not found even though thats literaly the exact same plant. i left it that way since the spec only says to search the key, display name and aliases, but the good thing is the not found message still lets the agent fall back to general advice so the user isnt completely stuck.
```

---

## Function 2: `get_seasonal_conditions()`

### Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `season` | `str \| None` | One of `"spring"`, `"summer"`, `"fall"`, `"winter"`, or `None` to auto-detect |

**Output:** `dict`

The full season dict from `_season_data`, plus one additional field:

| Added field | Type | Value |
|-------------|------|-------|
| `"detected_season"` | `bool` | `True` if auto-detected from the month; `False` if season was passed as an argument |

---

### Design Decisions

*This function is pre-implemented — read through these fields and the code before working on `lookup_plant`.*

---

#### Auto-detection logic

When `season` is `None`, get the current calendar month with `datetime.now().month`
and look it up in the `_MONTH_TO_SEASON` dict, which maps month numbers to season strings.

```python
current_month = datetime.now().month
season_key = _MONTH_TO_SEASON[current_month]
```

---

#### Season validation

If the caller passes an invalid season string (e.g., `"monsoon"`), the function
falls back to auto-detection — same as if `None` were passed. The `VALID_SEASONS`
set acts as the gate:

```python
VALID_SEASONS = {"spring", "summer", "fall", "winter"}
if season and season.lower() in VALID_SEASONS:
    ...  # use provided season
else:
    ...  # auto-detect
```

---

#### Return structure

The full season dict from `_season_data`, plus a `detected_season` boolean. Example for spring:

```python
{
    "season": "spring",
    "watering": "Increase watering frequency as plants break dormancy ...",
    "fertilizing": "Resume feeding with a balanced fertilizer ...",
    "light": "Days are lengthening — move plants closer to windows ...",
    "pests": "Watch for spider mites and aphids as temperatures rise ...",
    "detected_season": True   # True = auto-detected; False = caller specified
}
```

---

#### Implementation Notes

*Fill this in after testing.*

**Test: does calling with `season=None` return the correct season for the current month?**
```
Yes this lines up, i ran it with no argument and since right now its June the month is 6, which maps to summer in the _MONTH_TO_SEASON dict, and thats exactly what came back. so writing it out, current month is June, the expected season is summer, and the returned season was summer with detected_season set to True since it auto detected it insted of me passing one in.
```

**Test: does calling with `season="winter"` return winter data regardless of the current month?**
```
Yes, when i passed in season="winter" it gave me back the full winter data even though its actually summer right now, and detected_season came back as False which makes sense becuase i specified the season myself instead of letting it auto detect off the month.
```
