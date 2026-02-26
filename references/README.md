# Code Review Checklists Reference

This directory contains comprehensive checklists covering different aspects of code quality and best practices. Use these checklists during code reviews to identify issues and maintain consistent standards.

## Available Checklists

### 1. **SOLID Checklist** (`solid-checklist.md`)
Architecture and design principles for maintainable code.

**Covers:**
- Single Responsibility Principle (SRP)
- Open/Closed Principle (OCP)
- Liskov Substitution Principle (LSP)
- Interface Segregation Principle (ISP)
- Dependency Inversion Principle (DIP)
- Common code smells (Long methods, Feature envy, Dead code, etc.)
- Refactor heuristics

### 2. **Code Quality Checklist** (`code-quality-checklist.md`)
General code quality patterns across all languages.

**Covers:**
- Error handling (swallowed exceptions, overly broad catch, async errors)
- Performance & caching (N+1 queries, expensive operations, memory issues)
- Boundary conditions (null/undefined, empty collections, numeric boundaries)
- String handling edge cases

### 3. **Security Checklist** (`security-checklist.md`)
Security vulnerabilities and reliability risks.

**Covers:**
- Input/output safety (XSS, injection, SSRF, path traversal)
- Authentication & authorization (missing checks, IDOR)
- JWT & token security
- Secrets and PII handling
- Supply chain & dependencies
- CORS & security headers
- Runtime risks (unbounded loops, race conditions, resource exhaustion)
- Cryptography best practices
- Data integrity

### 4. **TypeScript Checklist** (`typescript-checklist.md`)
TypeScript-specific type safety and best practices.

**Covers:**
- Type safety & declarations (`any` vs `unknown`, null handling)
- Functions & overloads (return types, parameter types)
- Generics & constraints (design, inference, conditional types)
- Interfaces & types (design patterns, mapped types)
- Classes & OOP (access modifiers, inheritance)
- Type aliases & unions (discriminated unions, utility types)
- Enums & constants (enum pitfalls, branded types)
- Type refinement & narrowing (type guards, assertions)
- Module & declaration (imports, circular dependencies)
- Advanced patterns (infer, variance, opaque types)
- Configuration & strictness (compiler options)
- Testing & type safety (runtime validation)
- Performance & bundle size

### 5. **React Native Checklist** (`react-native-checklist.md`)
React Native-specific patterns and pitfalls.

**Covers:**
- Lifecycle & memory management (useEffect cleanup, unmounted components)
- Performance & optimization (FlatList, memoization, threading)
- Platform-specific code (iOS/Android differences, safe area)
- Navigation (state issues, deep linking, listeners)
- Storage & persistence (AsyncStorage limits, encryption)
- Touch & user interactions (gesture handling, animations)
- Image handling (optimization, caching, errors)
- Common crashes and debug patterns

### 6. **Redux Checklist** (`redux-checklist.md`)
Redux state management best practices.

**Covers:**
- State shape & design (normalization, ownership, derived data)
- Reducers & purity (mutation bugs, initial state, deep updates)
- Actions & action creators (design, async actions, race conditions)
- Selectors & memoization (reselect usage, composition)
- Middleware & effects (thunks, middleware ordering)
- Redux DevTools & debugging
- Store configuration (Redux Toolkit, hydration)
- Common patterns (containers vs presentational, subscriptions)
- Redux with TypeScript (type safety, action payloads)

### 7. **Component Organization Checklist** (`component-organization-checklist.md`)
React/React Native component design and structure.

**Covers:**
- Component size & complexity (300+ line guidelines, multiple responsibilities)
- Component responsibilities (data fetching, business logic separation)
- JSX & render structure (readability, repeated patterns)
- Hook patterns & effects (effect organization, custom hooks)
- Code extraction strategies (composition patterns, file structure)
- Performance & re-renders (memoization, list optimization)
- Testing & maintainability (testability, clear props)
- Size guidelines (Small/Medium/Large/XLarge metrics)

### 8. **Native Bridge Checklist** (`native-bridge-checklist.md`)
Swift, Kotlin, and React Native bridge integration patterns.

