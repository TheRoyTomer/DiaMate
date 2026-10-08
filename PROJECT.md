# PROJECT.md

# DiaMate — Project Specification & Roadmap

> **Scope and status (October 8, 2026):** Sections 1–33 specify Phase 1 in detail. Section 34 records the approved high-level Phase 2 and Phase 3 roadmap. Sections 35–36 are cross-phase rules and document governance. This is a specification, not evidence that every described feature has been implemented. Technical feasibility, clinical safety, WhatsApp/Meta policy eligibility, data protection, and CGM provider access require validation before applicable launches.

## 1. Project Goal

Build a local AI-powered diabetes assistant with three distinct user intents:

- `MEAL_ONLY`: analyze food images and estimate carbohydrates without requiring glucose or dose calculation.
- `CORRECTION_ONLY`: handle a glucose correction request without requiring meal images or food analysis; correction calculations in Phase 1 are **simulation-only**.
- `MEAL_AND_CORRECTION`: analyze the meal and independently simulate meal and correction components, displayed separately.

For meal analysis, receive one image by default and optionally multiple images of the same meal; detect foods and the optional **single predefined Reference Object**; confirm the initial detections with the user; estimate portions and carbohydrates; compare independent AI carbohydrate estimates with database-derived estimates; assess uncertainty; request additional details only when useful; and return a balanced per-food carbohydrate breakdown, total estimate, range, confidence and brief explanation. Show deeper calculation details on request.

For correction-only requests, accept current glucose, read validated settings, check any disclosed recent rapid-acting insulin injection against a user-configured **reminder window**, and run a deterministic correction **simulation** without Vision. The reminder window is not a measure of insulin-on-board or proof that insulin is no longer active.

Maintain short multi-turn context during a request. Corrections to detected foods and intended portion may be provided **before** finalization; modifying or recalculating a finalized meal is excluded from Phase 1.

Phase 1 runs locally, but calls to OpenAI and remote nutritional APIs require network connectivity. Do not implement WhatsApp, the Flutter management app, Firebase Authentication, PostgreSQL, CGM integration, or long-term tracking in this phase. No actionable insulin-dosing advice is provided in Phase 1.

---

## 2. Core Architecture Principles

The Agent orchestrates independent services and first identifies request intent (`IntentRouter`):

```text
USER INPUT
    ↓
INTENT ROUTER / AGENT
    ├── MEAL_ONLY / MEAL_AND_CORRECTION
    │   Images + initial food / single reference detection
    │       ↓
    │   USER CONFIRMATION of detected foods and reference
    │       ↓
    │   Portion estimation (Vision, reference, user context)
    │       ↓
    │   AI carbohydrate estimate     NutritionService + portion estimate
    │                    ↘           ↙
    │                  CarbValidationService
    │                         ↓
    │                  UncertaintyManager
    │                         ↓
    │             Carbohydrate result
    └── CORRECTION_ONLY
        Current glucose + settings + recent-injection information
                            ↓
               Reminder / safety context

MEAL_AND_CORRECTION or CORRECTION_ONLY, in test context only:
    validated numerical inputs → deterministic DoseCalculator → simulation result
```

The Agent runs tools, obtains user confirmations, maintains short-term state, and explains results; the LLM does not directly calculate insulin or change settings. Nutrition-service data and an AI estimate may share a mistaken portion-size assumption: agreement does **not** establish accuracy. A separate `UncertaintyManager` decides whether to show a qualified estimate, request focused clarification, or declare insufficient data. `RecentInsulinReminderService` is distinct from dose calculation, and a configurable reminder interval is **not** an insulin activity calculation. Mathematical correctness is distinct from clinical safety. Simulations must never be presented as actual injection instructions.

---

## 3. Technology Stack

- Python and FastAPI.
- OpenAI API with Vision and OpenAI Agents SDK.
- Configurable OpenAI model (e.g. `OPENAI_VISION_MODEL` environment variable); select a final model through measured accuracy, cost and reliability rather than hard-coding a model name.
- `NutritionService` with modular adapters for USDA FoodData Central, Open Food Facts, and an Israeli nutrition dataset **subject to feasibility and source validation**.
- Local JSON/file storage, optional local nutrition cache including source and retrieval metadata.
- Pydantic validation and automated tests with pytest.

Normalize serving units, cooking state and database carbohydrate definitions, including treatment of fiber, before comparing data. Handle failed lookups and source disagreement explicitly; do not fabricate entries. No database server in Phase 1. Keep core services independent of Phase 2 Flutter management, messaging channels (WhatsApp primary; Telegram fallback), Firebase Authentication, and PostgreSQL storage. Channel-specific adapters must not own agent logic.

---

## 4. Treatment Settings

Maintain six user-configured settings:

1. `long_acting_insulin_units`: stored for future context; excluded from Phase 1 meal and correction simulation formulas.
2. `carb_ratio`: grams carbohydrate covered by one unit of rapid insulin; must be positive for meal simulation.
3. `correction_factor`: expected glucose reduction in mg/dL per unit; must be positive for correction simulation.
4. `max_correction_drop`: largest glucose reduction allowed *within the mathematical correction formula*; not itself a clinical safety guarantee.
5. `correction_target`: configured target glucose in mg/dL for the correction formula.
6. `recent_insulin_warning_hours`: user-selected interval for a reminder about a previously disclosed rapid-acting insulin injection; **not** an insulin-on-board estimate or a claim that insulin is inactive outside the window.

Agent may read and explain settings but must never change them autonomously. Validate types and context-appropriate bounds; do not invent medical defaults. Current glucose, meal portions, and last injection time are **request-specific information**, not treatment settings. Simulations are not actionable recommendations.

---

## 5. Local Settings Storage

Store settings in `data/settings.json`, separately from reference files. Use JSON `null` for unset fields rather than zero-filled fake defaults:

```json
{
  "long_acting_insulin_units": null,
  "carb_ratio": null,
  "correction_factor": null,
  "max_correction_drop": null,
  "correction_target": null,
  "recent_insulin_warning_hours": null
}
```

Validate using Pydantic on read, reject invalid or missing fields needed for a given calculation, write changes safely via temporary file and replace, and exclude real personal settings from Git. Provide a nonpersonal `settings.example.json`. Never hard-code individual treatment settings in program logic. Injection time is transient request state in Phase 1; do not imply the system maintains a complete injection history. Storage design should permit Phase 2 migration to user-scoped PostgreSQL settings editable via the Flutter app. WhatsApp may read but must not edit treatment settings.

