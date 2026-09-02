# PROJECT.md

# Diabetes Meal Agent — Phase 1

## 1. Project Goal

Build a local AI-powered diabetes meal assistant that can:

1. Receive a meal image.
2. Detect the foods visible in the image.
3. Detect whether a previously saved Reference Object appears in the image.
4. Ask the user to confirm or correct the detected foods and detected Reference Object.
5. Only after confirmation, estimate portion sizes and carbohydrate content.
6. Receive the user's current glucose level.
7. Read predefined personal treatment settings.
8. Calculate meal bolus using deterministic Python code.
9. Calculate correction insulin using deterministic Python code.
10. Return a clear explanation separating AI estimation from mathematical dose calculation.
11. Maintain short conversational context when additional information or corrections are needed.

Phase 1 is local only.

Do NOT implement WhatsApp, PostgreSQL, Libre, Dexcom, MCP, CGM history, long-term tracking, or treatment-adjustment recommendations in this phase.

---

## 2. Core Architecture Principles

The system must separate responsibilities clearly.

```text
DETECTION
"What food/reference appears in the image?"

        ↓

USER CONFIRMATION
"Is the identification correct?"

        ↓

VISION / ESTIMATION
"How much food is present and how many carbohydrates are estimated?"

        ↓

DETERMINISTIC CALCULATOR
"What does the predefined insulin formula calculate?"

        ↓

AGENT
"What information is missing, which tool should run, and how should the result be explained?"
```

The LLM must never directly invent or calculate the insulin dose.

All insulin calculations must be performed by deterministic Python code.

---

## 3. Technology Stack

Use:

- Python
- FastAPI
- OpenAI API
- OpenAI Agents SDK
- Local JSON/file storage for Phase 1
- Automated tests with pytest or equivalent

Do not add a database server in Phase 1.

---

## 4. Treatment Settings

The system has five predefined treatment settings.

These are user-configured values.

The Agent may read and explain them.

The Agent must never modify them autonomously.

### 4.1 Long Acting Insulin

`long_acting_insulin_units`

Represents the amount of long-acting insulin used by the user.

In Phase 1 this value is stored for future context only.

It must NOT participate in meal bolus or correction calculations.

### 4.2 Carb Ratio

`carb_ratio`

Defines how many grams of carbohydrates are covered by one unit of rapid-acting insulin.

Example:

```text
carb_ratio = 8
```

Means:

```text
1 unit per 8 grams of carbohydrates
```

Formula:

```text
meal_bolus = estimated_carbs / carb_ratio
```

### 4.3 Correction Factor

`correction_factor`

Defines how many mg/dL one unit of rapid-acting insulin is expected to lower glucose.

Example:

```text
correction_factor = 40
```

### 4.4 Maximum Correction Drop

`max_correction_drop`

Defines the maximum glucose reduction that may be used in a single correction calculation.

Example:

```text
max_correction_drop = 200
```

### 4.5 Correction Target

`correction_target`

Defines the glucose value that correction calculations should target.

Example:

```text
correction_target = 150
```

---

## 5. Local Settings Storage

For Phase 1, treatment settings may be stored locally.

Example:

```text
data/settings.json
```

Suggested structure:

```json
{
  "long_acting_insulin_units": 0,
  "carb_ratio": 0,
  "correction_factor": 0,
  "max_correction_drop": 0,
  "correction_target": 0
}
```

Do not hard-code personal treatment values inside application logic.

---

## 6. Dose Calculator

Implement a dedicated deterministic service:

```text
DoseCalculator
```

This component must contain no LLM logic.

### 6.1 Meal Bolus

```text
meal_bolus = estimated_carbs / carb_ratio
```

### 6.2 Correction

```text
required_drop = current_glucose - correction_target
```

If:

```text
required_drop <= 0
```

Then:

```text
correction_bolus = 0
```

Otherwise:

```text
allowed_drop = min(required_drop, max_correction_drop)
correction_bolus = allowed_drop / correction_factor
```

### 6.3 Total Dose

```text
total_calculated_dose = meal_bolus + correction_bolus
```

Return separately:

- meal_bolus
- correction_bolus
- total_calculated_dose

Do not silently invent rounding rules.

---

## 7. Dose Calculator Testing

Write automated tests before integrating the calculator into the Agent.

At minimum test:

- Normal meal bolus calculation.
- No correction required.
- Correction below maximum allowed drop.
- Correction above maximum allowed drop.
- Current glucose exactly equal to target.
- Invalid carb ratio.
- Invalid correction factor.
- Missing values.
- Negative or impossible values.
- Boundary conditions.

