# MDS Formatter Evaluation System - Simplified Scoring Matrix

## 📊 SCORING METHODOLOGY

### Design Principles

The scoring system totals **exactly 100 points**, distributed according to the **operational criticality** of each formatter component.

**Design Criteria:**
1. **Operational Criticality**: Higher scores for elements that directly affect core functionality
2. **Data Quality Impact**: Priority on validations that protect PHI data integrity
3. **Framework Compliance**: Strict adherence to established MDS patterns
4. **Maintainability**: Code that facilitates debugging, auditing, and evolution

---

## 🎯 TOTAL POINT DISTRIBUTION (100 points)

| Category | Points | % of Total | Weight Justification |
|---------|--------|------------|----------------------|
| **1. Base Architecture** | 25 | 25% | Without correct architecture, the formatter cannot be invoked by TaskLauncher |
| **2. Processing Pipeline** | 20 | 20% | Controls the full lifecycle — errors cause runtime failures |
| **3. FileHelpers / Record Types** | 18 | 18% | Data parser — errors cause client data loss or corruption |
| **4. Handlers** | 17 | 17% | Core business logic — incorrect implementation = incorrect data |
| **5. Cross-Walks & Configuration** | 12 | 12% | Code mappings — without them, transactions/accounts remain unclassified |
| **6. Robustness & Edge Cases** | 8 | 8% | Prevents silent production errors — critical for reliability |
| **TOTAL** | **100** | **100%** | |

---

# 1. BASE ARCHITECTURE & FRAMEWORK COMPLIANCE (25 points)

**Rationale:** Base architecture is the **contractual foundation** between the formatter and the MDS framework. Without the correct structure:
- TaskLauncher cannot instantiate the formatter (runtime exception)
- The configuration pipeline cannot execute
- Cross-walk persistence is not available
- Recall protection cannot be implemented

**Failure Impact:** **BLOCKING** — the formatter is not executable if this section fails.

## 1.1 Correct Inheritance from BaseConverter (10 points)

**Why 10 points:**
- Validates the **primary framework contract**
- Failures cause **compilation errors** or **runtime exceptions**
- Most frequent MVP issue per MDS feedback

| Criterion | Points | Justification | Validation Method | Penalty |
|---------|--------|---------------|-------------------|---------| 
| Class inherits from BaseConverter | 4 | CRITICAL: TaskLauncher requires this inheritance | `class X : BaseConverter` | BLOCKING: Formatter not executable |
| Constructor receives ConverterCmd | 3 | CRITICAL: Execution context required | `public X(ConverterCmd cmd) : base(cmd)` | BLOCKING |
| Implements IConverterSettings | 2 | Enables persistent cross-walk configuration | Interface declaration present | Configuration lost between runs |
| Implements IAccountCache | 1 | Required for recall protection | Interface + `AccountCache Accounts` | Business rule violation |

**Total: 4 + 3 + 2 + 1 = 10 points**

## 1.2 Mandatory Abstract Method Implementations (10 points)

**Why 10 points:**
- `GetConverter()` and `QualifyFile()` are abstract in `BaseConverter`
- Missing implementations cause compilation failure
- They are the entry point to file processing
- Errors result in silently ignored files

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| GetConverter implemented | 5 | Core routing logic for handlers | Method exists, non-abstract | BLOCKING |
| All InputTypes covered | 2 | Prevents silent file skipping | Switch covers all types | Data loss |
| QualifyFile implemented | 3 | File classification logic | Method returns InputType | BLOCKING |

**Total: 5 + 2 + 3 = 10 points**

## 1.3 Framework Properties & Constants (5 points)

**Why 5 points:**
- Not blocking, but missing elements cause silent production defects
- Represent established MDS best practices

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| ClientCode constant | 2 | Required for logging and config | Public const present | Inconsistencies |
| Interface properties initialized | 2 | Framework access requirement | Constructor initialization | Runtime null errors |
| AccountType enum (conditional) | 1 | Clean business logic separation | Enum present if required | Logic coupling |

**Total: 2 + 2 + 1 = 5 points**

---

# 2. PROCESSING PIPELINE & FORMATTER LIFECYCLE (20 points)

**Rationale:** The processing pipeline governs the **entire execution lifecycle** of the formatter. Errors in this section do not merely affect individual records — they compromise **every execution run**.