---

## 6. Dose Calculator

Implement a deterministic, LLM-free `DoseCalculator` with independent operations:

```python
calculate_meal_bolus(estimated_carbs, carb_ratio)
calculate_correction_bolus(current_glucose, correction_target,
                           correction_factor, max_correction_drop)
calculate_total_bolus(meal_bolus, correction_bolus)
```

Original Phase 1 formulas, used strictly for mathematical simulation:

```text
meal_bolus = estimated_carbs / carb_ratio
required_drop = current_glucose - correction_target
if required_drop <= 0:
    correction_bolus = 0
else:
    allowed_drop = min(required_drop, max_correction_drop)
    correction_bolus = allowed_drop / correction_factor
total_calculated_dose = meal_bolus + correction_bolus
```

`MEAL_ONLY` for a carbohydrate estimate does **not** invoke DoseCalculator. `CORRECTION_ONLY` can invoke only the correction simulation without images or a carbohydrate figure. `MEAL_AND_CORRECTION` computes components separately and then sums them. Do not silently round, silently default missing values, or calculate from an unvalidated estimate. Return components separately where applicable.

These formulas omit insulin-on-board, previous dosing, CGM trends, exercise and other clinical factors. `RecentInsulinReminderService` may warn about disclosed recent insulin **before** displaying a simulation, but is not part of the formula; being outside a reminder window never constitutes clearance to give insulin. Do not present these outputs as recommendations to inject.

---

## 7. Dose Calculator Testing

Write automated tests *before* integrating DoseCalculator into the Agent:

- **Meal tests:** normal cases, zero carbs, missing/negative carbs, zero/negative/missing ratio, numeric boundaries.
- **Correction tests:** above/equal/below target, correction below/above maximum drop, invalid or missing factor, missing/negative/impossible inputs and boundaries.
- **Combined test:** one integration test that confirms both independently tested functions are invoked, their results summed and individual components returned, with no implicit rounding.
- **Reminder tests** (separate service): disclosed injection inside/outside configured window, missing injection time, custom warning window, and no representation of reminder end as the end of insulin action.
- **Routing and safety tests:** meal-only does not require glucose; correction-only never calls Vision; calculator can be tested offline without an LLM; invalid inputs stop simulation; no user-visible injection instruction is generated.

Passing software tests establishes implementation behavior, not clinical validity.

---

## 8. Reference Objects

Phase 1 supports **exactly one predefined Reference Object**, configured locally with image and measured dimensions. The object is optional in every meal photo. Attempt detection; if likely present ask for explicit user confirmation before using its dimensions as approximate scale. If absent, unmatched or invalid, continue using other evidence whenever possible. Do not describe scale as exact volume or weight measurement.

Multiple user-defined references and editing/removal are deferred to Phase 2, managed in the Flutter app; WhatsApp performs conversational detection/confirmation only. Preserve extensibility.

---

## 9. Predefined Reference Object Configuration

Instead of a conversational registration flow in Phase 1:

1. Put one reference image in the local reference directory.
2. Supply its ID, name, image path, measurement type and known dimension(s) via JSON.
3. Validate and load at startup.
4. Attempt matching when meal photos arrive; if detected, request user confirmation.
5. Use dimensions for portion context **only** after confirmation.

No user-facing registration, edit or deletion interface in Phase 1. Invalid/missing reference configuration is logged and disables reference use without crashing other meal analysis. Phase 2 will add registration and management through the Flutter app, not through conversational registration in WhatsApp.

---

## 10. Reference Object Model

Keep the extensible model structure despite only one configured instance:

```text
ReferenceObject
├── id
├── name
├── reference_image
├── measurement_type  # width | height | diameter | length
├── width_cm           # optional
├── height_cm          # optional
├── diameter_cm        # optional
└── length_cm          # optional
```

ID, name, image path and supported measurement type are required. The dimension selected by `measurement_type` must exist and be positive; other provided dimensions must also be positive. Validate through Pydantic. No Phase 1 user-facing model editing.

---

## 11. Reference Object Local Storage

```text
data/
├── settings.json
└── reference/
    ├── reference.json
    └── reference.jpg
```

Reference metadata and image are stored separately from treatment settings and persist across restarts. Validate both paths and dimensions on startup. Read through `ReferenceObjectService`; no multi-object collection or registration uploads in Phase 1. Phase 2 adds multiple user-defined objects, images, edits and deletion through the Flutter app, backed by PostgreSQL and protected file storage.

---

## 12. Reference Object Service

Implement `ReferenceObjectService` with Phase 1 operations:

```python
load_reference()
match_reference()
get_dimensions()
```

Load and validate the sole predefined object; match image(s) via Vision; return a potential match or no match rather than inventing one. The Agent (not this service) asks for confirmation. Release dimensions into a meal-estimation request only after a confirmed match. Handle absent/broken configuration gracefully and support testing with reference enabled or disabled.

Phase 2 extensions (Flutter-managed rather than WhatsApp commands): `add_reference`, `list_references`, `get_reference`, `update_reference`, `delete_reference`.

---

## 13. Initial Meal Detection

On meal-related requests, accept one image by default and optionally multiple images of the **same** meal. Identify visible foods and identifiable components (e.g. sauce, toppings), and attempt matching the **single** predefined Reference Object. Incorporate the user's already supplied food description and intended consumption fraction. Separate directly visible components from uncertain inferred ingredients; do not invent invisible foods.

Return structured detections for explicit user confirmation **before** final portion or carbohydrate estimates. Do not calculate insulin here. `CORRECTION_ONLY` entirely bypasses this workflow.

---

## 14. Mandatory User Confirmation Step

After initial meal detection, show the detected food list, any uncertain components, and any potential reference match; ask the user to confirm or correct. No **final** carbohydrate estimate and no dose simulation may be generated before food confirmation. A reference match must be separately confirmed before its dimensions are used. If no reference is detected, no reference confirmation is necessary. Existing user-provided descriptions help detection but do not bypass required explicit confirmation.

Example: “I detected chicken, rice and salad, and possibly your predefined fork. Is that right?” Corrections such as “It is bulgur, not rice; the fork is correct” must update structured state before detailed analysis.

---

## 15. User Corrections

Before finalization, allow user to replace an incorrectly detected food, add a missing food, remove a false positive, reject/confirm the reference, and clarify intended portion (e.g. “I plan to eat half”). Re-run affected analysis only as needed before the final result.

