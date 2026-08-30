# Reference — example recipe captions

Real TikTok captions used to validate the locked `Recipe` field list (see
`docs/plans/PLAN_recipe_app_foundation.md`). They double as **parser test fixtures**
(T3.1) and candidate **seed recipes** (T1.4). Two are German, one English, chosen to
span the range: a video-only caption with pre-computed macros, a grouped multi-section
recipe, and an English one with real steps + a storage note.

---

## Example 1 — Cremige Chicken Parmesan Pasta (German; video-only, pre-computed macros)

Raw caption:

```
Cremige Chicken Parmesan Pasta (Meal Prep)

💪 Pro Portion
594kcal · 57g Protein · 64g Carbs · 11g Fett

🛒 Zutaten (4 Portionen)
• 600g Hähnchenbrustfilet
• 340g Farfalle
• 140ml Kochsahne 7%
• 70g Parmesan
• 150g körniger Frischkäse light
• 2 Knoblauchzehen
• Salz + Pfeffer
• Petersilie (zum garnieren)
```

Teaches: caption already supplies per-portion macros (`macroSource: "manual"`);
nullable quantity/unit for `Salz + Pfeffer` and garnish `Petersilie`; count unit
(`Stück`) for `Knoblauchzehen`; empty `steps` (instructions only in the video).
_(Note: `(Meal Prep)` is a **candidate** tag the user may add in the form — the rule-based
parser does NOT auto-extract tags, T3.1; tags are user-curated via the T2.2 form.)_

---

## Example 2 — Marry Me Chicken Meal Prep (German; grouped sections)

Raw caption:

```
💍🍗 Das perfekte Marry Me Chicken Meal Prep 🍅🔥
Gesund, proteinreich & super easy für 4 Portionen 💪

🍗 Für das Hähnchen:
• 800 g Hähncheninnenfilets
• 1 TL Salz
• 1 TL Pfeffer
• 1 TL Knoblauch
• 1 TL Paprika

🍅 Für die Sauce:
• 1 rote Zwiebel
• 3 Knoblauchzehen
• 20 g getrocknete Tomaten
• 1 EL Tomatenmark
• 200 ml Hühnerbrühe
• 200 g Light Frischkäse
• italienische Kräuter
• 30 g Parmesan
• Salz & Pfeffer

🍚 Beilage:
• 200 g Basmatireis

🌱 Zum Toppen:
• Petersilie
• Parmesan

460kcal · 47g Kohlenhydrate · 53g Eiweiß · 7g Fett
```

Teaches: **ingredient sections** → per-row `group`; German spoon units `TL`/`EL`;
servings in free prose (`für 4 Portionen`); macro labels `Eiweiß`/`Kohlenhydrate`/`Fett`
in a different order; the same ingredient (Parmesan) appearing in two groups.

---

## Example 3 — High Protein Beijing Beef Rice Bowls (English; steps + storage note)

Raw caption (hashtag wall omitted — it is stripped, never auto-tagged):

```
Serves 4 — Per Meal: 535 calories · 53g P · 56g C · 10g F

Crispy Beef Strips
- 800g lean steak (I used rump steaks)
- 20ml soy sauce
- tsp black pepper
- 20g cornflour
- 5g baking soda (tenderises the beef)
- Cooking spray
Cook on high heat with cooking spray till golden and crispy on both sides.

Beijing Beef Glaze
- 60ml soy sauce
- 120ml water
- 2 tbsp sriracha
- 1 tbsp oyster sauce
- 1 tbsp hoisin
- 1 tbsp sweetener
- 5g cornflour mixed with a tbsp of water (prevents clumping)

Other ingredients
- 200g white rice (uncooked weight)
- 2 garlic cloves (diced)
- 1 medium red bell pepper (sliced into 1" pieces)
- 1 medium white onion (halve and slice)
- Sesame seeds (optional garnish)

Storage & Heating: store up to 5 days in the fridge. To reheat, add a tsp of water
to the box and microwave ~3 min, then mix into the rice.
```

Teaches: **real steps** interleaved with sections (flattened into ordered `steps[]`);
parentheticals → `note` (`(diced)`, `(uncooked weight)`); English units `tsp`/`tbsp`
(= `TL`/`EL`); bare `tsp black pepper` → quantity 1; abbreviated macro labels `P`/`C`/`F`;
a **storage/reheating block → `notes`**; hashtag wall stripped.
