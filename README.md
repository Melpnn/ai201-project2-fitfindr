# FitFindr

FitFindr is a thrift shopping agent that finds secondhand listings and suggests 
outfits based on your wardrobe.

## How to Run
1. Clone the repo
2. Install dependencies using `pip install -r requirements.txt`
3. Add your Groq API key to a `.env` file: `GROQ_API_KEY=your_key_here`
4. Run `python app.py`
5. Open the URL shown in your terminal

## Tools

### search_listings(description, size, max_price)
- Purpose: This tool searches through the listings provided based on the user's input, which should include things like item description, size, and max price. 
- Inputs: description (str), size (str or None), max_price (float or None)
- Output: A list of matching listing dicts. Each dict contains: id, title, description, category, style_tags, size, condition, price, colors, brand, and platform.
- Failure: Returns an empty list if nothing matches 

### suggest_outfit(new_item, wardrobe)
- **Purpose:** Given a thrifted item and the user's wardrobe, suggests a complete outfit combination using the owned pieces and new item
- **Inputs:** new_item (dict — a listing), wardrobe (dict with an 'items' key)
- **Output:** A string describing which wardrobe pieces to pair with the new item and why.
- **Failure:** If wardrobe is empty, returns general styling advice instead of crashing

### create_fit_card(outfit, new_item)
- **Purpose:** Generates a casual social media caption for the outfit
- **Inputs:** outfit (str from suggest_outfit), new_item (dict — the listing)
- **Output:** A string of 1-3 sentences in casual social media style
- **Failure:** If outfit is empty, returns a fallback caption using only the item details

## Planning Loop

The agent follows this logic for every query:

1. Parse the query to extract a description, size (if mentioned), and max price (if mentioned)
2. Call search_listings — if no results, tell the user and stop immediately
3. Pick the top result and call suggest_outfit with the user's wardrobe
4. Call create_fit_card with the outfit suggestion and selected item
5. Return the session with all results populated

The key decision point is after search_listings, where if it returns an empty list, the agent sets an error message and returns early without calling the LLM tools. This prevents suggest_outfit from receiving empty input.

## State Management

All data is stored in a session dict that gets passed through the planning loop:

```python
session = {
    "query": "",              # original user message
    "parsed": {},             # extracted description, size, max_price
    "search_results": [],     # all matches from search_listings
    "selected_item": {},      # top result, passed int suggest_outfit
    "wardrobe": {},           # loaded at session start
    "outfit_suggestion": "",  # returned by suggest_outfit
    "fit_card": "",           # returned by create_fit_card
    "error": None             # set if something fails, stops the loop
}
```

Each tool reads from the session and writes its output back into it. 
app.py reads the final session and maps each field to an output panel.

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | "No listings matched your search. Try a broader description or a higher max price." Agent stops and does not call suggest_outfit |
| suggest_outfit | Wardrobe is empty |"I don't have your wardrobe on file yet. Try describing what you already own and I can suggest pairings."|
| create_fit_card | Empty outfit string | "Found [title] for $[price] on [platform] — check it out." No LLM call made. |

**Example from testing:** Running `search_listings("designer ballgown", size="XXS", max_price=5)` returned `[]` with no exception. The agent responded with the error message and never called suggest_outfit.

## AI Usage

**Instance 1 — search_listings implementation:**
I gave Claude the Tool 1 spec block from planning.md including the input parameters, 
return value description, and failure mode. I asked it to implement the function using load_listings() from the data loader. The generated code used list comprehensions which I rewrote as regular for loops for readability. I also caught a bug where it was looping over an empty list, which I fixed before running tests.

**Instance 2 — planning loop implementation:**
I gave Claude the Planning Loop section, State Management section, and architecture 
diagram from planning.md. It generated the full run_agent() function. I verified it 
branched correctly on empty search results and stored values in the session dict before using it.

## Spec Reflection

My planning.md spec held up well during implementation. The planning loop logic I 
described matched the final code closely, and I learned a lot about the proper usage of AI and how to properly prompt and use it with intention. 