**Phase 1 boundary:** do not implement modifying or recalculating an **already finalized** meal after the result was delivered. Revisiting finalized meals may be considered in a future phase but is not committed to Phase 2 or 3. Do not confuse intended-to-eat quantity with full amount pictured.

---

## 16. Reference Matching

For Phase 1 use the Vision model to compare meal photos with the sole configured reference image. Output a possible match (with qualitative confidence / evidence) or no match. Never manufacture a match. Explicit user confirmation is mandatory even for apparently high-confidence matches. Do not use rejected or uncertain reference dimensions for scaling. Multiple images of one meal can be considered together; scale remains approximate.

---

## 17. Confirmed Meal State

Hold short-lived structured state for the current request:

```text
request_id
intent                 # MEAL_ONLY | CORRECTION_ONLY | MEAL_AND_CORRECTION
meal_images[]          # zero images for CORRECTION_ONLY
initial_detections
confirmed_foods
confirmed_reference    # optional, one predefined object only
reference_dimensions   # only if confirmed
intended_portion       # e.g. whole or half, if supplied
portion_estimates
ai_carb_estimate
nutrition_db_estimate
validation_results
uncertainty_state
current_glucose        # only if relevant
last_rapid_insulin_time # if disclosed; not a complete medical history
simulation_result      # if requested in supported simulation flow
finalized
```

State must persist across several messages while the request remains active. No long-term meal/injection history in Phase 1. After finalization, do not support retroactive editing of that meal.

---

## 18. Meal Vision / Portion Estimation

**Only after confirmation**, analyze all meal images jointly, using confirmed food names, user-supplied portion/consumption information, optional confirmed reference dimensions, and image context. Estimate **per-food portions** as weight or household measure when possible, with uncertainty intervals and explanations of limits. Default is one image; accept additional images proactively supplied by user, and request another angle only if likely useful. Consider top-down versus side-angle cues, occlusion, camera position, unknown plate diameter, sauces and filling.

Phase 1 begins with Vision and reference-size context; a separate geometric volume-to-density estimator is an experimental future option, not a required Phase 1 implementation. Do not assume an image alone provides exact weight.

---

## 19. Meal Analysis Output

Return structured per-food and total estimates. Suggested schema (illustrative, not a mandatory exact JSON contract):

```json
{
  "foods": [
    {
      "name": "cooked rice",
      "estimated_portion": "approximately 1.5 cups",
      "estimated_weight_g": null,
      "ai_carbs_g": 55,
      "database_carbs_g": 53,
      "estimated_carbs_g": 55,
      "min_carbs_g": 45,
      "max_carbs_g": 65,
      "confidence": "medium",
      "nutrition_source": "example-source"
    }
  ],
  "total_estimated_carbs_g": 55,
  "total_min_carbs_g": 45,
  "total_max_carbs_g": 65,
  "overall_confidence": "medium",
  "clarification_required": false
}
```

The numbers above are **illustrative**, not validated nutrition facts. Store unavailable values as `null`, not made-up precision. Preserve source provenance and method disagreement. Structured output distinguishes AI direct estimates, database-derived estimates and the chosen **qualified** working estimate. Detailed audit/explanation is available on request.

---

## 20. Reference Object Scale Context

If the predefined Reference Object is **detected and confirmed**, pass its measured dimensions into portion analysis, to inform approximate plate size, food footprint, relative size or volume. Explain that perspective, distance, depth, occlusion, food shape and camera angle limit accuracy. If no confirmed reference is available, estimate cautiously without it or request other supporting information. No reference is mandatory in a meal photo. Test with/without reference on measured meals before claiming improved accuracy.

---

## 21. Carb Estimation Responsibility

Two estimators contribute after food confirmation:

1. **AI direct estimate:** Vision interprets identified foods, portion size and carbohydrate content; reports uncertainty.
2. **Nutrition database estimate:** `NutritionService` retrieves suitable per-weight or per-serving carb data; Python combines values with estimated portion weights/serving sizes and handles unit normalization and source provenance.

`CarbValidationService` compares estimates and investigates significant disagreement. Small difference is **not proof** of accuracy (both may share the same wrong portion estimate). Large difference never automatically means the database is correct: request reliable extra evidence or provide an uncertainty-qualified result. Measured weight and precise product nutrition label take precedence over image guesses where applicable. No final selection thresholds are clinically or statistically validated yet; calibrate using real weighed meals.

The LLM does not perform insulin dose calculations: that belongs only to deterministic Python simulation.

---

## 22. Main Meal Flow

```text
Meal input (one image by default; optional more; optional user description)
    ↓
Detect foods, components, possible single predefined reference
    ↓
Show detections → WAIT FOR EXPLICIT USER CONFIRMATION
    ↓
Apply corrections / intended fraction BEFORE finalization
    ↓
Vision portion-size estimate with optional confirmed scale
    ↓
AI carbs estimate + NutritionService estimate
    ↓
CarbValidationService (sources, agreement, discrepancy)
    ↓
UncertaintyManager
    ├── enough information → qualified final carb estimate
    ├── useful clarification → ask for another image/weight/portion info
    └── insufficient information → explain limits; do not fabricate
    ↓
Balanced response: per-food carbs, total, range, confidence, brief reasons
    ↓
If MEAL_AND_CORRECTION in testing: obtain glucose/settings,
check disclosed recent injection, simulate components separately.
```

`MEAL_ONLY` needs no glucose. `CORRECTION_ONLY` bypasses this flow and handles glucose/settings/recent-injection reminder and correction simulation directly. Finalized meals are not reopened or edited in Phase 1.

---

## 23. Agent Responsibilities

The Agent identifies request intent first; invokes only needed services; manages image grouping and short multi-turn state; presents initial food/reference detections for confirmation; applies pre-finalization user corrections; finds missing information; reads but never autonomously edits settings; coordinates AI and database carb estimates; invokes validation and uncertainty decisions; obtains glucose only for a relevant simulation; requests information about a recent injection where relevant and handles configured reminder; and communicates a balanced estimate with optional detailed derivation.

The Agent must not fabricate current glucose, treatment parameters, missing food components, reference dimensions, injection history, or safety clearance. Avoid over-questioning: one or two targeted clarifications may be useful, but no endless loop; explain when a reasonable estimate is impossible. Do not deliver an actionable insulin dose in Phase 1.

---

## 24. Agent Tools

Suggested narrow-purpose tools/services:

