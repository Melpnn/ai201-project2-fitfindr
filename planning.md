# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
This tool searches through the listings provided based on the user's input, which should include things like item description, size, and max price. 

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str): ... the type of clothing the user wants
- `size` (str): ... the size of clothing, ranging from Small to Large, and can be None if not specified
- `max_price` (float): ... the maximum price the user is willing to spend

**What it returns:**
<!-- Describe the return value — what fields does a result contain? -->
A list of matching listing dicts. Each dict contains: id, title, description, category, style_tags, size, condition, price, colors, brand, and platform.
Returns an empty list if nothing matches.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if no listings match? -->
The agent tells the user "No listings matched your search. Try a broader description or a higher max price." The agent then stops and will not call suggest_outfit to avoid crashing.

---

### Tool 2: suggest_outfit

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Given a thrifted item and the user's wardrobe, suggests a complete outfit combination using the owned pieces and new item

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): ... the item the user is considering buying
- `wardrobe` (dict): ... the user's current wardrobe consisting of item dicts, each with id, name, category, colors, style_tags, notes

**What it returns:**
<!-- Describe the return value -->
A string describing which wardrobe pieces to pair with the new item and why.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->
If the wardrobe items list is empty, the agent responds: "I don't have your wardrobe on file yet. Try describing what you already own and I can suggest pairings."
---

### Tool 3: create_fit_card

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Generates a short, shareable caption for a complete outfit

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `outfit` (str): ... The outfit suggestion returned by suggest_outfit
- `new_item` (dict): ... The listing for the thrifted item

**What it returns:**
<!-- Describe the return value -->
A string of 1-3 sentences in casual social media style

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the outfit data is incomplete? -->
If outfit is empty or new_item is missing fields, the agent says "Found [title] for $[price] on [platform] — check it out." Agent uses only the
available new_item fields and skips outfit details entirely.
---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->

1. User sends a message. Agent extracts description, size, and max_price from it.

2. Call search_listings(description, size, max_price).
   - If results is empty: tell the user "No listings matched your search. 
     Try a broader description or a higher max price." Stop here. Do not 
     call suggest_outfit.
   - If results is not empty: set selected_item = results[0]. Continue.

3. Call suggest_outfit(new_item=selected_item, wardrobe=session["wardrobe"]).
   - If outfit is empty: tell the user "I don't have your wardrobe on file 
     yet. Try describing what you already own and I can suggest pairings." 
     Stop here. Do not call create_fit_card.
   - If outfit is not empty: store outfit string. Continue.

4. Call create_fit_card(outfit=outfit, new_item=selected_item).
   - Return the fit card caption to the user.
---

## State Management

**How does information from one tool get passed to the next?**
<!-- Describe how your agent stores and accesses state within a session. What data is tracked? How is it passed between tool calls? -->

session = {
    "query": "",           # the user's original message
    "wardrobe": {},        # loaded at session start via get_example_wardrobe()
    "search_results": [],  # filled by search_listings
    "selected_item": {},   # set to results[0] after search_listings runs
    "outfit": "",          # filled by suggest_outfit
    "fit_card": "",        # filled by create_fit_card
    "error": None          # set if any tool fails, stops the loop
}
---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | "No listings matched your search. Try a broader description or a higher max price." Agent stops and does not call suggest_outfit |
| suggest_outfit | Wardrobe is empty |"I don't have your wardrobe on file yet. Try describing what you already own and I can suggest pairings."|
| create_fit_card | Outfit input is missing or incomplete |generates a fallback caption using only the available new_item details and doesn't crash|

---

## Architecture

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     Use ASCII art or a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html).
     Do NOT embed an image — graders need to read your diagram directly in the file;
     an embedded image or screenshot cannot be evaluated.
     You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->

User query
    │
    ▼
Planning Loop ────────────────────────────────────────────┐
    │                                                     │ 
    ├─► search_listings(description, size, max_price)     │
    │       │ results == []                               │
    │       ├──► [ERROR] "No listings found..." ─────────►│
    │       │                                             │
    │       │ results == [item, ...]                      │
    │       ▼                                             │
    │   session["selected_item"] = results[0]             │
    │       │                                             │
    ├─► suggest_outfit(selected_item, wardrobe)           │
    │       │ outfit == "" or None                        │
    │       ├──► [ERROR] "No wardrobe on file..." ───────►│
    │       │                                             │
    │       │ outfit == "Pair this with..."               │
    │       ▼                                             │
    │   session["outfit"] = outfit                        │
    │       │                                             │
    └─► create_fit_card(outfit, selected_item)            │
            │                                             │
        session["fit_card"] = "..."                       │
            │                                         ◄───┘ error paths terminate here
            ▼
        Return fit_card to user
---

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:**
For search_listings, I'll give Claude the Tool 1 spec block from planning.md
(inputs, return value, failure mode) and ask it to implement the function using
load_listings() from the data loader. I'll verify the generated code filters by
all three parameters (description, size, max_price) and handles the empty results
case before using it. Then I'll test it with 3 queries: one that matches, one
with no results, and one with no size specified.

For suggest_outfit, I'll give Claude the Tool 2 spec block and the
wardrobe_schema.json structure and ask it to implement the function. I'll verify
it uses the wardrobe items list and handles the empty wardrobe case. I'll test it
with get_example_wardrobe() and get_empty_wardrobe() to confirm both options work.

For create_fit_card, I'll give Claude the Tool 3 spec block and ask it to
implement the function. I'll verify it produces a casual 1–3 sentence caption and
that it generates a fallback when outfit is empty or new_item is missing fields.


**Milestone 4 — Planning loop and state management:**
I'll give Claude the Planning Loop section, the State Management section, and the
Architecture diagram from planning.md and ask it to implement the full planning
loop. I'll verify it follows the exact conditional logic (empty results stops the
loop, empty outfit stops before create_fit_card), correctly writes to each session
key, and returns early on errors. I'll trace through the complete interaction
example manually to confirm the output matches what's described in planning.md.
---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
<!-- What does the agent do first? Which tool is called? With what input? -->
Tool called: search_listings
Input: description="vintage graphic tee", max_price=30.0, size=None 
The agent calls this first because we need to find the item before we can style using it.
Output: returns lst_002 — "Y2K Baby Tee — Butterfly Print", $18, on depop, size S/M

**Step 2:**
<!-- What happens next? What was returned from step 1? What tool is called now? -->
Tool called: suggest_outfit
Input: new_item=Y2K Baby Tee, wardrobe=example_wardrobe
The search returned a result, so we pair it with what the user owns.
The wardrobe has some pieces it finds that would match, like w_001 (baggy dark wash jeans)
Output: "Pair the Y2K butterfly tee with your dark wash baggy jeans"


**Step 3:**
<!-- Continue until the full interaction is complete -->
Tool called: create_fit_card
Input: outfit=output from suggest_outfit , new_item=Y2K Baby Tee
Output: "found this y2k butterfly tee on depop for $18, and it matches my baggy jeans"

**Final output to user:**
<!-- What does the user actually see at the end? -->
The user sees the found listing (Y2K Baby Tee, $18, depop), the outfit suggestion (pair with baggy jeans), and the fit card caption.
