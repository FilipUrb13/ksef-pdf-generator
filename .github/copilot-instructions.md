# KSeF PDF Generator - AI Agent Instructions

## Project Overview
TypeScript library for generating PDF visualizations of Polish KSeF (Krajowy System e-Faktur / National e-Invoice System) invoices and UPO (confirmations). Converts XML invoices conforming to FA(1), FA(2), FA(3) schemas and UPO v4.2/v4.3 into PDF documents using pdfmake.

## Architecture

### Component Structure
- **`src/lib-public/`**: Public API and generators - main library exports
  - `generate-invoice.ts`: Entry point that routes FA(1), FA(2), or FA(3) based on `KodFormularza.kodSystemowy` 
  - `FA1-generator.ts`, `FA2-generator.ts`, `FA3-generator.ts`: Schema-specific PDF builders
  - `UPO-generator.ts`: UPO confirmation PDF generator
  - **`generators/`**: Modular PDF content generators organized by schema (FA1/, FA2/, FA3/, UPO4_2/, UPO4_3/, common/)
  - **`types/`**: TypeScript interfaces mirroring XML schemas (fa1.types.ts, fa2.types.ts, etc.)
- **`src/shared/`**: Common utilities across all generators
  - `XML-parser.ts`: Converts XML to JSON using xml-js, strips namespace prefixes
  - `PDF-functions.ts`: pdfmake helpers (formatting, tables, styles, QR codes)
  - `consts/const.ts`: Tax rates, invoice types, country codes matching KSeF schemas
- **`src/app-public/`**: Demo web application (Vite dev server)

### Data Flow
1. **XML → JSON**: `parseXML()` reads File, uses `xml2js`, strips prefixes with `stripPrefixes()`
2. **Type Detection**: `generateInvoice()` reads `Faktura.Naglowek.KodFormularza._attributes.kodSystemowy` to determine FA(1)/FA(2)/FA(3)
3. **PDF Generation**: Schema-specific generator calls modular generators (Naglowek, Wiersze, Rozliczenie, etc.)
4. **Output**: Returns Promise<Blob> or Promise<string> (base64) via pdfmake

### Key Patterns

#### Polish-English Mixed Naming
Variable/type names mirror XML schema structure from Polish tax authorities. **Do not refactor to English** - preserves traceability to official schemas:
```typescript
// Correct - matches schema
interface Podmiot1 { DaneIdentyfikacyjne?: DaneIdentyfikacyjne; }
const P_13_1 = faktura.Fa?.P_13_1?._text; 
```

#### FP Type Pattern
XML text nodes wrapped in objects with `_text` and optional `_attributes`:
```typescript
export interface FP { _text?: string; _attributes?: Record<string, string>; }
```
Access via: `invoice.Fa?.P_2?._text` (always use optional chaining)

#### Generator Module Pattern
Each visual section is a separate generator function returning pdfmake `Content`:
```typescript
// src/lib-public/generators/FA1/Adnotacje.ts
export function generateAdnotacje(adnotacje?: Adnotacje): Content[] {
  if (!adnotacje) return [];
  return [createHeader('Adnotacje'), createLabelText('P_18A', adnotacje.P_18A)];
}
```
Compose in main generator: `content: [...generateNaglowek(), generateWiersze(), ...]`

#### Test Structure
- Use Vitest with vi.mock() for dependencies
- Co-located: `Adnotacje.ts` → `Adnotacje.spec.ts`
- Mock shared functions: `createHeader`, `formatText`, `hasValue` for unit isolation
- Test generator output structure: `expect(result.some(el => JSON.stringify(el).includes('...'))).toBeTruthy()`

## Critical Workflows

### Development
```bash
npm run dev          # Start demo app on http://localhost:5173
npm run test         # Watch mode testing
npm run test:ui      # Vitest UI at http://localhost:51204/__vitest__/
npm run test:ci      # CI mode with coverage (generates /coverage/index.html)
```

### Building Library
```bash
npm run build        # Outputs to /dist: ES module + UMD + .d.ts types
```
Vite config uses `vite-plugin-dts` for type generation. Excludes .spec.ts and app-public from build.

### Adding New Invoice Type Support
1. Create type definitions in `src/lib-public/types/faN.types.ts` mirroring XML schema
2. Add generator: `src/lib-public/FaN-generator.ts` 
3. Create modular generators in `src/lib-public/generators/FaN/`
4. Update switch in `src/lib-public/generate-invoice.ts`:
   ```typescript
   case 'FA (N)':
     pdf = generateFAN((xml as any).Faktura as FakturaN, additionalData);
   ```
5. Export in `src/index.ts`

## Project-Specific Conventions

### Type Definitions
- Types in `types/` directories mirror XML structure exactly
- Use `FP` interface for fields with `_text` property
- Array types: `Podmiot3?: Podmiot3[]` (fields ending in 3 often arrays)

### PDF Formatting
- Use `FormatTyp` enum from `shared/enums/common.enum.ts` for consistent styling
- Helper functions: `formatText()`, `createLabelText()`, `hasValue()`, `getValue()`
- Tax rate constants: `TStawkaPodatku_FA1`, `TStawkaPodatku_FA2`, `TStawkaPodatku_FA3` in `const.ts`

### Schema Handling
- **FA(1)**: Standard invoices (RodzajFaktury: VAT, KOR, ZAL, etc.)
  - Korekt rabat: Check `invoice.Fa?.OkresFaKorygowanej` - generates `Rabat` instead of `Wiersze`
- **FA(2)**: Invoices with additional entities (Podmiot3, different tax summaries)
- **FA(3)**: Export/import invoices
- **UPO**: Confirmations use landscape A4, different structure

### Error Handling
No explicit error boundaries - rely on optional chaining (`?.`) throughout. Invalid XML rejected in `parseXML()` promise.

## External Dependencies
- **pdfmake**: PDF generation - import fonts via `pdfmake/build/vfs_fonts`
- **xml-js**: XML parsing - use `compact: true` mode
- **Vitest**: Testing with jsdom environment for File/FileReader APIs

## Integration Points
Library exports via `src/index.ts`:
```typescript
export { generateInvoice, generatePDFUPO } from './lib-public';
export { generateFA1, generateFA2, generateFA3 } from './lib-public/...';
```
Consumers call `generateInvoice(file, additionalData, 'blob' | 'base64')` or schema-specific generators.

## DO NOT
- Translate Polish variable names to English (breaks schema mapping)
- Use `any` type freely - `@typescript-eslint/no-explicit-any` is OFF, but prefer proper types
- Modify `const.ts` tax rates/codes without verifying against official KSeF schemas
- Add business logic to formatters in `PDF-functions.ts` - keep pure utilities