```text
classify_request_intent()
get_treatment_settings()
load_reference()
match_reference()
get_dimensions()
detect_foods()
analyze_confirmed_meal()
get_nutrition_data()
compare_carb_estimates()
assess_uncertainty()
check_recent_insulin_reminder()
calculate_meal_bolus()           # simulation only
calculate_correction_bolus()     # simulation only
calculate_total_bolus()          # simulation only
```

Do **not** implement `add_reference`, `list_references`, `update_reference`, `delete_reference` as conversational Phase 1 tools. Those are Phase 2 app/backend management operations, not conversational editing tools in WhatsApp. Keep service code separate from Agent prompting. Reference configuration is one local object; database cache is separate from settings.

---

## 25. Multi-Turn Conversation Example

**Meal-only example:**

```text
User: [photo of chicken, bulgur and salad] "How many carbs? I'll eat half."
Agent: "I identified chicken, rice, salad, and possibly your predefined fork.
        Is that correct?"
User: "That's bulgur, not rice; the fork is correct."
Agent: "Confirmed: chicken, bulgur, salad and the reference fork.
        I'll estimate the half-portion you plan to eat."
Agent: [uses both estimates, checks uncertainty, and if sufficient,
        returns per-food carbohydrate range, total and confidence]
```

**Correction-only example:**

```text
User: "My glucose is high; I'm at 245. I want to check a correction."
Agent: [routes to CORRECTION_ONLY; asks for missing necessary details,
        checks any disclosed prior rapid insulin against the configured
        reminder window; explains limitations]
System: [can compute mathematical correction in a test-only simulation;
         not an instruction to inject]
```

Never infer that no earlier insulin is active merely because the user has not mentioned an injection.

---

## 26. Missing Information

Never invent clinically significant or estimation-critical details. Examples: unconfirmed food identity, hidden sauces, unclear portion size, unconfirmed reference match, missing configured reference dimensions, incorrect/poor image, missing glucose for a correction simulation, missing treatment settings, missing injection history. Lack of a recorded injection time is **unknown**, not evidence of no insulin on board.

Request only relevant clarifications, such as another angle, package nutrition, measured weight, household serving size or a known object. When extra details cannot solve uncertainty, communicate that an adequately supported estimate is unavailable. `CORRECTION_ONLY` does not ask for photos or food data.

---

## 27. Uncertainty Handling

Implement `UncertaintyManager` using **per-food** evidence along five axes:

1. Confirmed identification of each food.
2. Portion-size quality and uncertainty.
3. Nutritional source relevance and quality.
4. Disagreement between AI and nutritional database estimates.
5. Missing ingredients, sauces, fraction to be eaten or other essential information.

Surface qualitative `high`, `medium`, or `low` confidence, with an estimate **range** and concise reason. States/actions: `SHOW_ESTIMATE`, `REQUEST_CLARIFICATION`, `INSUFFICIENT_DATA`. A high LLM-reported confidence is not a calibrated probability. Do not choose fixed “safe” 15%/30% discrepancy thresholds before validation. A large disagreement can trigger clarification, not automatic selection of database output. Agreement cannot validate common weight-estimation errors.

Select follow-up questions that can materially reduce uncertainty; request another image only if likely helpful; do not repeatedly nag. If evidence remains insufficient, say so. An estimation-confidence label **never** constitutes medical clearance to calculate or recommend an actual insulin dose. Separate any future `MedicalSafetyGate` from uncertainty scoring.

---

## 28. User Response Format

Default **balanced** response, in conversational style suitable for eventual WhatsApp:

- Confirmed foods with **carbohydrate estimate per food**.
- **Estimated total** displayed clearly.
- **Range** and `high`/`medium`/`low` confidence.
- Brief explanation of major assumptions and sources of uncertainty.

On user request such as “How did you calculate that?”, show estimated serving/weight, AI estimate, database estimate and provenance, discrepancies and limitations. Keep full technical detail out of the default response. Do not display simulation results as actionable injection recommendations.

In Phase 1 the user can correct foods/portion **before** final result, but cannot change and recompute an already finalized meal. If inadequate evidence exists, communicate missing information rather than force a final numeric answer.

---

## 29. Real Meal Testing

Once the full meal analysis flow works, assemble an initial **20–30 weighed/reference-measured meals**, covering simple foods and mixed dishes (e.g. rice, pasta, bread, hummus, pizza, sauces). Record images, true food identity, AI detection, user corrections, actual prepared weight where possible, labels/recipes, reference presence, model estimated portions, AI carbs, database carbs, final estimate/range and confidence.

Compare on the same meals using these configurations:

A. One image only.
B. Two angles of the same meal.
C. Images with the predefined Reference Object.
D. Images with the reference and additional user-supplied portion information.

Collect true labels or measured weights before testing to minimize circular evaluations. Multiple images may help but are not assumed superior until evaluated. The initial sample supports exploration, **not clinical validation**.

---

## 30. Accuracy Evaluation

Assess separately:

- Food identification versus confirmed ground truth and frequency of successful user corrections.
- Portion weight error by food and by photographic configuration.
- Carbohydrate error against trustworthy measured/recipe/label references; coverage of estimated ranges.
- AI-only versus database-derived versus validated-combined workflow.
- Detection and handling of disagreements, including cases of shared portion-size error.
- Benefit (or lack thereof) from reference and additional camera angles.
- Frequency of large actual errors that the system nevertheless labeled **high confidence**.
- Whether focused clarifications reduced error or merely added user friction.

Do not establish arbitrary confidence thresholds as clinically safe. Evaluate pilot findings and expand testing before using any results for health decisions. If reference objects do not materially help, reconsider the approach for future phases.

---

## 31. Suggested Project Structure

```text
DiaMate/
├── app/
│   ├── main.py
│   ├── agent/
│   │   └── diabetes_agent.py
│   ├── services/
│   │   ├── intent_router.py
│   │   ├── meal_detection.py
│   │   ├── meal_vision.py
│   │   ├── nutrition_service.py
│   │   ├── carb_validation.py
│   │   ├── uncertainty_manager.py
│   │   ├── dose_calculator.py
│   │   ├── reference_service.py
│   │   └── recent_insulin_reminder.py
│   ├── nutrition_adapters/
│   │   ├── usda.py
│   │   ├── open_food_facts.py
│   │   └── israeli_foods.py   # feasibility to be confirmed
│   ├── models/
│   │   ├── meal.py
│   │   ├── treatment_settings.py
│   │   └── reference_object.py
│   └── config.py
├── data/
│   ├── settings.json        # private; gitignored
│   ├── reference/
│   │   ├── reference.json
│   │   └── reference.jpg
│   └── nutrition_cache/
├── tests/
│   ├── test_dose_calculator.py
│   ├── test_reference_service.py
│   ├── test_meal_detection.py
│   ├── test_nutrition_service.py
│   ├── test_carb_validation.py
│   ├── test_uncertainty_manager.py
│   ├── test_intent_router.py
│   └── test_recent_insulin_reminder.py
├── settings.example.json
├── .env
├── .gitignore
├── requirements.txt
└── PROJECT.md
```

