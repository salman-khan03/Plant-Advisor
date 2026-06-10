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
For each plant in the database, lowercase every alias in its "aliases" list and
test membership of the normalized input against that lowercased list:

    if normalized in [alias.lower() for alias in plant["aliases"]]:
        return {"found": True, "plant": plant}

This is a linear scan over the 15 plants (O(n) plants × O(a) aliases each), which
is trivial at this size. If the database grew to thousands of plants, I would build
a single flat lookup dict ONCE at module load — mapping every normalized name
variant (key, display_name, scientific_name, and each alias) to its plant slug —
so any lookup becomes a single O(1) dict access instead of a scan. For 15 plants
the scan is simpler and fast enough, so I keep it.

I also match scientific_name in the same pass, since the tool definition explicitly
tells the LLM it can pass a scientific name (e.g., "Monstera deliciosa").
```

---

#### Not-found message

*When a plant isn't found, the agent will read your message and use it to decide what to tell the user. Write the exact string you'll return — make it useful to the agent, not just to a human reading logs.*

```
f"'{plant_name}' is not in the plant care database. Do not invent specific care "
f"instructions for it. Acknowledge to the user that this plant isn't in your "
f"database, then offer general guidance based on the plant type or what the user "
f"describes, and suggest they confirm details with a specialized source."

This message does two jobs the LLM can act on directly:
  1. It states the gap plainly so the agent stops trying to look it up.
  2. It instructs the agent on the desired behavior — graceful degradation, not a
     dead-end "I don't know" and not a confidently-wrong fabrication. The message,
     not just the system prompt, carries the instruction because it lands in the
     tool-result context the LLM is reasoning over at the moment it decides what
     to say.
```

---

#### Implementation Notes

*Fill this in after implementing and running the app.*

**Test: does `"devil's ivy"` return the pothos entry?**
```
Yes — {"found": True, "plant": {"display_name": "Pothos", ...}} (matched via alias).
```

**Test: does `"SNAKE PLANT"` return the snake plant entry?**
```
Yes — even with surrounding whitespace ("  SNAKE PLANT ") it returns found: True,
because the input is .strip().lower()'d before matching against the display name.
```

**One edge case you discovered while implementing:**
```
The plant data keys (slugs) use underscores ("snake_plant"), but users type spaces
("snake plant"). A raw user phrase like "snake plant" therefore does NOT hit the
fast O(1) key path — it's caught one step later by the case-insensitive display_name
match ("Snake Plant".lower() == "snake plant"). Worth knowing: the direct-key path
mainly serves slugs the LLM passes, while human phrasing usually resolves via
display_name or aliases.
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
Current month: 6 (June)
Expected season: summer
Returned season: summer  (dict["name"] == "summer", detected_season: True)
```

**Test: does calling with `season="winter"` return winter data regardless of the current month?**
```
Yes — passing season="winter" returns the winter dict with detected_season: False,
independent of the current month.
```
