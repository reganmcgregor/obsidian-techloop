---
notion_page_id: 3558c3d3-c03e-8145-8de5-daf02cd9e73a
notion_parent_id: 8588c3d3-c03e-823a-a182-81abcfdd2d6a
notion_teamspace: marketing
canonical: obsidian
department: marketing
type: context
synced_at: 2026-05-03T16:34:50Z
---

# AI Writing Guidelines

**Purpose:** This document defines quality standards for all AI-generated text across TechLoop's automation pipeline. Every LLM prompt in the system references these rules.
**Companion:** [[Tone of Voice]] — brand voice and copywriting framework.

---

## The Core Rule

**All AI-generated text must sound like a knowledgeable human wrote it.** If a reader can tell it was written by AI, it has failed. AI tells erode trust — the exact opposite of TechLoop's brand promise.

---

## 1. AI Tells to Eliminate

### Filler Openers (Delete on Sight)

These add zero information and scream "AI-generated":

| Bad | Fix |
|-----|-----|
| "In today's world of..." | Start with the actual point |
| "When it comes to..." | Just say the thing |
| "Whether you're a gamer or a professional..." | Cut entirely or be specific |
| "Looking for the perfect..." | State what the product does |
| "Are you ready to take your setup to the next level?" | No. Delete. |
| "In the ever-evolving landscape of..." | Absolutely not |

### Hedging Language (Be Direct)

| Bad | Good |
|-----|------|
| "This may help improve..." | "This improves..." |
| "It's worth noting that..." | Just note it |
| "You might find that..." | State the fact |
| "Could potentially offer..." | "Offers..." |
| "It should be mentioned..." | Mention it then |

### Formulaic Transitions (Vary Your Approach)

Kill these connectors — they make every paragraph sound identical:

- "Furthermore" → just start the next sentence
- "Additionally" → and / also / or just start fresh
- "Moreover" → cut it
- "That being said" → "But" or nothing
- "It is important to note" → just say it
- "On the other hand" → "But"
- "In conclusion" → don't announce your conclusion, just conclude

### Empty Superlatives (Prove It or Cut It)

| Bad | Good |
|-----|------|
| "Best-in-class performance" | State the benchmark |
| "Unparalleled quality" | What makes it quality? |
| "Next-level experience" | Describe the experience |
| "Premium build quality" | "Aluminium chassis, reinforced hinges" |
| "Seamlessly integrates" | "Works with X, Y, Z" |

---

## 2. Writing Style Rules

### Sentence Structure
- **Vary length.** Three short sentences. Then a longer one that develops the thought further. Then short again. Monotonous rhythm = AI tell.
- **Use contractions.** "It's", "you'll", "don't", "won't". Not "it is", "you will", "do not".
- **Start sentences differently.** Not every sentence should follow Subject-Verb-Object. Mix it up.
- **Cut ruthlessly.** If a sentence doesn't add new information, delete it. Every word earns its place.

### Tone (from [[Tone of Voice]])
- Knowledgeable mate, not a corporation
- Aussie, not ocker — "exy" is fine, "crikey" is not
- Opinionated where appropriate — "this pricing is cooked" is fine
- Australian English spelling throughout (colour, optimise, defence)

### Things That Sound Human
- Specific details instead of vague claims
- Occasional short fragments. Like this.
- Starting a sentence with "And" or "But"
- Acknowledging trade-offs honestly
- Using "you" and "your" naturally
- Referencing real use cases, not hypothetical ones

---

## 3. Specs Are Sacred

This rule is non-negotiable across all LLM prompts:

- **Feature claims use the manufacturer's exact language.** If Corsair says "8,000Hz hyper-polling", we write "8,000Hz hyper-polling".
- **Do not paraphrase technical specs** in ways that could change meaning. "DDR5-6000 CL30" stays as "DDR5-6000 CL30".
- **Do not "Aussiefy" specs.** Our tone of voice applies to opinions and framing, never to technical facts.
- **When in doubt, keep the original wording.** We do not lead our community astray.

---

## 4. Before & After Examples

### Product Description — Bad (AI Slop)

> In today's fast-paced world of gaming and productivity, having the right keyboard can make all the difference. The Corsair Galleon 100 SD is a cutting-edge mechanical keyboard that seamlessly integrates Stream Deck functionality. Whether you're a streamer, a gamer, or a professional, this keyboard may be exactly what you've been looking for. Furthermore, it features premium build quality and an unparalleled typing experience.

**Problems:** Filler opener, "cutting-edge", "seamlessly", hedging ("may be"), "furthermore", empty superlatives, tells the reader nothing specific.

### Product Description — Good (Human)

> The Galleon 100 SD is Corsair's first keyboard with a built-in Stream Deck — six customisable LCD keys right above the function row. It runs on 8,000Hz hyper-polling with FlashTap SOCD for competitive FPS, and the Stream Deck keys connect directly to the Elgato ecosystem for OBS, Twitch, and Voicemod control. Six layers of sound dampening keep it quiet without killing the tactile feel.

**Why it works:** Leads with what makes it different, uses manufacturer's exact spec language, no filler, every sentence adds information, reads like a person who actually knows keyboards wrote it.

### Short Description — Bad

> Experience the ultimate in high-performance computing with this exceptional laptop that delivers outstanding performance for both work and play.

### Short Description — Good

> 14-inch ultrabook with an Intel Core Ultra 7, 16GB DDR5, and a 512GB NVMe SSD. 1.4kg with all-day battery. Solid daily driver for dev work or light creative tasks.

---

## 5. Prompt Template

Every LLM prompt in the TechLoop pipeline should include a version of this block:

```
WRITING QUALITY:
- Sound like a knowledgeable human, not an AI
- No filler openers ("In today's world", "When it comes to", "Whether you're")
- No hedging ("may", "might", "could potentially", "it's worth noting")
- No formulaic transitions ("Furthermore", "Additionally", "Moreover")
- No corporate fluff ("cutting-edge", "industry-leading", "premium solutions", "seamlessly")
- Short, punchy sentences. Use contractions. Vary rhythm.
- Every sentence must add information — cut anything that doesn't
- Australian English spelling
```

---

## Tags

#guidelines #ai #writing #tone-of-voice