This is a **proposed** layout, not proof these files exist. Do not commit real private treatment data or secrets. The local test interface may be API-based or a simple browser chat; WhatsApp and standalone mobile interfaces are later integrations.

---

## 32. Phase 1 Exclusions

Do **not** implement in Phase 1:

- WhatsApp or a production mobile app.
- PostgreSQL or other database server; user account management.
- Libre/Dexcom/CGM retrieval or history.
- MCP, long-term meal or injection history, long-term trend analysis.
- Autonomous treatment adjustments, carb-ratio/correction-factor adaptation or personalized clinical dosing recommendations.
- Complete insulin-on-board model or production clinical safety clearance.
- Conversational registration, multi-object management, user editing or removal of references.
- Editing/recalculating an already finalized meal.
- Complex 3D camera reconstruction as a Phase 1 dependency.
- Automatically asserting that a reminder time window determines residual insulin activity.

The Phase 2 and Phase 3 roadmap is specified in section 34; these systems remain outside Phase 1.

---

## 33. Phase 1 Definition of Done

Phase 1 is done when **all** of the following local flows and validations work:

**Flow A — Predefined reference configuration**

```text
Local reference image + JSON dimensions
    ↓
Startup validation / loading
    ↓
Reference match attempted (when image is provided)
    ↓
User confirmation if likely detected
    ↓
Dimensions supplied to portion estimator only if confirmed
```

The service must function gracefully if no usable reference is configured; object registration is **not** required.

**Flow B — Meal-only analysis**

```text
One or several photos of one meal
    ↓
Initial food + optional predefined reference detection
    ↓
Explicit user confirmation and pre-final corrections
    ↓
Portion estimate with range
    ↓
AI + nutrition-database carbohydrate estimates
    ↓
Validation of disagreement + uncertainty assessment
    ↓
Focused question if useful; otherwise qualified final answer
    ↓
Balanced per-food/total carb estimates, range, confidence,
with on-demand detail
```

**Flow C — Correction-only simulation**

```text
Correction intent + supplied current glucose
    ↓
Validate relevant user settings
    ↓
Review disclosed recent injection and show reminder if applicable
    ↓
Run deterministic correction simulation, without Vision or meal data
    ↓
Clearly label result as test-only / non-actionable
```

**Flow D — Combined simulation**

Use the confirmed meal analysis plus correction-specific inputs; invoke separately verified meal and correction calculations and sum using one tested integration path. Maintain clear component separation and non-actionable simulation labeling.

**Across all flows:** support multi-turn current-request context; unit tests and invalid-input behaviors pass; original meal cannot be retroactively edited in Phase 1; uncertainty is communicated and unsupported estimates are not fabricated; evaluation on an initial 20–30 reference meals is documented; do not characterize mathematical or image accuracy as clinical safety.

---

## 34. Future Phases and Product Roadmap

### Product Experience — Approved Direction

DiaMate is designed around two complementary user experiences that share one backend and one user identity:

- **Flutter mobile management app:** account registration/login; management of treatment settings and user-defined Reference Objects; review of meal, carbohydrate, glucose and injection history; future CGM views and reports. App screen layouts are deferred until implementation.
- **WhatsApp conversational agent:** the **primary and intended sole conversational interface** for meal photos, clarification questions, food analysis, glucose-related requests, and explanations. Users initiate conversations; DiaMate does not send unsolicited messages or proactively start WhatsApp chats.
- **Shared FastAPI + DiaMate Agent/Core:** both channels access the same user-owned records through permission-checked backend services. The app is not required to include a second agent chat interface.
- **Messaging adapter boundary:** build `MessagingAdapter`/provider adapters so **Telegram** can replace WhatsApp if Meta rejects or restricts the intended deployment. Telegram is a contingency, **not** an additional mandatory Phase 2 feature.

Phase 1 is still local-first and precedes Flutter, WhatsApp and the multi-user database.

### Phase 2 — Flutter Management App, Messaging and Persistent Multi-User Platform

**Goal:** Turn the validated local agent into a persistent, multi-user product. Merge the originally separate WhatsApp and database/tracking phases into this phase.

**2A. Accounts, backend and storage**

- Adopt **Firebase Authentication** for account registration and login; verify identity tokens in the backend.
- Adopt **PostgreSQL** for user-scoped profiles, treatment settings, Reference Objects metadata, conversations/session state, structured meals and carbohydrate estimates, manually supplied glucose readings, injection records and relevant audit metadata.
- Store uploaded meal/reference images in protected file/object storage and database references, rather than assuming images must be stored as database blobs.
- Build authorization and isolation across users for every read/write; do not equate a messaging number alone with complete authorization.
- Persist enough context to resume active conversations across server restarts and deduplicate inbound messages. Do not indefinitely store entire messaging transcripts by default; define retention, deletion and user-access policies.
- Store **user-reported** injections and doses distinctly from mathematical **simulation outputs**; a simulation is not an actual or advised injection.

**2B. Flutter management application**

- Account onboarding, sign-in, management of settings and profile, history, and saved objects.
- Users **edit treatment settings in Flutter only**, with explicit confirmation and validation for sensitive changes. WhatsApp may answer questions about saved settings, **but cannot modify them**.
- Users register, photograph, name, dimension, inspect, edit and delete **multiple Reference Objects in Flutter**. The WhatsApp agent only detects possible matches in meal images and asks for confirmation before use.
- Show structured history of meals, carbohydrate estimates, user-supplied glucose readings and reported injections. Do not assume the app needs an embedded AI chat screen.
- Provide a prominent **“Talk to DiaMate on WhatsApp”** action.

**2C. WhatsApp conversational agent and secure account linking**

