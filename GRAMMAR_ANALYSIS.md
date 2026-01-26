# StepCode ANTLR4 Grammar Analysis

This document provides a comprehensive analysis of the `StepCode.g4` grammar file, identifying problems, inconsistencies, and potential enhancements.

## Table of Contents

1. [Critical Issues](#critical-issues)
2. [Moderate Issues](#moderate-issues)
3. [Minor Issues](#minor-issues)
4. [Bilingual Keyword Inconsistencies](#bilingual-keyword-inconsistencies)
5. [Unused/Dead Rules](#unuseddead-rules)
6. [Enhancement Opportunities](#enhancement-opportunities)
7. [Best Practice Recommendations](#best-practice-recommendations)

---

## Critical Issues

### 1. Left Recursion Pattern Issues

The grammar uses direct left recursion in several expression rules:

```antlr
expression
   : booleanMultiplicativeExpression | expression OR expression
   ;

booleanMultiplicativeExpression
    : booleanRelationalExpression | booleanMultiplicativeExpression AND booleanMultiplicativeExpression
    ;

booleanRelationalExpression
    : simpleExpression | booleanRelationalExpression relationaloperator booleanRelationalExpression
    ;

simpleExpression
   : term | simpleExpression additiveoperator simpleExpression
   ;

term
   : baseTerm | term multiplicativeoperator term
   ;

baseTerm
   : signedFactor | baseTerm exponentiationOperator baseTerm
   ;
```

**Problem:** While ANTLR4 handles direct left recursion, these rules allow **both** left and right recursion simultaneously (`A : B | A op A`). This creates ambiguity about associativity.

**Impact:**
- `3 - 2 - 1` could be parsed as `(3 - 2) - 1 = 0` or `3 - (2 - 1) = 2`
- Operator associativity is undefined

**Recommended Fix:**
```antlr
// Use only left recursion for left-associative operators
expression
   : booleanMultiplicativeExpression
   | expression OR booleanMultiplicativeExpression
   ;

// Use only right recursion for right-associative operators (like exponentiation)
baseTerm
   : signedFactor
   | signedFactor exponentiationOperator baseTerm  // Right-associative
   ;
```

### 2. Missing `ENDIF` in `elifStatement`

```antlr
elifStatement
   : ELIF expression THEN compoundStatement (elifStatement | elseStatement? | ENDIF)
   ;
```

**Problem:** The rule `(elifStatement | elseStatement? | ENDIF)` allows `ENDIF` as an alternative, but when `elifStatement` is called from `ifStatement`, the parent rule also expects `ENDIF`:

```antlr
ifStatement
   : IF expression THEN compoundStatement (elifStatement | elseStatement?) ENDIF
   ;
```

This creates ambiguity: if there's an `ELIF` followed by `ENDIF`, where does the `ENDIF` belong?

**Recommended Fix:**
```antlr
elifStatement
   : ELIF expression THEN compoundStatement (elifStatement | elseStatement)?
   ;

ifStatement
   : IF expression THEN compoundStatement (elifStatement | elseStatement)? ENDIF
   ;
```

### 3. Ambiguous `procedureStatement` Rule

```antlr
procedureStatement
   : identifier (LPAREN parameterList? RPAREN)
   ;
```

**Problem:** The parentheses are in a group but not optional. This means `foo()` is valid but `foo` alone is not valid as a procedure statement. However, `functionDesignator` also matches the same pattern:

```antlr
functionDesignator
   : identifier LPAREN parameterList? RPAREN
   ;
```

**Impact:** Both rules can match `foo()`, causing parsing ambiguity in `factor`.

**Recommended Fix:** Make `procedureStatement` explicitly require parentheses or make them optional:
```antlr
procedureStatement
   : identifier LPAREN parameterList? RPAREN
   ;
```

---

## Moderate Issues

### 4. `NUM_REAL` Matches Integers

```antlr
NUM_REAL
   : ('0' .. '9') + (('.' ('0' .. '9') + (EXPONENT)?)? | EXPONENT)
   ;
```

**Problem:** Due to the nested optional groups, `NUM_REAL` will match integer-only patterns like `123` (when the whole optional part is skipped).

**Impact:** Lexer ambiguity between `NUM_INT` and `NUM_REAL`. ANTLR will use the first matching rule, but this can cause unexpected behavior.

**Recommended Fix:**
```antlr
NUM_REAL
   : ('0' .. '9')+ '.' ('0' .. '9')+ EXPONENT?
   | ('0' .. '9')+ EXPONENT
   ;
```

### 5. Statements Semicolon Handling

```antlr
statements
   : statement (SEMI statement)* (SEMI)?
   ;

assignmentStatement
   : variable ASSIGN expression SEMI
   ;
```

**Problem:** Individual statements like `assignmentStatement` already include `SEMI`, but `statements` uses `SEMI` as a separator. This creates issues:
- `a := 1; b := 2;` would parse as: `assignmentStatement(a:=1;)` SEMI `assignmentStatement(b:=2;)` - double semicolons

**Impact:** Inconsistent semicolon handling across the grammar.

**Recommended Fix:** Either:
1. Remove `SEMI` from individual statements and use `statements` for termination, or
2. Remove semicolon separators from `statements`

### 6. Missing Error Recovery Tokens

The grammar lacks `SYNC` tokens or error alternatives for better error recovery.

**Recommended Addition:**
```antlr
statement
   : unlabelledStatement
   | writeStatement | readStatement
   | breakStatement | continueStatement | returnStatement
   | error  // Add error recovery
   ;
```

### 7. Identifier Cannot Start with Underscore

```antlr
IDENT
   : ('A' .. 'Z') ('A' .. 'Z' | '0' .. '9' | '_' | '@')*
   ;
```

**Problem:** Identifiers must start with a letter but allow `_` in subsequent positions. Many programming languages allow `_identifier` for private/internal naming.

**Recommended Fix:**
```antlr
IDENT
   : ('A' .. 'Z' | '_') ('A' .. 'Z' | '0' .. '9' | '_' | '@')*
   ;
```

---

## Minor Issues

### 8. Redundant `empty_` Rule

```antlr
emptyStatement_
   :
   ;

empty_
   :
   /* empty */
   ;
```

**Problem:** Two empty rules that serve no purpose and are never referenced in the grammar.

**Recommended Fix:** Remove both rules.

### 9. Inconsistent Bracket Alternatives

```antlr
arrayType
   : ARRAY LBRACK typeList RBRACK OF componentType
   | ARRAY LBRACK2 typeList RBRACK2 OF componentType
   ;

set_
   : LBRACK elementList RBRACK
   | LBRACK2 elementList RBRACK2
   ;
```

**Problem:** `LBRACK2`/`RBRACK2` (`(.` and `.)`) is an obsolete Pascal notation. Modern educational code shouldn't need this.

**Recommendation:** Consider deprecating `LBRACK2`/`RBRACK2` unless specifically needed for legacy code compatibility.

### 10. Block Rule Complexity

```antlr
block
   : (labelDeclarationPart | constantDefinitionPart | typeDefinitionPart | variableDeclarationPart | dimensionStatement |
   procedureAndFunctionDeclarationPart | usesUnitsPart | IMPLEMENTATION | statements)*
   ;
```

**Problem:** This rule allows any ordering of declarations and statements. While flexible, it's unusual for educational pseudo-code to allow mixed declarations and statements.

**Recommendation:** Consider enforcing declaration-before-statement ordering for better educational value:
```antlr
block
   : declarationPart? statementPart?
   ;

declarationPart
   : (variableDeclarationPart | dimensionStatement | constantDefinitionPart)*
   ;

statementPart
   : statements
   ;
```

---

## Bilingual Keyword Inconsistencies

### Missing Spanish Alternatives

| Token | English | Spanish | Issue |
|-------|---------|---------|-------|
| `DOWNTO` | `DOWNTO` | - | Missing Spanish: `DECRECIENDO` or `HASTA ABAJO` |
| `DO` | `DO` | - | Missing Spanish (use `HACER` consistently) |
| `GOTO` | `GOTO` | - | Missing Spanish: `IR A` |
| `LABEL` | `LABEL` | - | Missing Spanish: `ETIQUETA` |
| `CONST` | `CONST` | - | Missing Spanish: `CONSTANTE` |
| `TYPE` | `TYPE` | - | Missing Spanish: `TIPO` |
| `RECORD` | `RECORD` | - | Missing Spanish: `REGISTRO` |
| `FILE` | `FILE` | - | Missing Spanish: `ARCHIVO` |
| `SET` | `SET` | - | Missing Spanish: `CONJUNTO` |
| `ARRAY` | `ARRAY` | - | Missing Spanish: `ARREGLO` |
| `NIL` | `NIL` | - | Missing Spanish: `NULO` |
| `IN` | `IN` | - | Missing Spanish: `EN` |
| `OF` | `OF` | - | Missing Spanish: `DE` |
| `WITH` | `WITH` | - | Missing Spanish: `CON` |
| `PACKED` | `PACKED` | - | Missing Spanish: `EMPAQUETADO` |
| `USES` | `USES` | - | Missing Spanish: `USA` or `IMPORTAR` |
| `UNIT` | `UNIT` | - | Missing Spanish: `UNIDAD` |
| `INTERFACE` | `INTERFACE` | - | Missing Spanish: `INTERFAZ` |
| `IMPLEMENTATION` | `IMPLEMENTATION` | - | Missing Spanish: `IMPLEMENTACION` |
| `DIV` | `DIV` | - | Could add: `DIVENTERO` |
| `MOD` | `MOD` | - | Could add: `MODULO` or `RESTO` |
| `CHR` | `CHR` | - | Could add: `CARACTER` (as function) |

### Inconsistent Spanish Keyword Variations

| Token | Current | Missing Variation |
|-------|---------|-------------------|
| `FUNCTION` | `FUNCION` | Missing accent: `FUNCIÓN` |
| `ELIF` | `SINO SI` | Could also support: `SINO SI` (with space handling) |
| `ENDFUNCTION` | `FINFUNCION` | Missing: `FINFUNCIÓN` |

---

## Unused/Dead Rules

The following rules are defined but never referenced or used in the interpreter:

1. **`gotoStatement`** - Defined but GOTO is generally discouraged in educational contexts
2. **`labelDeclarationPart`** and **`label`** - Only useful with GOTO
3. **`withStatement`** and **`recordVariableList`** - Records not implemented in interpreter
4. **`recordType`**, **`fieldList`**, **`fixedPart`**, **`recordSection`**, **`variantPart`**, **`variant`**, **`tag`** - Record types not implemented
5. **`setType`** and **`baseType`** - Set types not implemented
6. **`fileType`** - File operations not implemented
7. **`pointerType`** - Pointers not implemented
8. **`scalarType`** - Enum-like types not implemented
9. **`subrangeType`** - Subrange types not implemented
10. **`stringtype`** (with size) - Fixed-size strings not implemented
11. **`usesUnitsPart`** - Module system not implemented
12. **`constantDefinitionPart`** and **`constantDefinition`** - Constants not implemented
13. **`typeDefinitionPart`** and **`typeDefinition`** - Custom types not implemented
14. **`functionType`** and **`procedureType`** - Function types not implemented
15. **`emptyStatement_`** and **`empty_`** - Never used
16. **`parameterwidth`** - Format specifiers not implemented

**Recommendation:** Either implement these features or remove the unused rules to simplify the grammar.

---

## Enhancement Opportunities

### 1. Add Ternary/Conditional Expression

```antlr
// New rule for ternary operator
conditionalExpression
   : expression QUESTION expression COLON expression
   ;

// Or Spanish-friendly version
conditionalExpression
   : SI expression ENTONCES expression SINO expression
   ;
```

### 2. Add Increment/Decrement Operators

```antlr
INCREMENT : '++' ;
DECREMENT : '--' ;

postfixExpression
   : variable INCREMENT
   | variable DECREMENT
   ;

prefixExpression
   : INCREMENT variable
   | DECREMENT variable
   ;
```

### 3. Add Compound Assignment Operators

```antlr
PLUS_ASSIGN : '+=' ;
MINUS_ASSIGN : '-=' ;
STAR_ASSIGN : '*=' ;
SLASH_ASSIGN : '/=' ;

compoundAssignment
   : variable (PLUS_ASSIGN | MINUS_ASSIGN | STAR_ASSIGN | SLASH_ASSIGN) expression SEMI
   ;
```

### 4. Add String Interpolation

```antlr
INTERPOLATED_STRING
   : '`' (~[`\\] | '\\' . | '${' .*? '}')* '`'
   ;
```

### 5. Add Multi-line Comments

```antlr
BLOCK_COMMENT
   : '/*' .*? '*/' -> skip
   ;

// Or Spanish-style
BLOCK_COMMENT_ES
   : '(*' .*? '*)' -> skip
   ;
```

### 6. Add Range Expressions for For Loops

```antlr
rangeExpression
   : expression DOTDOT expression
   ;

forStatement
   : FOR identifier IN rangeExpression (DO | HACER) compoundStatement ENDFOR
   ;
```

### 7. Add Array Literals

```antlr
arrayLiteral
   : LBRACK (expression (COMMA expression)*)? RBRACK
   ;
```

### 8. Add Switch Expression with Ranges

```antlr
caseRange
   : constant DOTDOT constant
   ;

caseListElement
   : (constList | caseRange) (COLON | AS) compoundStatement
   ;
```

### 9. Add Named Parameters

```antlr
namedParameter
   : identifier ASSIGN expression
   ;

parameterList
   : (actualParameter | namedParameter) (COMMA (actualParameter | namedParameter))*
   ;
```

### 10. Add Lambda/Anonymous Functions

```antlr
lambdaExpression
   : LPAREN identifierList? RPAREN ARROW expression
   | LPAREN identifierList? RPAREN ARROW LBRACE statements RBRACE
   ;

ARROW : '=>' | '->' ;
```

---

## Best Practice Recommendations

### 1. Separate Lexer and Parser Grammars

For larger grammars, consider splitting into separate files:
- `StepCodeLexer.g4` - All token definitions
- `StepCodeParser.g4` - All parser rules

### 2. Add Grammar Documentation

Add doc comments to rules:
```antlr
/**
 * Main program structure
 * @example Proceso MiPrograma ... FinProceso
 */
program
   : directives* subprogram* main subprogram* EOF
   ;
```

### 3. Use Labels for Better AST Generation

```antlr
expression
   : booleanMultiplicativeExpression                    # primaryExpr
   | left=expression OR right=expression                # orExpr
   ;
```

### 4. Add Semantic Predicates for Context-Sensitive Parsing

```antlr
// Example: Distinguish between function call and array access
factor
   : {isFunction(getCurrentToken().getText())}? functionDesignator
   | variable
   ;
```

### 5. Improve Error Messages

Add custom error listeners with meaningful messages:
```antlr
// In parser
@members {
    public void notifyErrorListeners(Token offendingToken, String msg, RecognitionException e) {
        // Custom error handling
    }
}
```

### 6. Add Token Channels for Comments

Instead of skipping comments, preserve them:
```antlr
COMMENT_1
   : '//' ~[\r\n]* -> channel(COMMENTS)
   ;
```

This allows tools to access comments for documentation generation.

---

## Summary

### Priority Matrix

| Priority | Issue | Impact |
|----------|-------|--------|
| **High** | Operator associativity ambiguity | Incorrect calculations |
| **High** | `NUM_REAL` matching integers | Lexer confusion |
| **Medium** | Semicolon inconsistency | Parser errors |
| **Medium** | Missing Spanish keywords | User experience |
| **Low** | Unused rules | Code bloat |
| **Low** | Missing features | Functionality gaps |

### Recommended Action Plan

1. **Immediate:** Fix operator associativity in expression rules
2. **Short-term:** Fix `NUM_REAL` lexer rule and semicolon handling
3. **Medium-term:** Complete Spanish keyword coverage
4. **Long-term:** Remove unused rules or implement missing features
5. **Optional:** Add enhancement features based on user needs

---

*Analysis generated for StepCode v0.12.0*