---

## 8. Reference Objects

Reference Objects are a required Phase 1 feature.

The user can register physical objects with known dimensions.

Examples:

- Fork
- Plate
- Card
- Box
- Spoon
- Personal container
- Any reusable physical object

These objects may later appear next to food in meal images and provide scale context.

---

## 9. Reference Object Registration Flow

Example:

```text
User:
"Add this fork as a reference"

[image]
```

Agent:

```text
"What is its length?"
```

User:

```text
"19 cm"
```

The system stores the Reference Object.

---

## 10. Reference Object Model

Suggested model:

```text
ReferenceObject
├── id
├── name
├── reference_image
├── measurement_type
├── width_cm
├── height_cm
├── diameter_cm
└── length_cm
```

Not every dimension must be populated.

Examples:

```text
name: fork
measurement_type: length
length_cm: 19
```

```text
name: plate
measurement_type: diameter
diameter_cm: 27
```

---

## 11. Reference Object Local Storage

Suggested structure:

```text
data/
├── settings.json
└── references/
    ├── references.json
    └── images/
        ├── reference_001.jpg
        ├── reference_002.jpg
        └── ...
```

Reference data must survive application restarts.

---

## 12. Reference Object Service

Implement a dedicated service:

```text
ReferenceObjectService
```

Suggested methods:

```text
add_reference()
list_references()
get_reference()
delete_reference()
match_reference()
get_dimensions()
```

---

## 13. Initial Meal Detection

When the user submits a meal image, first perform initial detection only.

Detect:

- Visible food items.
- Potential matches with saved Reference Objects.

Do NOT yet calculate final carbohydrate values.

Do NOT calculate insulin.

---

## 14. Mandatory User Confirmation Step

After initial detection, the Agent must tell the user what it detected and wait for confirmation or correction.

Example:

```text
I detected:
- chicken
- rice

I also detected your saved 19 cm fork as a reference.

Is that correct?
```

The user may respond:

```text
Yes
```

or:

```text
No, that is bulgur, not rice.
```

or:

```text
The chicken is correct, but that is not my saved fork.
```

The Agent must update the detected state accordingly.

### Hard Requirement

```text
NO FINAL CARB ESTIMATION OR DOSE CALCULATION
BEFORE USER CONFIRMATION
```

Specifically:

- Do not finalize carbohydrate estimation before food confirmation.
- Do not use a detected Reference Object for scale before user confirmation.
- Do not calculate meal bolus before confirmation.
- Do not calculate correction-based total dose before confirmation.

---

## 15. User Corrections

The user must be able to:

- Replace an incorrectly detected food.
- Add a missing food.
- Remove a falsely detected food.
- Reject a detected Reference Object.

Example:

```text
AI:
rice

User:
That is bulgur.
```

The confirmed state becomes:

```text
bulgur
```

---

## 16. Reference Matching

For Phase 1, reference matching may use the vision model itself.

The matching process compares:

- Saved reference images.
- Current meal image.

Suggested output:

```json
{
  "matched_reference": "fork",
  "confidence": "high"
}
```

Even a high-confidence match requires explicit user confirmation before use.

If no reliable match exists, return no match.

Do not invent a Reference Object match.

---

## 17. Confirmed Meal State

After user confirmation, maintain structured state containing:

```text
confirmed_foods
confirmed_reference
reference_dimensions
meal_image
current_glucose
```

---

## 18. Meal Vision / Portion Estimation

After user confirmation, run detailed meal analysis.

Inputs:

- Original meal image.
- Confirmed food list.
- Confirmed Reference Object, if any.
- Known dimensions of confirmed Reference Object.
- Prompt instructions.

The model must focus on portion estimation and carbohydrate estimation.

It must not calculate insulin.

---

## 19. Meal Analysis Output

Use structured output.

Suggested format:

```json
{
  "foods": [
    {
      "name": "rice",
      "estimated_portion": "approximately 1.5 cups",
      "estimated_carbs": 55,
      "min_carbs": 45,
      "max_carbs": 65,
      "confidence": "medium"
    }
  ],
  "total_estimated_carbs": 55,
  "total_min_carbs": 45,
  "total_max_carbs": 65,
  "overall_confidence": "medium"
}
```

---

## 20. Reference Object Scale Context

If a Reference Object is confirmed, include its known dimensions in the meal estimation context.