- Use the official **WhatsApp Business Platform / Cloud API**, subject to acceptance of the planned AI/health-data use and applicable legal/privacy requirements.
- A Flutter button opens the DiaMate WhatsApp conversation via a deep link with a **pre-filled, short-lived, single-use account-linking message**. The user taps Send; the backend verifies the opaque linking token and associates the sender with the signed-in account. Users do **not** have to manually copy/enter a code.
- Do not place treatment values or account secrets in the pre-filled message. Tokens expire, are one-use, are validated server-side, and links can be revoked or re-established in the app.
- User **initiates** each conversation. DiaMate responds to user-initiated chat messages; no proactive reminders, outbound campaigns or unsolicited new conversations are in scope.
- Preserve user-specific conversation context, multiple meal photos, required confirmation, focused clarifications, nutritional validation and balanced responses from Phase 1.
- `MEAL_ONLY`, `CORRECTION_ONLY` and `MEAL_AND_CORRECTION` intent routing remain independent. Carry forward the restriction against giving unvalidated actionable insulin-dosing instructions.
- Handle media retrieval securely; validate webhook signatures, message IDs, idempotency and delivery failures. Messaging integrations must not bypass backend permissions.
- **Telegram fallback** uses the same agent/core and database, replacing only the messaging provider adapter and corresponding account-linking channel where possible.

**2D. Beta, verification and launch readiness**

- Start with a **closed, invitation-only beta** involving the project owner and a small number of additional test users, each with private data and settings.
- Architect for an eventual **public multi-user product** from the outset, without committing to public launch on a fixed phase date. Public release may occur at the end of Phase 2 or during Phase 3, **only when readiness gates are satisfied**.
- Complete security/privacy reviews, data isolation tests, consent, retention/deletion, account recovery, source-of-truth validation and operating-cost checks before wider rollout.
- **WhatsApp eligibility is not confirmed.** Verify current Meta AI-provider/business policy and handling of health-related data before processing real sensitive user data on the platform. User-initiated messaging addresses the proactive-message issue but does not itself establish policy eligibility.
- Phase 1 insulin formulas are *simulations*, not validated clinical dosing. Opening the product publicly does **not** automatically authorize live insulin recommendations or clinically validated treatment adjustment. Obtain appropriate clinical/regulatory review separately before offering such capabilities.

**Phase 2 explicit non-goals:** CGM-provider synchronization and advanced glucose-history analytics (Phase 3); autonomous changes to therapy; automatic dosing changes; a second conversational agent built into the Flutter app; final UI designs as part of this high-level roadmap.

### Phase 3 — Automatic CGM Synchronization and Glucose Intelligence

**Goal:** Extend DiaMate to retrieve authorized CGM data and provide current/latest available glucose, historical analysis, and clinician-discussion insights.

**Integration and ingestion**

- Prioritize an **authorized, automatic synchronization** path for **Libre and/or Dexcom**, whichever has feasible provider-approved access first. Maintain provider-specific `CGMAdapter` implementations behind a common `CGMService` contract.
- Validate developer approval, country/device compatibility, allowed data fields, access scopes, latency, quotas, and contract/privacy conditions. **Do not promise access to either provider until verified**, and do not use unauthorized bypasses.
- Phase 3 uses **automatic synchronization only**; manual CGM export/file-import workflows are explicitly out of scope.
- Store readings with source, measurement timestamp, ingestion timestamp, units, quality/staleness status and user ownership. Normalize units, avoid duplicates, handle outages and delayed readings, and allow disconnection and deletion of imported data.
- Return the **latest available** glucose with measurement time and a trend indicator **only if the provider supplies reliable trend data**; never claim a delayed value is live.

**Three levels of Phase 3 capabilities**

1. **Data display:** CGM history, glucose graphs, averages, time-in-range, time-below-range and other interpretable metrics when data sufficiency permits; show timestamps, gaps and limitations.
2. **Analysis and insights:** compare historical glucose with recorded meals and injections, inspect post-meal curves, identify recurring patterns, and explain associations without presenting them as proven causal effects.
3. **Treatment Review Insights:** surface **questions and topics for discussion with a clinician**, based on supporting patterns and data completeness. It may flag areas such as breakfast carbohydrate ratios, timing, persistent overnight highs or recurrent lows as subjects to evaluate together with a healthcare professional. **Do not output numeric prescriptions to change basal insulin, carb ratios, correction factors, or bolus doses; do not autonomously change settings.** A disclaimer alone does not establish safety or regulatory compliance.

**Clinician report**

- Offer a user-requested structured summary for a diabetes appointment: relevant CGM metrics, recurring patterns, documented meals/injections, episodes of low glucose, identified limitations and questions for the care team.
- Clearly distinguish recorded facts, calculations, statistical observations and tentative AI interpretations. The report is for discussion, **not** a diagnosis or a treatment order.

**Safety and scope**

- Apply data freshness/completeness checks; make missing meals/injections explicit. Do not infer missing doses or assert clinical causality from correlation.
- No autonomous treatment adjustments, insulin pump control, or numeric personalized dose-change prescriptions.
- Medical-device, data-processing, provider and clinical safety obligations require assessment before any applicable real-user functionality goes live.

### Later / Optional Capabilities (Not Committed to Phases 1–3)

- Editing and recomputing finalized past meals, if demonstrated useful.
- Optional additional messaging providers or non-chat interfaces.
- More advanced CGM forecasting and proactive notifications, subject to safety, platform rules and a distinct approval decision.
- Clinically validated therapy-adjustment support, if ever pursued, as a separate project with dedicated medical/regulatory evaluation.

---

## 35. Non-Negotiable Rules

1. The LLM must **not autonomously change** treatment settings. In Phase 2, authorized user changes occur via Flutter with validation and confirmation; WhatsApp is read-only for settings.
2. The LLM must **not directly calculate insulin doses**. Any mathematical insulin computations are performed by deterministic, separately testable Python code.
3. Phase 1 dose outputs are **simulation-only** and may not be presented as instructions to inject. Deploying Phase 2 or Phase 3 does not silently remove this restriction.
4. Long-acting insulin does not participate in the Phase 1 meal or correction simulation formulas; a recent-injection reminder window is **not** an insulin-on-board calculation.
5. In **meal-related** flows, user confirmation of detected foods is mandatory before final carbohydrate estimation or meal-bolus simulation. **Correction-only** requests do not need a food image or food confirmation.
6. A detected Reference Object must be confirmed before being used for scale; no object is required for every meal. Phase 1 has one predefined object; Phase 2 has multiple Flutter-managed objects.
7. Do not invent missing glucose readings, treatment settings, portion dimensions, injection records, nutrition database facts, CGM values or provider permissions.
8. Clearly represent portion/carbohydrate uncertainty and information provenance. AI/database agreement can reflect shared error and is not proof of accuracy.
9. Separate structured nutrition analysis from mathematical dose simulation and from any medical-safety assessment. Do not equate technical test success with clinical validation.
10. Build provider-agnostic interfaces for messaging and CGM, with user-scoped authorization for all persisted information.
11. The WhatsApp channel is user-initiated, and use of WhatsApp for a health-related AI product remains conditional on demonstrated platform-policy and legal/data-protection eligibility. Prefer Telegram as fallback if needed.
12. Phase 3 insights may propose **discussion topics for a clinician**, never numeric autonomous adjustment instructions or treatment changes.
13. Public availability is contingent on privacy, platform, reliability and applicable clinical/regulatory readiness. A planned launch phase is not itself evidence of compliance.

