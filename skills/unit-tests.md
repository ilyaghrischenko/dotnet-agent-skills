---
name: unit-tests
description: Generate unit tests for the selected C# code using xUnit and FluentAssertions, with auto-validation.
---

# Unit Tests Generator

## Overview
This skill generates comprehensive C# unit tests for the selected code, adhering strictly to xUnit and FluentAssertions conventions, and automatically verifies them.

## Rules & Requirements

### 1. Frameworks & Attributes
- **Testing:** `xUnit`
- **Assertion:** `FluentAssertions`
- **Parameterized Tests:** Extensively use `[Theory]` and `[InlineData]` to group similar test cases, reduce code duplication, and test multiple input combinations efficiently.

### 2. Coverage Exhaustiveness
- You MUST cover ALL meaningful success and failure scenarios, not just basic logical branches.
- Explicitly test boundary/edge values (e.g., minimum/maximum limits like 00:00, 23:59, zero, negative numbers, nulls, empty collections, empty strings).
- Test state transitions and boundary overlaps where applicable.

### 3. Code Style & Typing
- **Explicit Types:** Use EXPLICIT types only. DO NOT use `var`. Use `var` ONLY if the type is completely obvious after the `=` sign (e.g., `var list = new List<string>();`).
- **Modifiers:** Use `const` for local variables in the `// Arrange` section whenever possible (for compile-time constants).
- **Structure:** Every test method MUST explicitly include `// Arrange`, `// Act`, and `// Assert` comments.
- **Empty Strings:** You MUST use `string.Empty` instead of `""` everywhere inside the test method bodies (in Arrange, Act, and Assert sections). **CRITICAL EXCEPTION:** You MUST use `""` inside attributes like `[InlineData("")]`. Do NOT use `[InlineData(string.Empty)]`, as `string.Empty` is not a compile-time constant and will cause a C# compiler error.

### 4. Source Code Constraints
- **DO NOT fix bugs** in the source code. If the source code has a logic or syntax error, leave the source unchanged and add `// TODO: [Description of the error]` inside the affected test method.

## Workflow & Execution
1. **Plan Before Coding:** Analyze the target code and identify all necessary test cases (happy paths, edge cases, boundaries). Do not output the code yet. 
2. **Generate:** Write the complete test class following the rules above. Place the final test file in the matching test project directory.
3. **Execution Loop:** Run the generated tests using the CLI (e.g., `dotnet test`).
4. **Self-Correction:** If the tests fail, analyze the CLI error output, fix the test code, and run again. Limit yourself to a MAXIMUM of 3 correction attempts.
5. **Final Output:** If tests pass, confirm completion. If they still fail after 3 attempts, STOP. Do not generate more code. Output a concise explanation of the failure for manual intervention.