Example:

```text
Confirmed Reference Object:
fork

Known length:
19 cm
```

Use this information to improve estimates of:

- Plate size.
- Food footprint.
- Portion size.
- Relative dimensions.
- Approximate food volume.

The Reference Object improves context but does not create exact geometric measurement.

Account for uncertainty caused by:

- Perspective.
- Camera angle.
- Depth.
- Occlusion.
- Irregular food shape.
- Distance from camera.

Always prefer uncertainty ranges over false precision.

---

## 21. Carb Estimation Responsibility

The LLM is responsible for:

```text
image
→ food interpretation
→ portion estimation
→ carbohydrate estimation
```

The LLM is NOT responsible for:

```text
carbs
→ insulin dose
```

That belongs exclusively to DoseCalculator.

---

## 22. Main Meal Flow

```text
Meal Image
    ↓
Initial Food Detection
    ↓
Reference Detection
    ↓
Show Detection to User
    ↓
WAIT FOR CONFIRMATION
    ↓
Apply User Corrections
    ↓
Confirmed Foods + Confirmed Reference
    ↓
Detailed Meal Vision
    ↓
Carbohydrate Range
    ↓
Obtain Current Glucose
    ↓
Read Treatment Settings
    ↓
Dose Calculator
    ↓
Final Explanation
```

---

## 23. Agent Responsibilities

The Agent is responsible for conversation orchestration.

It should:

- Understand user intent.
- Distinguish between meal analysis and Reference Object registration.
- Identify missing information.
- Read treatment settings.
- Call food detection.
- Call Reference matching.
- Present detections to the user.
- Wait for confirmation.
- Apply user corrections.
- Call detailed meal estimation.
- Obtain current glucose if missing.
- Call DoseCalculator.
- Explain the result.
- Maintain short-term conversation state.

The Agent must not autonomously change treatment settings.

---

## 24. Agent Tools

Suggested tools:

```text
get_treatment_settings()

add_reference_object()

list_reference_objects()

get_reference_object()

delete_reference_object()

match_reference_object()

detect_foods()

analyze_confirmed_meal()

calculate_dose()
```

Each tool should have a clear, narrow responsibility.

---

## 25. Multi-Turn Conversation Example

```text
User:
[meal image]
```

Agent:

```text
I detected:
- chicken
- rice

I also detected your saved 19 cm fork.

Is that correct?
```

User:

```text
No. It is bulgur, not rice.
The fork is correct.
```

Agent:

```text
Updated:
- chicken
- bulgur

Confirmed reference:
19 cm fork.

What is your current glucose?
```

User:

```text
184
```

Only now may detailed carbohydrate estimation and dose calculation continue.

---

## 26. Missing Information

Do not invent critical missing information.

Request clarification when required.

Examples:

- Current glucose missing.
- Food identification not confirmed.
- Reference match not confirmed.
- New Reference Object has no dimensions.
- Image quality is too poor.
- Portion cannot be estimated reasonably.
- Food is highly ambiguous.

---

## 27. Uncertainty Handling

Do not present false precision.

Prefer:

```text
Estimated carbohydrate range:
60–90 g

Working estimate:
~75 g

Confidence:
Low
```

Instead of:

```text
73 g
```

when the image does not justify that precision.

If uncertainty is too high, ask for additional information such as:

- Another photo.
- Better angle.
- Reference Object.
- Package nutrition information.
- Known weight.
- Approximate serving size.

---

## 28. User Response Format

The final response should clearly separate:

1. Confirmed food identification.
2. Confirmed Reference Object.
3. AI carbohydrate estimation.
4. Deterministic insulin calculation.

Example:

```text
Confirmed meal:
- Chicken
- Rice
- Salad

Reference:
Your saved 19 cm fork was confirmed.

Estimated carbohydrates:
Rice: 45–60 g
Sauce: 8–12 g
Vegetables: 5–8 g

Estimated total:
58–80 g

Working estimate:
~69 g

Meal bolus:
69 / configured carb ratio = ...

Correction:
...

Total calculated dose:
...
```

---

## 29. Real Meal Testing

After the end-to-end flow works, build a small test dataset.

Target:

```text
20–30 meals
```

For each meal, record when possible:

- Meal image.
- Actual foods.
- Initial AI detection.
- User corrections.
- Known weight.
- Nutrition information.
- Estimated true carbohydrate value.
- Whether a Reference Object was present.
- Which Reference Object was used.
- Agent carbohydrate estimate.
- Confidence level.