Without a correctly implemented pipeline:
- Configuration values are not loaded
- Cross-walks cannot be persisted or recalled
- Runtime context is lost between executions
- Handlers operate with incomplete state

**Failure Impact:** **HIGH / SYSTEMIC** — failures propagate across all processed files.

## 2.1 LoadSettings Override (8 points)

**Why 8 points:**
- `LoadSettings()` is responsible for **hydrating runtime configuration**
- Missing or partial implementation causes defaults to override persisted values
- One of the most common causes of silent production defects

| Criterion | Points | Justification | Validation Method | Penalty |
|---------|--------|---------------|-------------------|---------| 
| Override LoadSettings | 3 | Mandatory lifecycle hook | Method override exists | Defaults used |
| Calls base.LoadSettings | 2 | Preserves framework state | Explicit base call | Partial config |
| Loads Cross-Walks | 2 | Required for classification | Repository access | Unmapped records |
| Defensive null handling | 1 | Prevents runtime crashes | Guards present | Runtime exception |

**Total: 3 + 2 + 2 + 1 = 8 points**

## 2.2 Configure Override (7 points)

**Why 7 points:**
- `Configure()` binds **runtime behavior** to configuration state
- Determines handler activation and processing rules
- Errors here lead to inconsistent behavior across runs

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| Override Configure | 3 | Mandatory lifecycle hook | Method exists | Undefined behavior |
| Uses loaded settings | 2 | Ensures config consistency | Settings referenced | Hard-coded logic |
| Initializes handlers | 2 | Correct pipeline routing | Handler list populated | Files ignored |

**Total: 3 + 2 + 2 = 7 points**

## 2.3 SaveSettings Override (5 points)

**Why 5 points:**
- Ensures **persistence of runtime modifications**
- Required for cross-walk evolution
- Missing implementation causes regression between runs

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| Override SaveSettings | 2 | Lifecycle completion | Method exists | State lost |
| Persists Cross-Walks | 2 | Required for learning systems | Repository save | Regression |
| Calls base.SaveSettings | 1 | Framework consistency | Explicit base call | Partial persistence |

**Total: 2 + 2 + 1 = 5 points**

---

# 3. FILEHELPERS & RECORD TYPE DEFINITIONS (18 points)

**Rationale:** FileHelpers record definitions form the **data ingestion layer** of the formatter. Any error at this stage results in **irreversible data corruption, truncation, or loss** before business logic is applied.

Parsing errors are particularly dangerous because they often:
- Do not raise runtime exceptions
- Produce structurally valid but semantically incorrect records
- Propagate silently into downstream systems

**Failure Impact:** **HIGH / DATA-INTEGRITY CRITICAL** — defects affect client data correctness.

## 3.1 Record Class Attributes (6 points)

**Why 6 points:**
- Record-level attributes define file structure and delimiters
- Incorrect attributes misalign every field in the record
- Errors affect 100% of parsed rows

| Criterion | Points | Justification | Validation Method | Penalty |
|---------|--------|---------------|-------------------|---------| 
| Correct FileHelpers record attribute | 2 | Establishes parsing mode | Attribute present | Misaligned parsing |
| Delimiter matches input file | 2 | Structural correctness | Attribute value check | Field shifting |
| IgnoreEmptyLines / TrimOptions | 1 | Prevents phantom records | Attribute flags | Garbage records |
| Encoding explicitly defined | 1 | Prevents character loss | Encoding property | Data truncation |

**Total: 2 + 2 + 1 + 1 = 6 points**

## 3.2 Field-Level Attributes (6 points)

**Why 6 points:**
- Field attributes control column position, length, and formatting
- Minor mistakes result in swapped or truncated values
- Extremely difficult to detect post-processing

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| FieldOrder explicitly defined | 2 | Prevents implicit ordering | Attribute check | Mis-mapped fields |
| FieldOptional used correctly | 1.5 | Handles missing columns | Attribute logic | Runtime parsing errors |
| FieldLength / FieldTrim | 1.5 | Prevents overflow/truncation | Attribute values | Data loss |
| Converter or format specified | 1 | Ensures type correctness | Converter attribute | Invalid values |

**Total: 2 + 1.5 + 1.5 + 1 = 6 points**

## 3.3 Database Field Mapping & Validation (6 points)

