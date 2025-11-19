# HyperFormula Parser API

This document describes the exposed parser utilities and AST types that allow you to parse and work with formulas without instantiating a full HyperFormula instance.

## Overview

HyperFormula now exports its internal parser classes and AST types, allowing you to:
1. Parse formulas to Abstract Syntax Trees (AST)
2. Inspect formula structure and dependencies
3. Transform and manipulate formulas programmatically
4. Build custom formula evaluation logic

## Key Components

### 1. ParserWithCaching

The main parser class that converts formula strings to AST with built-in caching.

```typescript
import { ParserWithCaching, buildLexerConfig, SimpleCellAddress } from 'hyperformula';

// Create parser instance
const config = /* your config */;
const functionRegistry = /* your function registry */;
const sheetMapping = (sheetName: string) => 0; // Simple mapping

const parser = new ParserWithCaching(
  config,
  functionRegistry,
  sheetMapping
);

// Parse a formula
const formulaAddress: SimpleCellAddress = { sheet: 0, row: 0, col: 0 };
const result = parser.parse('=SUM(A1:A10)', formulaAddress);

// Access the AST
console.log(result.ast);
console.log(result.errors);
console.log(result.dependencies);
```

### 2. AST Types

All AST node types are exported:

#### Basic Types
- `NumberAst` - Number literals (e.g., `42`, `3.14`)
- `StringAst` - String literals (e.g., `"Hello"`)
- `EmptyArgAst` - Empty arguments

#### Operations
- **Arithmetic**: `PlusOpAst`, `MinusOpAst`, `TimesOpAst`, `DivOpAst`, `PowerOpAst`
- **Comparison**: `EqualsOpAst`, `NotEqualOpAst`, `GreaterThanOpAst`, `LessThanOpAst`, `GreaterThanOrEqualOpAst`, `LessThanOrEqualOpAst`
- **Unary**: `MinusUnaryOpAst`, `PlusUnaryOpAst`, `PercentOpAst`
- **String**: `ConcatenateOpAst`

#### References
- `CellReferenceAst` - Single cell reference (e.g., `A1`, `$A$1`)
- `CellRangeAst` - Cell range (e.g., `A1:B10`)
- `ColumnRangeAst` - Column range (e.g., `A:B`)
- `RowRangeAst` - Row range (e.g., `1:10`)

#### Functions and Expressions
- `ProcedureAst` - Function calls (e.g., `SUM(A1:A10)`)
- `NamedExpressionAst` - Named expressions
- `ParenthesisAst` - Parenthesized expressions
- `ArrayAst` - Array literals (e.g., `{1,2;3,4}`)

#### Error Types
- `ErrorAst` - Error values (e.g., `#REF!`, `#DIV/0!`)
- `ErrorWithRawInputAst` - Errors with original input

### 3. AST Node Type Enum

```typescript
import { AstNodeType } from 'hyperformula';

// Check node types
if (ast.type === AstNodeType.FUNCTION_CALL) {
  // It's a function call
}
```

Available types:
- `EMPTY`, `NUMBER`, `STRING`
- `PLUS_OP`, `MINUS_OP`, `TIMES_OP`, `DIV_OP`, `POWER_OP`
- `EQUALS_OP`, `NOT_EQUAL_OP`, `GREATER_THAN_OP`, `LESS_THAN_OP`, etc.
- `FUNCTION_CALL`, `CELL_REFERENCE`, `CELL_RANGE`, `NAMED_EXPRESSION`
- `PARENTHESIS`, `ARRAY`, `ERROR`

### 4. AST Builder Functions

Build AST nodes programmatically:

```typescript
import {
  buildNumberAst,
  buildStringAst,
  buildPlusOpAst,
  buildProcedureAst,
  buildCellReferenceAst,
  CellAddress
} from 'hyperformula';

// Build: =A1 + 42
const cellRef = buildCellReferenceAst(
  new CellAddress(0, 0, 'A1', false, false)
);
const number = buildNumberAst(42);
const addition = buildPlusOpAst(cellRef, number);

// Build: =SUM(A1:A10)
const sum = buildProcedureAst('SUM', [
  /* range ast */
]);
```

### 5. Dependency Collection

Extract dependencies from formulas:

```typescript
import { collectDependencies, Ast } from 'hyperformula';

const dependencies = collectDependencies(ast, functionRegistry);

// Dependencies include:
// - Cell references
// - Range references
// - Named expressions
// - etc.
```

### 6. Unparser

Convert AST back to formula strings:

```typescript
import { Unparser, SimpleCellAddress } from 'hyperformula';

const unparser = new Unparser(config, lexerConfig, sheetMapping);
const formulaString = unparser.unparse(ast, address);
```

### 7. Address Utilities

Work with cell addresses:

```typescript
import {
  cellAddressFromString,
  simpleCellAddressFromString,
  simpleCellAddressToString,
  simpleCellRangeFromString,
  simpleCellRangeToString
} from 'hyperformula';

// Parse addresses
const addr = simpleCellAddressFromString(sheetMapping, 'Sheet1!A1', 0);
const range = simpleCellRangeFromString(sheetMapping, 'Sheet1!A1:B10', 0);

// Convert to string
const addrStr = simpleCellAddressToString(sheetMapping, addr);
const rangeStr = simpleCellRangeToString(sheetMapping, range);
```

## Complete Example: Custom Formula Parser

```typescript
import {
  ParserWithCaching,
  Ast,
  AstNodeType,
  SimpleCellAddress,
  buildLexerConfig,
  collectDependencies
} from 'hyperformula';

// Setup
const config = { /* your config */ };
const functionRegistry = /* your function registry */;
const lexerConfig = buildLexerConfig(config);
const sheetMapping = (name: string) => 0;

const parser = new ParserWithCaching(
  config,
  functionRegistry,
  sheetMapping
);

// Parse formulas
function parseFormula(formula: string, address: SimpleCellAddress) {
  const result = parser.parse(formula, address);
  
  if (result.errors.length > 0) {
    console.error('Parsing errors:', result.errors);
    return null;
  }
  
  return {
    ast: result.ast,
    dependencies: result.dependencies,
    hasVolatile: result.hasVolatileFunction
  };
}

// Analyze AST
function analyzeAst(ast: Ast): any {
  switch (ast.type) {
    case AstNodeType.NUMBER:
      return { type: 'number', value: ast.value };
    
    case AstNodeType.STRING:
      return { type: 'string', value: ast.value };
    
    case AstNodeType.CELL_REFERENCE:
      return { type: 'cellRef', reference: ast.reference };
    
    case AstNodeType.FUNCTION_CALL:
      return {
        type: 'function',
        name: ast.procedureName,
        args: ast.args.map(analyzeAst)
      };
    
    case AstNodeType.PLUS_OP:
      return {
        type: 'plus',
        left: analyzeAst(ast.left),
        right: analyzeAst(ast.right)
      };
    
    // ... handle other node types
    
    default:
      return { type: 'unknown' };
  }
}

// Usage
const formula = '=SUM(A1:A10) + 5';
const address: SimpleCellAddress = { sheet: 0, row: 0, col: 0 };

const parsed = parseFormula(formula, address);
if (parsed) {
  console.log('AST:', parsed.ast);
  console.log('Dependencies:', parsed.dependencies);
  console.log('Analysis:', analyzeAst(parsed.ast));
}
```

## Fire-and-Forget Formula Evaluation

If you already have a HyperFormula instance and want to evaluate a formula without adding it to a cell, use the `calculateFormula` method:

```typescript
const hf = HyperFormula.buildFromSheets({
  Sheet1: [[1, 2, 3], [4, 5, 6]]
});

// Evaluate without modifying the sheet
const result = hf.calculateFormula('=SUM(A1:C2)', 0);
console.log(result); // 21
```

## Benefits

### 1. No Instance Required
Parse formulas without creating a full HyperFormula instance, useful for:
- Formula validation
- Syntax highlighting
- Formula analysis tools
- Custom formula builders

### 2. Performance
- Built-in caching reduces parsing overhead
- Reuse parser instances across multiple formulas
- Minimal memory footprint

### 3. Flexibility
- Custom evaluation logic
- Formula transformation and optimization
- Integration with custom data structures
- Educational tools and debuggers

## Advanced Use Cases

### Formula Validation
```typescript
function validateFormula(formula: string): boolean {
  const result = parser.parse(formula, someAddress);
  return result.errors.length === 0;
}
```

### Extract Cell References
```typescript
function extractCellRefs(ast: Ast): string[] {
  const refs: string[] = [];
  
  function traverse(node: Ast) {
    if (node.type === AstNodeType.CELL_REFERENCE) {
      refs.push(node.reference.toString());
    } else if ('left' in node && 'right' in node) {
      traverse(node.left);
      traverse(node.right);
    }
    // ... handle other node types
  }
  
  traverse(ast);
  return refs;
}
```

### Formula Complexity Analysis
```typescript
function analyzeComplexity(ast: Ast): number {
  let complexity = 0;
  
  function traverse(node: Ast) {
    complexity++;
    
    if (node.type === AstNodeType.FUNCTION_CALL) {
      complexity += 5; // Functions add complexity
      node.args.forEach(traverse);
    } else if ('left' in node && 'right' in node) {
      traverse(node.left);
      traverse(node.right);
    }
    // ... handle other node types
  }
  
  traverse(ast);
  return complexity;
}
```

## Notes

- The parser is stateless and thread-safe
- Caching is automatic and transparent
- AST nodes are immutable
- Address mappings depend on your sheet structure
- Function registry must be provided for function parsing