---

## 36. Source of Truth and Change Control

This document is the source of truth for **DiaMate project scope and the approved high-level roadmap** as of October 8, 2026.

- Sections **1–33** specify the Phase 1 build in detail.
- Section **34** records the Phase 2/3 roadmap and deferred options; roadmap requirements still require implementation-stage refinement.
- Section **35** contains cross-phase non-negotiable behavior and safety boundaries.
- Section **37** tracks implementation progress; checklist completion never overrides requirements or safety rules.
- If a detailed implementation conflicts with this document, prefer the requirements here unless the user explicitly approves a change.
- Do not silently reinterpret a planned capability as implemented or a proposed provider integration as approved.
- Revisit the roadmap when Meta/WhatsApp or CGM-provider eligibility, clinical review, validation results or user requirements materially change.

---

## 37. Development Checklist & Progress Tracking

This section tracks small, verifiable implementation tasks across all approved phases. It is a **living implementation tracker**, not a replacement for Sections 1–36.

**Checklist rules**

- `[ ]` = not yet verified as complete; `[x]` = implemented, tested, and verified against the relevant requirements.
- Tasks are **not** automatically complete because a design was approved or code was written. Mark completion only after checking the code and relevant test results.
- A task may be checked only with a brief factual record of evidence (test result, reviewed behavior, or other validation), supplied in a commit, pull request, or development summary.
- If a task becomes obsolete or scope changes, obtain project-owner approval and update both the requirements and this checklist rather than silently deleting or checking it.
- Never use task completion as a substitute for medical validation, privacy review, platform approval, or provider authorization.
- **All tasks below start unchecked:** completion status has not been audited against the current repository. Previously implemented items should be checked only after verification.
- Track changes to this checklist in version control alongside the code; future `AGENTS.md` instructions should define how AI development agents update it.

### Phase 1 — Local AI Core

#### 1A. Repository and local environment
- [ ] Verify existing repository structure, dependencies, and runnable FastAPI skeleton.
- [ ] Set up documented local Python environment and dependency installation.
- [ ] Configure environment variables for OpenAI model and credentials; prevent secret commits.
- [ ] Add reliable application startup and basic health endpoint.
- [ ] Add a simple local testing interface/API supporting text and image uploads.
- [ ] Establish a reproducible `pytest` setup and baseline test command.

#### 1B. Treatment settings and deterministic simulation
- [ ] Define Pydantic model for the six approved treatment/reminder settings.
- [ ] Add `settings.example.json` with unset values and local private `settings.json` storage.
- [ ] Validate loaded settings and reject missing, invalid, or impossible inputs.
- [ ] Implement safe settings reads/writes and appropriate `.gitignore` rules.
- [ ] Implement independent meal-bolus simulation function.
- [ ] Implement independent correction-bolus simulation function.
- [ ] Implement combined mathematical result from the two independent functions.
- [ ] Ensure correction-only requests need no meal image or carb estimate.
- [ ] Keep simulation output clearly non-actionable and separate from safety checks.
- [ ] Implement the user-configured recent-injection reminder as a separate service.
- [ ] Test meal simulation cases and invalid/boundary inputs.
- [ ] Test correction simulation cases, limits, and invalid/boundary inputs.
- [ ] Test one combined orchestration scenario and recent-injection reminders.

#### 1C. Single predefined Reference Object
- [ ] Choose, photograph, and measure one real reference object.
- [ ] Save its image and local JSON configuration.
- [ ] Define and validate the reusable `ReferenceObject` model.
- [ ] Load reference metadata/image without crashing if missing or invalid.
- [ ] Detect a possible reference match from a meal image using Vision.
- [ ] Require explicit user confirmation before using dimensions.
- [ ] Continue estimation without a reference when none is present or confirmed.
- [ ] Test matching, rejection, and missing-reference behavior.

#### 1D. Food detection and conversations
- [ ] Implement initial image-based food/component detection with structured output.
- [ ] Accept one image by default and multiple images for the same meal.
- [ ] Incorporate user-provided food descriptions and intended eaten fraction.
- [ ] Present detected foods and possible reference for explicit confirmation.
- [ ] Support add/remove/replace corrections before final analysis.
- [ ] Preserve structured meal state over multiple chat turns.
- [ ] Implement intent routing for `MEAL_ONLY`, `CORRECTION_ONLY`, and `MEAL_AND_CORRECTION`.
- [ ] Ensure correction-only requests bypass food detection and reference matching.
- [ ] Prevent final carb estimation and meal simulation before food confirmation.
- [ ] Test multi-turn confirmation, correction, and intent routing.

#### 1E. Portion and carbohydrate estimation
- [ ] Implement confirmed-meal portion estimation with weight/size uncertainty ranges.
- [ ] Use confirmed reference dimensions as approximate scale context when available.
- [ ] Support clarification requests and additional photos only when useful.
- [ ] Implement a direct AI carbohydrate estimate per identified food.
- [ ] Implement `NutritionService` with modular provider adapters.
- [ ] Add an initial supported nutrition database lookup and source metadata.
- [ ] Handle nutritional definitions, serving units, preparation state, and missing fields.
- [ ] Add local cache for retrieved nutritional values.
- [ ] Calculate database-based carbohydrate estimate from food and portion inputs.
- [ ] Implement `CarbValidationService` for per-food and meal-level comparisons.
- [ ] Implement `UncertaintyManager` to decide estimate, clarification, or insufficient data.
- [ ] Do not treat agreement of AI and database paths as proof of accuracy.
- [ ] Test calculation paths, nutritional mapping, provenance, and uncertainty outcomes.