**Why 6 points:**
- Parsed fields must map cleanly to database schema
- Mismatches cause rejected inserts or silent nulls
- Errors often surface only in downstream reconciliation

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| One-to-one field mapping | 2 | Schema integrity | Mapping review | Insert failures |
| Required fields validated | 2 | DB constraints compliance | Null checks | Transaction rollback |
| Data types aligned | 1 | Prevents casting errors | Type comparison | Runtime exception |
| Default values defined | 1 | Prevents null propagation | Initialization logic | Silent nulls |

**Total: 2 + 2 + 1 + 1 = 6 points**

---

# 4. HANDLERS & BUSINESS LOGIC IMPLEMENTATION (17 points)

**Rationale:** Handlers encapsulate the **core business logic** of the formatter. While the pipeline ensures execution and FileHelpers ensure correct ingestion, handlers determine **how data is interpreted, classified, and transformed**.

Errors at this layer do not usually crash the system — they are more dangerous because they:
- Produce logically incorrect but technically valid outputs
- Pass undetected through automated checks
- Directly impact billing, reporting, or regulatory outcomes

**Failure Impact:** **HIGH / BUSINESS-CRITICAL** — incorrect logic equals incorrect data.

## 4.1 Handler Registration & Routing (5 points)

**Why 5 points:**
- Incorrect routing results in records being processed by the wrong handler
- Some records may be skipped entirely
- Errors are often silent

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| All handlers registered | 2 | Ensures full coverage | Handler list inspection | Records ignored |
| Correct InputType routing | 2 | Prevents misclassification | Switch / map validation | Wrong business rules |
| Default / fallback handler | 1 | Prevents data loss | Default path exists | Unhandled records |

**Total: 2 + 2 + 1 = 5 points**

## 4.2 Business Rule Implementation (7 points)

**Why 7 points:**
- This is where domain logic is actually enforced
- Errors directly affect client-facing outcomes
- Often difficult to detect without domain expertise

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| Rules match specifications | 3 | Contractual correctness | Spec comparison | Compliance breach |
| Conditional logic complete | 2 | Handles all scenarios | Branch coverage | Edge-case failure |
| Uses cross-walks correctly | 2 | Consistent classification | Lookup validation | Mislabeling |

**Total: 3 + 2 + 2 = 7 points**

## 4.3 Error Handling & Logging (3 points)

**Why 3 points:**
- Errors must be observable
- Silent failures undermine auditability
- Logging is essential for client support

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| Meaningful error messages | 1 | Supports debugging | Log content review | Opaque failures |
| Correct log level usage | 1 | Signal vs noise | Level inspection | Missed alerts |
| Exception containment | 1 | Prevents pipeline crash | Try/catch review | Execution abort |

**Total: 1 + 1 + 1 = 3 points**

## 4.4 Handler State & Side Effects (2 points)

**Why 2 points:**
- Handlers should be stateless or explicitly managed
- Hidden state causes non-deterministic behavior

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| No hidden mutable state | 1 | Deterministic processing | Code inspection | Inconsistent output |
| Explicit dependencies | 1 | Testability & clarity | Constructor review | Tight coupling |

**Total: 1 + 1 = 2 points**

---

# 5. CROSS-WALKS & CONFIGURATION MANAGEMENT (12 points)

**Rationale:** Cross-walks and configuration artifacts define how raw input values are **translated into standardized, reportable categories**. While not always blocking at runtime, errors here lead to **systematic misclassification** across large data volumes.

This section evaluates the formatter's ability to:
- Load configurable mappings correctly
- Apply them consistently during processing
- Persist learned or updated mappings safely

**Failure Impact:** **MEDIUM–HIGH / SYSTEMATIC** — defects scale with data volume.

## 5.1 Cross-Walk Definition & Structure (6 points)

**Why 6 points:**
- Defines the canonical mapping between source and target domains
- Structural issues affect every lookup
- Often maintained by non-developers, increasing risk

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| Explicit source → target keys | 2 | Prevents ambiguous mapping | Schema inspection | Misclassification |
| Strongly typed value objects | 1.5 | Compile-time safety | Type review | Runtime casting errors |
| Default / unknown handling | 1.5 | Prevents null propagation | Fallback logic | Silent nulls |
| Versionable structure | 1 | Supports evolution | Metadata present | Breaking changes |

