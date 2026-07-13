# Telegram Chat Screenshots with the FitClaw AI Agent hosted on a AWS EC2 instance

## Get reminders to track your Meals (cron jobs that you set up)

<img width="1041" height="802" alt="image" src="https://github.com/user-attachments/assets/8e338f1c-389e-4fc2-99b9-4975ae2edaea" />

## Log a Meal (pulls calories from shared nutrition tables)

<img width="1180" height="900" alt="Screenshot From 2026-07-14 04-32-02" src="https://github.com/user-attachments/assets/3d9d50b7-392e-4fe6-9f74-f91117fb9691" />

<img width="1071" height="699" alt="image" src="https://github.com/user-attachments/assets/329da696-6027-465a-b54b-06f20189e313" />


## Log your workouts (just use tts and describe your workouts)

<img width="1155" height="893" alt="Screenshot From 2026-07-14 04-32-46" src="https://github.com/user-attachments/assets/a5aee7f1-bf13-4ac1-8094-d05a420befbe" />

<img width="1155" height="893" alt="Screenshot From 2026-07-14 04-32-58" src="https://github.com/user-attachments/assets/e667b741-1db9-4092-8464-343539cf5bc7" />

## Get your Protien summary for the day 
<img width="1082" height="900" alt="Screenshot From 2026-07-14 04-49-57" src="https://github.com/user-attachments/assets/27d73c83-3aa2-468a-8b5f-bddf81544b8d" />

<img width="1072" height="761" alt="Screenshot From 2026-07-14 04-50-04" src="https://github.com/user-attachments/assets/4c90b729-ad43-4324-bd09-4cdadb2f01a1" />

## Get final summary/report for the day (saved for future reference)

<img width="1070" height="899" alt="Screenshot From 2026-07-14 04-52-06" src="https://github.com/user-attachments/assets/432cf650-6b67-42b6-a5c0-ff1b29af1985" />

<img width="1070" height="899" alt="Screenshot From 2026-07-14 04-52-10" src="https://github.com/user-attachments/assets/deac2699-edc9-4b53-b53b-7f2b7ae4fe27" />

## Coming Next: Ordering via Swiggy MCP
Currently FitClaw logs meals and workouts and reports on nutrition gaps (like the protein summary above), 
but doesn't act on them yet. The next phase wires in Swiggy MCP so the agent can close the loop — 
going from "you're short on protein today" to actually placing an order.

### Planned Flow
1. **Detect the gap** — after logging a meal, FitClaw compares intake against the day's macro targets 
   (same nutrition tables used for the protein summary above).
2. **Search Swiggy Food MCP** — if there's a shortfall, the agent calls `search_restaurants` / `search_menu` 
   to find a dish that fills the gap within the user's budget.
3. **Confirm in chat** — FitClaw sends 2–3 options back to Telegram/Whatsapp with calories/protein/price; user replies 
   `"order"` or picks one.
4. **Place & track** — agent calls `update_food_cart` → `apply_food_coupon` (if available) → `place_food_order`, 
   then sends delivery ETA via `track_food_order`.
5. **Log automatically** — once delivered, the order is logged as a meal entry, updating the day's summary 
   without the user re-entering anything.

### Also planned
- **Instamart weekly grocery autopilot** — Sunday grocery list generated from the week's nutrition gaps, 
  approved with one tap, ordered via `search_products` → `update_cart` → `checkout`.
- **Dineout booking** — when eating out fits better than delivery, FitClaw suggests restaurants matching 
  the day's macro budget and books a table via `search_restaurants_dineout` → `get_available_slots` → `book_table`.

*Status: Swiggy MCP integration is in progress — meal/workout logging and reporting (shown above) are live; 
ordering and grocery autopilot are the next milestone.*