---

## 30. Accuracy Evaluation

### Food Detection

Compare:

```text
Initial AI Detection
vs.
User Confirmed Food List
```

Track how often confirmation prevents incorrect downstream calculations.

### Carb Estimation

Compare known/estimated actual carbs against the Agent's estimated range.

### Reference Object Effectiveness

Test comparable meal images:

```text
Without Reference Object
vs.
With Reference Object
```

Evaluate whether Reference Objects materially improve portion and carbohydrate estimation.

If testing shows poor improvement, reconsider the approach before Phase 2.

---

## 31. Suggested Project Structure

```text
diabetes-agent/
├── app/
│   ├── main.py
│   ├── agent/
│   │   └── diabetes_agent.py
│   ├── services/
│   │   ├── meal_detection.py
│   │   ├── meal_vision.py
│   │   ├── dose_calculator.py
│   │   └── reference_service.py
│   ├── models/
│   │   ├── meal.py
│   │   ├── treatment_settings.py
│   │   └── reference_object.py
│   └── config.py
│
├── data/
│   ├── settings.json
│   └── references/
│       ├── references.json
│       └── images/
│
├── tests/
│   ├── test_dose_calculator.py
│   ├── test_reference_service.py
│   ├── test_meal_detection.py
│   └── ...
│
├── .env
├── requirements.txt
└── PROJECT.md
```

---

## 32. Phase 1 Exclusions

Do NOT implement during Phase 1:

- WhatsApp.
- PostgreSQL.
- User accounts.
- Libre.
- Dexcom.
- MCP.
- CGM history.
- Long-term injection history.
- Long-term meal history.
- Trend analysis.
- Automatic treatment adjustment.
- Automatic carb-ratio adjustment.
- Automatic correction-factor adjustment.
- Medical treatment recommendations based on historical patterns.
- Full Insulin On Board logic.

---

## 33. Phase 1 Definition of Done

Phase 1 is complete when both flows work locally.

### Flow A — Register Reference Object

```text
Reference Image
    ↓
User provides dimensions
    ↓
Reference Object created
    ↓
Reference persists locally
    ↓
Object can be listed and reused
```

### Flow B — Analyze Meal

```text
Meal Image
    ↓
Detect Foods
    ↓
Detect Saved Reference Object
    ↓
Show Detection to User
    ↓
WAIT FOR USER CONFIRMATION
    ↓
Apply Corrections
    ↓
Confirmed Foods / Reference
    ↓
Estimate Portion Sizes
    ↓
Estimate Carbohydrate Range
    ↓
Receive Current Glucose
    ↓
Read Treatment Settings
    ↓
Calculate Meal Bolus
    ↓
Calculate Allowed Correction
    ↓
Return Clear Final Response
```

The system must support several messages within the same meal conversation.

---

## 34. Future Phases

### Phase 2 — WhatsApp Integration

Move all user communication to WhatsApp.

The existing Agent and Core services should remain mostly unchanged.

### Phase 3 — Database and Tracking

Add persistent database storage for:

- Meals.
- Meal images.
- Estimated carbohydrates.
- Injection events.
- Suggested dose.
- Actual dose.
- Treatment Settings.
- Reference Objects.
- Conversation-linked meal events.

### Phase 4 — CGM Integration

Support:

- Libre
- Dexcom

through provider/adaptor interfaces.

Future capabilities:

- Current glucose retrieval.
- Glucose history.
- Post-meal glucose curves.
- Correlation between meals, insulin and CGM response.
- Historical trend analysis.

---

## 35. Non-Negotiable Rules

1. The LLM must not modify treatment settings.
2. The LLM must not directly calculate insulin doses.
3. All dose calculations must use deterministic Python functions.
4. Long-acting insulin is not part of the Phase 1 dose formulas.
5. Food identification must be confirmed by the user before final carb estimation.
6. Reference Object detection must be confirmed before it is used for scale.
7. No dose calculation may happen before food confirmation.
8. The system must clearly communicate uncertainty.
9. The system must never invent missing glucose values, treatment settings or dimensions.
10. Architecture should remain modular so WhatsApp, PostgreSQL and CGM providers can be added later without rewriting the Core.

---

## 36. Source of Truth

Treat this file as the source of truth for Phase 1.

Before making architectural or behavioral changes, verify that they are compatible with this document.

If implementation choices conflict with this document, prefer the behavior defined here unless the user explicitly changes the project requirements.
