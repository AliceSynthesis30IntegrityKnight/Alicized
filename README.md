# Alicized
import re

# ---------------- the rules ----------------
VALUES = [-5, -4, -3, -2, -1, 1, 2, 3, 4, 5]          # ranks 1..10 -> -5..+5, no zero

def rank_to_value(rank):
    return VALUES[rank - 1]

def rerank(raw, biases):
    """Forced spread: width 10, so every rank 1..10 is used exactly once. Ties broken by the idea's own bias."""
    order = sorted(range(len(raw)), key=lambda i: (raw[i], biases[i]))
    values = [0] * len(raw)
    for position, i in enumerate(order):
        values[i] = VALUES[position]
    return values

WIRES_PER_LINK = 3                                     # parallel wires: the path count is the multiplier

def layer(prev_values, prev_active, biases, links):
    """Each idea = its own bias + every link from an active idea in the previous layer (+ adds, - subtracts).
    Skip at zero: an idea that received nothing stays inactive and passes nothing on.
    Then the whole layer is re-ranked with the forced spread."""
    raw, active = list(biases), [False] * len(biases)
    for src, dst, sign in links:
        if prev_active[src]:
            raw[dst] += sign * prev_values[src] * WIRES_PER_LINK
            active[dst] = True
    return rerank(raw, biases), active

# ---------------- layer 1: 10 clue ideas (fire when their words appear) ----------------
CLUES = ["greet", "thanks", "bad", "good", "money", "ask", "bye", "time", "stop", "book"]
CLUE_WORDS = [r"\b(hello|hi|hey)\b", r"\b(thanks?|thank you|cheers)\b", r"\b(broken|bad|terrible|awful|worst|late)\b",
              r"\b(great|love|awesome|excellent|amazing)\b", r"\b(refund|money|charged?|price|pay)\b",
              r"\?|\b(how|what|when|why|where)\b", r"\b(bye|goodbye|see you)\b", r"\b(today|tomorrow|tonight|friday|time)\b",
              r"\b(cancel|stop|unsubscribe|quit)\b", r"\b(book|reserve|table|ticket)\b"]
CLUE_RANKS = [3, 6, 8, 7, 9, 4, 2, 5, 10, 1]          # each rank used once

# ---------------- layer 2: 10 concept ideas ----------------
CONCEPTS = ["polite", "unhappy", "happy", "transaction", "asking", "leaving", "scheduling", "problem", "opening", "ending"]
CONCEPT_RANKS = [4, 9, 6, 8, 5, 2, 7, 10, 3, 1]
L1_TO_L2 = [  # (from clue, to concept, +1 adds / -1 subtracts)
    (0, 8, +1), (0, 0, +1), (1, 0, +1), (1, 9, +1), (1, 1, -1),
    (2, 1, +1), (2, 7, +1), (2, 2, -1), (3, 2, +1), (3, 1, -1), (3, 7, -1),
    (4, 3, +1), (4, 7, +1), (5, 4, +1), (6, 5, +1), (6, 9, +1),
    (7, 6, +1), (8, 5, +1), (8, 3, +1), (9, 6, +1), (9, 3, +1),
]

# ---------------- layer 3: 10 output ideas (the classes) ----------------
CLASSES = ["greeting", "thanks", "complaint", "praise", "refund", "question", "goodbye", "booking", "cancellation", "other"]
CLASS_RANKS = [8, 5, 10, 6, 9, 4, 3, 7, 2, 1]
L2_TO_L3 = [  # (from concept, to class, +1 adds / -1 subtracts)
    (8, 0, +1), (0, 0, +1),                      # greeting  <- opening, polite
    (0, 1, +1), (9, 1, +1), (1, 1, -1),          # thanks    <- polite, ending; not unhappy
    (1, 2, +1), (7, 2, +1), (2, 2, -1),          # complaint <- unhappy, problem; not happy
    (2, 3, +1), (1, 3, -1),                      # praise    <- happy; not unhappy
    (3, 4, +1), (7, 4, +1), (2, 4, -1),          # refund    <- transaction, problem; not happy
    (4, 5, +1), (6, 5, +1),                      # question  <- asking, scheduling
    (5, 6, +1), (9, 6, +1),                      # goodbye   <- leaving, ending
    (6, 7, +1), (3, 7, +1), (7, 7, -1),          # booking   <- scheduling, transaction; not problem
    (5, 8, +1), (3, 8, +1), (6, 8, -1),          # cancel    <- leaving, transaction; not scheduling
    (4, 9, -1), (0, 9, -1),                      # other     <- pushed down by any real signal
]

def classify(text):
    t = text.lower()
    a1 = [bool(re.search(p, t)) for p in CLUE_WORDS]                   # input: which clues appear
    b1 = [rank_to_value(r) for r in CLUE_RANKS]
    l1 = rerank([b + (10 if a else 0) for b, a in zip(b1, a1)], b1)    # layer 1
    l2, a2 = layer(l1, a1, [rank_to_value(r) for r in CONCEPT_RANKS], L1_TO_L2)   # layer 2
    l3, a3 = layer(l2, a2, [rank_to_value(r) for r in CLASS_RANKS], L2_TO_L3)     # layer 3
    live = [i for i in range(10) if a3[i]]
    best = max(live, key=lambda i: l3[i]) if live else CLASSES.index("other")     # highest-ranked active class
    return CLASSES[best], {CLASSES[i]: l3[i] for i in live}

if __name__ == "__main__":
    for text in ["Hello there!", "Thanks so much, that was great", "My order arrived broken and late, terrible",
                 "I want a refund, you charged me twice", "Can I book a table for Friday?", "What time do you open?",
                 "Please cancel my subscription", "This is amazing, I love it", "Goodbye, see you tomorrow"]:
        label, ranks = classify(text)
        print(f"{text:<46} -> {label:<13} {ranks}")