**Total: 2 + 1.5 + 1.5 + 1 = 6 points**

## 5.2 Cross-Walk Loading & Lookup Logic (6 points)

**Why 6 points:**
- Correct loading ensures mappings are available at runtime
- Lookup logic must be deterministic and performant

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| Loaded during LoadSettings | 3 | Lifecycle correctness | Call trace | Empty mappings |
| Safe lookup pattern | 1.5 | Prevents exceptions | TryGetValue usage | Runtime failure |
| Deterministic resolution | 1.5 | Consistent results | Unit-test review | Non-repeatable output |

**Total: 3 + 1.5 + 1.5 = 6 points**

---

# 6. ROBUSTNESS, DEFENSIVE CODING & EDGE CASES (8 points)

**Rationale:** Robustness measures how well the formatter behaves under **non-ideal, unexpected, or degraded conditions**. While these scenarios may not occur in every execution, failure to handle them leads to **production instability, support escalations, and loss of trust**.

This section evaluates the formatter's ability to:
- Fail safely instead of catastrophically
- Detect and surface anomalous conditions
- Continue processing when partial data is valid

**Failure Impact:** **MEDIUM / RELIABILITY-CRITICAL** — defects surface in production under stress.

## 6.1 Null, Empty, and Missing Data Handling (3 points)

**Why 3 points:**
- Real-world files frequently contain missing or optional values
- Unchecked nulls are a leading cause of runtime exceptions

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| Explicit null checks | 1 | Prevents crashes | Code inspection | Runtime exception |
| Safe string handling | 1 | Avoids trim/parse errors | Guard clauses | Processing abort |
| Defaults for missing data | 1 | Deterministic output | Default assignment | Silent nulls |

**Total: 1 + 1 + 1 = 3 points**

## 6.2 Boundary Conditions & Data Extremes (2 points)

**Why 2 points:**
- Extreme values (length, size, numeric bounds) reveal hidden defects
- Often missed in happy-path testing

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| Length / range validation | 1 | Prevents overflow | Conditional checks | Truncation |
| Safe numeric parsing | 1 | Prevents exceptions | TryParse usage | Execution abort |

**Total: 1 + 1 = 2 points**

## 6.3 Partial Failure Tolerance (2 points)

**Why 2 points:**
- A single bad record should not invalidate an entire file
- Required for high-volume batch processing

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| Record-level exception handling | 1 | Isolates failures | Try/catch scope | Batch abort |
| Continues after recoverable errors | 1 | Maximizes throughput | Control flow | Data loss |

**Total: 1 + 1 = 2 points**

## 6.4 Observability & Diagnostics (1 point)

**Why 1 point:**
- Diagnostics support faster resolution but are not blocking

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------| 
| Context-rich logs | 0.5 | Faster debugging | Log review | Slow triage |
| Metrics / counters | 0.5 | Trend detection | Metric presence | Blind spots |

**Total: 0.5 + 0.5 = 1 point**

---

# FINAL SCORING SUMMARY

## Quality Thresholds
- **🟢 PRODUCTION READY**: ≥ 90 points (90%+)
- **🟡 NEEDS MINOR FIXES**: 75-89 points (75-89%)
- **🟠 NEEDS MODERATE FIXES**: 60-74 points (60-74%)
- **🔴 MAJOR REWORK REQUIRED**: < 60 points (<60%)

## Mandatory Failure Conditions
Regardless of numeric score, a formatter is **automatically rejected** if any of the following occur:
- Missing BaseConverter inheritance
- Missing mandatory lifecycle overrides
- FileHelpers parsing defects
- Systematic business-rule misclassification

These conditions represent **non-negotiable quality gates**.

## Final Release Decision
> A formatter may only be released when:
> - All blocking conditions are cleared
> - Final score meets or exceeds the required threshold
> - Evidence artifacts are archived and reviewable

**TOTAL VERIFICATION:**
- Base Architecture: 10 + 10 + 5 = **25 points**
- Processing Pipeline: 8 + 7 + 5 = **20 points**
- FileHelpers: 6 + 6 + 6 = **18 points**
- Handlers: 5 + 7 + 3 + 2 = **17 points**
- Cross-Walks: 6 + 6 = **12 points**
- Robustness: 3 + 2 + 2 + 1 = **8 points**

**GRAND TOTAL: 100 points** ✓






