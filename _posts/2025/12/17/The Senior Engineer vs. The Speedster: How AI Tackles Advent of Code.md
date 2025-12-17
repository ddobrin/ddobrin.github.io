# The Senior Engineer vs. The Speedster: How AI Tackles Advent of Code

Every December, many developers from around the world tackle the [Advent of Code](https://adventofcode.com/) challenges — a much loved tradition of daily programming puzzles, increasingly complex, ranging from warm-up exercises to complex algorithmic challenges. 

This year, I decided to take a totally different approach to solving these puzzles: delegate to Gemini models and run an experiment that combines the creative puzzles of Advent of Code with the power of state-of-the-art LLMs. I was curious of the results, with different models tested. 

## TL;DR
**A Focused Goal**
- use state-of-the-art models: _**Gemini 3 Pro Preview**_ and _**Gemini 3 Flash Preview**_
- **employ only minimalistic** prompting
- give each model the same puzzle and dataset with a simple prompt, for all 12 days (different than in previous years with 2 puzzles/day now)
- validate that each model can solve it correctly in a **one-shot** approach, no hints, no intervention, for **all** 12 days
- compare not just *whether* the models solve it, but *how* they solve it, as a follow-up
- Gemini CLI is used for working with both models 

**Results**
- _**BOTH**_ models successfully solve _**ALL 24 puzzles**_ in one-shot, as I hoped they will !!
- the results reveal some interesting differences in what I'd call "engineering maturity"

**Note to the Reader**
- experimenting with all puzzles is left to the reader, with even more interesting solutions in the last few puzzles
- this is not an exhaustive, complex prompting type of exercise, it is just a snapshot of what can be done with LLMs today
- why this exercise? Imagine what you can do with this exercise with optimization, human-in-the-loop, etc !!

![alt text](image.png)

## Why Advent of Code Is Great for a Personal LLM Evaluation

Before looking into the exercise, let's touch briefly on two points.

### Advent of Code: Serious Challenges Packaged as Personal Developer Fun

Advent of Code occupies an interesting middle ground — it's both a fun exercise *and* a serious programming challenge. The problems genuinely test algorithmic thinking, especially in the later days of the schedule (last days usuaaly involve graph algorithms, dynamic programming, or computational geometry). 

Many professional developers use it for interview prep or to challenge themselves and stay sharp.

### Why It Works as an LLM Benchmark

Advent of Code is actually an excellent framework for comparing LLMs:

| Criterion | Why It Matters |
|-----------|----------------|
| **Objective correctness** | Problems have definitive right answers with test cases |
| **Graduated difficulty** | Days 1-6 focus more on basic reasoning; Days 7-12 reveal which models understand complex algorithms |
| **Logical focus** | focus on solving a problem in a logical way, no distraction from using frameworks or libraries in the code |
| **Flexibility** | Implement in the programming language of your choice |
| **Multi-dimensional evaluation** | You can assess correctness, code quality, performance, explanation clarity, and debugging ability all at once |
| **Real-world-like** | Unlike small demo challenges, Advent of Code requires parsing varied and messy input, handling edge cases, and sometimes optimizing for scale |
| **Reproducible** | Same problems, same inputs, easy to compare apples-to-apples across models |

---

## The Experiment Setup

Create an account on the Advent of Code website to unlock solution corectness evaluation and the subsequent exercises.

### The Models

| Model | Persona | Expected Behavior |
|-------|---------|-------------------|
| **Gemini 3 Pro Preview** | "The Senior Engineer" | Robust, maintainable, production-ready code. Proper error handling, modern APIs, clean architecture |
| **Gemini 3 Flash Preview** | "The Speedster" | Get the right answer *fast*. Correct logic, but less emphasis on "gold plating" or defensive coding |

### The Prompt

Both models received the same simple prompt, with file name changes only: the puzzle description and input, with a request to solve it in Java. No hints about code quality, no instructions about error handling - just solve the problem
```
Read carefully the following puzzle text from puzzle1.md and the associated input file puzzle1.input for testing; plan and implement the puzzle in Java 25 with a main() method for testing
``` 

### Evaluation Methodology

I used a hybrid approach, this is just a guideline - **remember** that we used minimalistic prompting and have not looked into optimizations!

1. **Automated correctness testing** - pass/fail against the actual puzzle input online at [Advent of Code](https://adventofcode.com/)
2. **Side-by-side analysis** - in an IDE
3. **Independent LLM evaluation** for qualitative aspects:
   - Code clarity and structure
   - Algorithmic adherence
   - Software engineering principles
   - Handling of edge cases
4. **Performance benchmarking** - execution time comparison   

## Let's Examine Puzzle 1 in Details: The Secret Entrance

The [first puzzle](https://adventofcode.com/2025/day/1) involves a combination safe with a dial numbered 0-99. Starting at position **50**, you receive a list of instructions to rotate the dial Left (L) or Right (R). The challenge: calculate the "password" by counting how many times the dial points to **0** after completing each rotation.

### The Algorithmic Challenge

This is a classic modular arithmetic problem with a twist: handling negative numbers correctly when rotating left past zero. In Java, the `%` operator can return negative values for negative operands, so the implementation must account for this.

**Example walkthrough:**
Starting at 50, applying rotations: L68, L30, R48, L5, R60, L55, L1, L99, R14, L82

| Rotation | Position After | Ends at 0? |
|----------|----------------|------------|
| L68 | 82 | No |
| L30 | 52 | No |
| R48 | **0** | Yes |
| L5 | 95 | No |
| R60 | 55 | No |
| L55 | **0** | Yes |
| L1 | 99 | No |
| L99 | **0** | Yes |
| R14 | 14 | No |
| L82 | 32 | No |

**Password: 3** (the dial pointed to 0 three times)

## Code Comparison: Where Engineering Maturity Shows

Both models solved the puzzle correctly. But *how* they solved it reveals the seniority angle.

### 1. The Core Logic: Handling Negative Modulo

**The Senior Engineer (Gemini 3 Pro)** used an explicit check for negative numbers:

```java
// Senior approach - explicit and readable
if (direction == 'L') {
    currentPos = (currentPos - amount) % 100;
    if (currentPos < 0) {
        currentPos += 100;
    }
} else if (direction == 'R') {
    currentPos = (currentPos + amount) % 100;
} else {
    throw new IllegalArgumentException("Unknown direction: " + direction);
}
```

**The Speedster (Gemini 3 Flash)** used a mathematical one-liner:

```java
// Speedy approach - clever but redundant
if (direction == 'R') {
    currentPos = (currentPos + distance) % 100;
} else if (direction == 'L') {
    currentPos = (currentPos - (distance % 100) + 100) % 100;
}
```

**Analysis:** 
* Both are correct, but Speedster's version includes a redundant `distance % 100` operation—the outer modulo already handles wrap-around. 
* The Senior's code is also easier to debug: the logic is explicit and the negative case is clearly handled.

### 2. API Design: Flexibility vs. Hardcoding

| Aspect | Senior (Pro 3) | Speedster (Flash 3) |
|--------|--------------|-------------------|
| Method signature | `solve(List<String>, int startPos)` | `solve(List<String>)` |
| Start position | Parameterized (configurable) | Hardcoded (`int currentPos = 50;`) |
| Return type | `long` (prevents overflow) | `int` |

**Winner: Senior Engineer**

By parameterizing `startPos`, the Senior's code is immediately testable with different scenarios without modifying any source code. This is a hallmark of production-ready design. The `long` return type also shows foresight about potential overflow with large inputs.

### 3. Error Handling: Fail-Fast vs. Silent

| Behavior | Senior (Pro) | Speedster (Flash) |
|----------|--------------|-------------------|
| Unknown direction | `throw IllegalArgumentException` | Silently ignored |
| File not found | Explicit check with clear message | Generic IOException |
| IOException handling | `throws IOException` (caller decides) | try-catch (handled internally) |

**Winner: Senior Engineer**

* The Senior implements "Fail Fast" logic. If the input contains an unknown direction (e.g., 'U'), it throws an exception immediately. 
* The Speedster silently ignores invalid lines — which might prevent a crash, but could lead to incorrect results with no warning. This is a classic debugging issue in production.

### 4. Modern Java Idioms

| Aspect | Senior (Pro 3) | Speedster (Flash 3) |
|--------|--------------|-------------------|
| Path API | `Path.of()` (Java 11+) | `Paths.get()` (Java 7+) |
| File existence check | `Files.exists()` | None |

**Winner: Senior Engineer**

* `Path.of()` is the modern, preferred API. Using it signals awareness of current Java best practices.

### 5. Test code
**BOTH** Pro and Flash models generated almost identical test code, based on the outline of Puzzle 1
```
class Puzzle1Test {

    @Test
    void testExample() {
        List<String> instructions = List.of(
            "L68",
            "L30",
            "R48",
            "L5",
            "R60",
            "L55",
            "L1",
            "L99",
            "R14",
            "L82"
        );
        
        long result = Puzzle1.solve(instructions, 50);
        assertEquals(3, result);
    }
}
```

### 6. Performance Analysis

| Metric | Senior (Pro) | Speedster (Flash) |
|--------|--------------|-------------------|
| Time complexity | O(N) | O(N) |
| Operations per rotation | 1 modulo + 1 conditional | 2-3 modulo operations |

**Winner: Senior Engineer (marginally)**

* The extra `distance % 100` in Flash's implementation is unnecessary work. 
* Both are O(N) and the difference is negligible in practice, but the Senior's code is technically more efficient.

## The Scorecard

| Criterion | Senior (Pro 3) | Speedster (Flash 3) |
|-----------|--------------|-------------------|
| **Correctness** | Pass | Pass |
| **API Design** | **Winner** (configurable) | Basic (hardcoded) |
| **Error Handling** | **Winner** (fail-fast) | Silent/permissive |
| **Input Robustness** | Good | Good |
| **Modern Java** | **Winner** (`Path.of`) | Older (`Paths.get`) |
| **Performance** | **Winner** (marginal) | Good |

## Key Insights: What Have We Learned

### The Speedster as a Developer Under Deadline

Gemini 3 Flash acted like a developer racing against a deadline: it got the right answer efficiently, but left behind technical debt. Hardcoded values, older APIs, silent error handling—these are the hallmarks of "I'll fix it later" code that often takes time or never gets fixed.

### The Senior as a Production-Minded Engineer

Gemini 3 Pro solved the problem *and* built a reusable, robust component. The use of dependency injection (passing `startPos` as a parameter), explicit error handling, and modern APIs demonstrates a deeper understanding of software craftsmanship.

### The Adoption Decision

**For a quick script or prototype?** Flash is perfectly fine. It gets the job done.

**For the production codebase?** Adopt the Senior's solution. The generated code is:
- Testable without modification
- Self-documenting through explicit error handling
- Future-proof with modern idioms
- Easier to debug when things go wrong

## Conclusion

For Puzzle 1 of Advent of Code 2025, the distinction between the Senior Engineer and the Speedster was clear:

- **Gemini Pro** delivered a working solution *and* a production-ready component
- **Gemini Flash** delivered a working solution with minimal fuss, but left technical debt

The metaphor extends beyond this experiment. When using AI for code generation, the model you choose—and the prompts you craft—can produce either throwaway prototypes or building blocks for serious software.

**For quick scripts? The Speedster is fine. For the production codebase? Start by adopting the Senior's solutions.**