**Covers:**
- Bridge architecture & design (error handling, event emitters)
- Swift-specific (memory management, thread safety, type safety, error handling)
- Kotlin-specific (memory management, thread safety, type safety, Android-specific)
- Bridge data serialization (type conversion, data size, performance)
- Platform-specific bridge issues (iOS/Android differences)
- Testing & validation (unit testing, integration testing)
- Common bridge crashes

### 9. **Removal Plan Template** (`removal-plan.md`)
Template for planning safe code removal and refactoring.

**Covers:**
- Priority levels (P0/P1/P2 removal decisions)
- Safe removal documentation
- Deferred removal with migration plans
- Breaking changes management
- Validation & verification steps
- Checklist before removal

## Usage Instructions

### For Code Reviewers

1. **Confirm Target Branch**: Ask which branch this PR is against
   - Could be: `main`, `develop`, `release/1.2.0`, `staging`, feature branch, hotfix, etc.
   - This is the baseline for git comparison
   - Different branches have different stability requirements
   - Use: `git diff origin/TARGET_BRANCH`

2. **Preview Changes**: Use `git diff` to scope what was changed
2. **Identify File Types**: Determine which checklists to load
3. **Load Relevant Checklists**: Based on file types:
   - Always load: `code-quality-checklist.md`, `security-checklist.md`, `solid-checklist.md`
   - TypeScript files: Add `typescript-checklist.md`
   - React/RN components: Add `component-organization-checklist.md`
   - React Native code: Add `react-native-checklist.md`
   - Redux code: Add `redux-checklist.md`
   - Native code (.swift, .kt): Add `native-bridge-checklist.md`
   - Removal work: Reference `removal-plan.md`
4. **Review Against Checklists**: Go through each relevant checklist
5. **Report Findings**: Use severity levels (P0-P3) as defined in SKILL.md

### For Code Authors

1. **Self-Review**: Run through relevant checklists before submitting PR
2. **Type-Based Selection**: Choose checklists matching your file changes
3. **Common Patterns**: Review the "Questions to Ask" sections in each checklist
4. **Testing**: Ensure your changes address the areas covered

## Quick Reference: What to Load by File Type

| File Type | Checklists to Load |
|-----------|-------------------|
| `.ts`, `.tsx` | Code Quality, TypeScript, Component Org (if component) |
| `.js`, `.jsx` | Code Quality, Component Org (if component) |
| `.tsx` (React Native) | Code Quality, TypeScript, Component Org, React Native |
| `.ts` (Redux) | Code Quality, TypeScript, Redux |
| `.swift` | Code Quality, Native Bridge |
| `.kt` | Code Quality, Native Bridge |
| Any file | Security, SOLID (architecture review) |

## Severity Levels

Use these levels when reporting issues (defined in SKILL.md):

- **P0 - Critical**: Security vulnerability, data loss risk, correctness bug → **Must block merge**
- **P1 - High**: Logic error, significant SOLID violation, performance regression → **Should fix before merge**
- **P2 - Medium**: Code smell, maintainability concern, minor SOLID violation → **Fix in this PR or create follow-up**
- **P3 - Low**: Style, naming, minor suggestion → **Optional improvement**

## Tips for Effective Reviews

1. **Read the questions**: Each checklist has "Questions to Ask" sections—use these!
2. **Look for patterns**: The same issues often appear across files
3. **Balance thoroughness with practicality**: Don't block on P3 issues if P0/P1 are clear
4. **Explain the why**: Reference the specific checklist item when commenting
5. **Suggest improvements**: Most checklists include code examples of good patterns
6. **Consider context**: Not all items apply to all projects—adapt to your team's standards

## Contributing to Checklists

When adding new items to checklists:

1. **Include examples**: Show both bad and good code patterns
2. **Ask questions**: End sections with "Questions to Ask" for reviewers
3. **Organize by concern**: Group related items together
4. **Keep it actionable**: Items should be checkable, not abstract
5. **Stay focused**: Each checklist should cover one domain well

---

**Last Updated**: February 2026
**Review Framework**: SKILL.md