#### 1F. Agent and response experience
- [ ] Connect the OpenAI Agents SDK to narrow, well-defined service tools.
- [ ] Make tools respect the agent's intent routing and confirmation requirements.
- [ ] Return the approved balanced response: per-food carbs, total, range, confidence, explanation.
- [ ] Support user-requested detailed breakdown including sources and estimated portions.
- [ ] Clearly separate nutritional estimates from any insulin simulation.
- [ ] Handle missing glucose/settings/reference data without inventing values.
- [ ] Do not implement editing/recomputing finalized meals in Phase 1.
- [ ] Verify end-to-end reference-supported and reference-free meal conversations.
- [ ] Verify end-to-end correction-only and combined simulation conversations.

#### 1G. Evaluation and Phase 1 completion
- [ ] Assemble 20–30 documented test meals with ground-truth or reliable comparison data where available.
- [ ] Evaluate initial food-detection accuracy against user-confirmed labels.
- [ ] Measure weight/portion and carbohydrate estimation error and uncertainty calibration.
- [ ] Compare one-photo versus additional-photo test conditions.
- [ ] Compare outcomes with and without the predefined Reference Object.
- [ ] Compare direct AI estimates with nutrition-database estimates and failure cases.
- [ ] Document model choice, API costs, limitations, and measurable results.
- [ ] Run all automated tests and complete the two Phase 1 end-to-end acceptance flows.
- [ ] Review non-negotiable rules and keep insulin calculation capabilities simulation-only.

### Phase 2 — Multi-User Product, Flutter and WhatsApp

#### 2A. Persistent multi-user backend
- [ ] Design PostgreSQL schema for users, settings, conversations, meals, insulin events and references.
- [ ] Add database migrations and backup/recovery approach.
- [ ] Integrate Firebase Authentication and backend token verification.
- [ ] Enforce user-specific authorization on every sensitive endpoint and stored record.
- [ ] Implement invite-only closed beta onboarding.
- [ ] Migrate Phase 1 settings and reference data through appropriate services.
- [ ] Store structured meal results, glucose entries, reported injections, and minimal necessary chat state.
- [ ] Define retention, deletion, and secure image storage behavior.
- [ ] Test access isolation across multiple accounts and recovery from failures.

#### 2B. Flutter management application
- [ ] Implement registration/login and account management.
- [ ] Implement viewing and editing settings with explicit user confirmation and validation.
- [ ] Implement multiple Reference Objects: create, image upload, dimensions, update, delete.
- [ ] Implement viewing structured meal/glucose/injection history.
- [ ] Provide controls for account data deletion, privacy, and WhatsApp linking status.
- [ ] Implement 'Talk to DiaMate' action that opens a prefilled WhatsApp chat.
- [ ] Keep agent conversation and food-image analysis outside the Flutter app.

#### 2C. WhatsApp integration (eligibility-dependent)
- [ ] Verify current Meta eligibility for the intended health-related AI service and data use.
- [ ] Confirm WhatsApp Business Cloud API setup and production access requirements.
- [ ] Implement a provider-neutral `MessagingAdapter` contract.
- [ ] Implement secure one-time account linking through an app-initiated WhatsApp message.
- [ ] Handle inbound text/images, media retrieval, message delivery, retries, and duplicates.
- [ ] Restore the right user and conversation state across independent WhatsApp messages.
- [ ] Make the bot reply to user-initiated interactions only; no unsolicited outreach.
- [ ] Support image-based meal analysis, confirmations, questions, and detailed breakdown via chat.
- [ ] Permit reading settings via WhatsApp; route settings changes to Flutter.
- [ ] Add unlink/relink handling and prohibit revealing data for unmatched/unverified accounts.
- [ ] Implement Telegram adapter only if the fallback is needed and approved.

#### 2D. Closed beta and launch readiness
- [ ] Perform authentication, privacy, security, and multi-user separation testing.
- [ ] Test reliability and conversational recovery under duplicate/delayed messages.
- [ ] Track operational errors, API usage, and per-user costs without exposing sensitive data in logs.
- [ ] Gather closed-beta usability and portion-estimation feedback with appropriate safeguards.
- [ ] Assess legal, platform, data protection, and applicable medical-safety obligations before real-user availability.
- [ ] Decide public-launch timing only after the required readiness reviews; do not equate closed beta with clinical validation.

### Phase 3 — Automatic CGM and Glucose Intelligence

#### 3A. Authorized provider integration
- [ ] Check approved API access and geographic/device limitations for Libre and Dexcom.
- [ ] Select a supported provider for the first authorized automatic integration.
- [ ] Implement a provider-neutral `CGMService` and provider-specific adapter.
- [ ] Implement user authorization, connection, refresh, and revocation flows.
- [ ] Automatically sync the authorized CGM data without manual file import.
- [ ] Normalize units, timestamps, freshness, provenance, and incomplete readings.
- [ ] Avoid duplicate records and handle delayed/missing readings and provider outages.
- [ ] Allow user disconnection and removal of imported data.

#### 3B. Data access, visualization, and analysis
- [ ] Display latest available glucose with its measurement time and staleness status.
- [ ] Show trend information only when reliable provider data supports it.
- [ ] Display glucose history, charts, averages, time-in-range, and time-below-range with coverage checks.
- [ ] Link CGM records to documented meals and injections without inventing missing entries.
- [ ] Analyze post-meal glucose trajectories and recurring temporal patterns.
- [ ] Answer historical glucose questions through the agent using grounded data.
- [ ] State missing data, limitations, and the difference between correlation and causation.

#### 3C. Clinician discussion insights and report
- [ ] Generate evidence-linked treatment-review discussion topics, without numerical change instructions.
- [ ] Never autonomously modify treatment settings or prescribe dose changes.
- [ ] Generate a user-requested clinician report with metrics, relevant events, patterns, and questions.
- [ ] Distinguish raw CGM measurements, calculations, and tentative AI interpretations.
- [ ] Evaluate clinical safety, legal scope, data accuracy and access requirements before enabling real-user functionality.
- [ ] Test report accuracy, edge cases, missing data handling, and privacy controls.

### Deferred / Optional — Not committed to Phases 1–3

- [ ] Reassess editing and recalculating previously finalized meals if needed.
- [ ] Consider additional messaging interfaces beyond the selected primary/fallback channels.
- [ ] Evaluate proactive notifications or CGM forecasting only through a separately approved safety review.
- [ ] Consider any clinically validated dose-adjustment support as a separate explicitly approved project.

**Maintenance note:** When implementation work begins, review this checklist alongside the relevant detailed sections and keep each task's completion status accurate. Approval of the specification is not proof that a feature has been built.

