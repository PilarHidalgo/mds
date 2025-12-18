
# Scoring Matrix with Standards and Score Justification
## MDS Formatter Generator Project Evaluation System

---

## 📊 SCORING METHODOLOGY

### Scoring System Design Principles

The scoring system is designed to total **exactly 100 points**, distributed according to the **operational criticality** of each formatter component.

**Score Allocation Formula:**

Category Score = (Criticality × Impact × Error Frequency) / Normalization Factor

Where:
- **Criticality**: How severe is the failure? (1 = Warning, 5 = Blocking)
- **Impact**: How many components are affected? (1 = Local, 5 = System-wide)
- **Frequency**: How common is this error in real-world formatters? (1 = Rare, 5 = Very common)

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

**Distribution Rationale:**

- **Architecture (25%)**: Highest weight because it is a prerequisite for everything else. Without proper inheritance from `BaseConverter`, the formatter does not work.
- **Pipeline (20%)**: Second highest weight because it controls configuration and persistence. Errors here affect all executions.
- **FileHelpers (18%)**: Third because incorrect parsing results in client data loss.
- **Handlers (17%)**: Fourth because they implement specific business logic.
- **Cross-Walks (12%)**: Lower relative weight because they are configurable post-deployment.
- **Robustness (8%)**: Lowest weight because these are optimizations, not blocking requirements.

---

# 1. BASE ARCHITECTURE & FRAMEWORK COMPLIANCE (25 points)

## Total Weight Justification: 25 points (25% of total score)

**Rationale:** Base architecture is the **contractual foundation** between the formatter and the MDS framework. Without the correct structure:
- TaskLauncher cannot instantiate the formatter (runtime exception)
- The configuration pipeline cannot execute
- Cross-walk persistence is not available
- Recall protection cannot be implemented

**Failure Impact:** **BLOCKING** — the formatter is not executable if this section fails.

**Contextual Evidence of Criticality:**
> "Major Issues: Handlers not inheriting from Meditech base classes" (MDS Project Follow-Up)

---

## 1.1 Correct Inheritance from BaseConverter (10 points)

### Subtotal Justification: 10 points (40% of Architecture)

**Why 10 points:**
- Validates the **primary framework contract**
- Failures cause **compilation errors** or **runtime exceptions**
- Most frequent MVP issue per MDS feedback

**Score Calculation:**
```
Criticality: 5/5 (BLOCKING)
Impact: 5/5 (System-wide)
Frequency: 5/5 (Common MVP error)
Score = (5 × 5 × 5) / 12.5 = 10 points
```

| Criterion | Points | Score Justification | Validation Method | Penalty |
|---------|--------|---------------------|-------------------|---------|
| Class inherits from BaseConverter | 4 | CRITICAL: TaskLauncher requires this inheritance | `class X : BaseConverter` | BLOCKING: Formatter not executable |
| Constructor receives ConverterCmd | 3 | CRITICAL: Execution context required | `public X(ConverterCmd cmd) : base(cmd)` | BLOCKING |
| Implements IConverterSettings | 1.5 | Enables persistent cross-walk configuration | Interface declaration present | Configuration lost between runs |
| Implements IAccountCache | 1.5 | Required for recall protection | Interface + `AccountCache Accounts` | Business rule violation |

**Validation agent logic** 
```csharp
public ValidationResult ValidateBaseArchitecture(Type formatterType)
{
    var result = new ValidationResult { Category = "Architecture", MaxScore = 10 };
    
    // Check 1: BaseConverter inheritance (4 pts)
    if (!formatterType.IsSubclassOf(typeof(BaseConverter)))
    {
        result.AddCriticalError(
            "BLOCKER: Class does not inherit from BaseConverter", 
            $"Expected: class {formatterType.Name} : BaseConverter"
        );
        result.Score = 0; // Bloqueante - no sumar otros puntos
        return result;
    }
    result.Score += 4;
    result.AddEvidence($"✓ Inherits from BaseConverter");
    
    // Check 2: Constructor (3 pts)
    var ctor = formatterType.GetConstructor(new[] { typeof(ConverterCmd) });
    if (ctor == null)
    {
        result.AddCriticalError(
            "BLOCKER: Missing constructor with ConverterCmd parameter",
            $"Expected: public {formatterType.Name}(ConverterCmd cmd) : base(cmd)"
        );
        return result; // No continuar
    }
    result.Score += 3;
    result.AddEvidence($"✓ Constructor accepts ConverterCmd");
    
    // Check 3: IConverterSettings (1.5 pts)
    if (!typeof(IConverterSettings).IsAssignableFrom(formatterType))
    {
        result.AddWarning(
            "Missing IConverterSettings interface - cross-walks won't persist",
            "Impact: Users must reconfigure on each execution"
        );
    }
    else
    {
        result.Score += 1.5;
        result.AddEvidence($"✓ Implements IConverterSettings");
    }
    
    // Check 4: IAccountCache (1.5 pts)
    if (!typeof(IAccountCache).IsAssignableFrom(formatterType))
    {
        result.AddWarning(
            "Missing IAccountCache - recall protection disabled",
            "Impact: Deleted accounts may receive new transactions"
        );
    }
    else
    {
        result.Score += 1.5;
        result.AddEvidence($"✓ Implements IAccountCache");
    }
    
    return result;
}
```

---

## 1.2 Mandatory Abstract Method Implementations (10 points)

### Subtotal Justification: 10 points (40% of Architecture)

**Why 10 points:**
- `GetConverter()` and `QualifyFile()` are abstract in `BaseConverter`
- Missing implementations cause compilation failure
- They are the entry point to file processing
- Errors result in silently ignored files

**Score Calculation:**
```
Criticality: 5/5 (Compilation failure)
Impact: 5/5 (No files processed)
Frequency: 4/5 (Common in auto-generated code)
Score = (5 × 5 × 4) / 10 = 10 points
```

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| GetConverter implemented | 5 | Core routing logic for handlers | Method exists, non-abstract | BLOCKING |
| All InputTypes covered | 2 | Prevents silent file skipping | Switch covers all types | Data loss |
| QualifyFile implemented | 3 | File classification logic | Method returns InputType | BLOCKING |

**Validation agent logic** 
```csharp
public ValidationResult ValidateAbstractMethods(Type formatterType)
{
    var result = new ValidationResult { Category = "Abstract Methods", MaxScore = 10 };
    
    // GetConverter validation (5 pts base + 2 pts coverage)
    var getConverter = formatterType.GetMethod("GetConverter", 
        BindingFlags.Public | BindingFlags.Instance);
    
    if (getConverter == null || getConverter.IsAbstract)
    {
        result.AddCriticalError(
            "BLOCKER: GetConverter() not implemented",
            "This is an abstract method in BaseConverter - code will not compile"
        );
        return result;
    }
    result.Score += 5;
    result.AddEvidence("✓ GetConverter() implemented");
    
    // Analyze coverage of InputTypes
    var inputTypesCovered = AnalyzeInputTypeCoverage(getConverter);
    var expectedTypes = DetermineExpectedInputTypes(formatterType);
    
    if (inputTypesCovered.Count >= expectedTypes.Count)
    {
        result.Score += 2;
        result.AddEvidence($"✓ GetConverter() handles all {expectedTypes.Count} expected InputTypes");
    }
    else
    {
        result.AddWarning(
            $"GetConverter() only handles {inputTypesCovered.Count}/{expectedTypes.Count} InputTypes",
            $"Missing: {string.Join(", ", expectedTypes.Except(inputTypesCovered))}"
        );
        // Partial credit
        result.Score += 2.0 * (inputTypesCovered.Count / (double)expectedTypes.Count);
    }
    
    // QualifyFile validation (3 pts)
    var qualifyFile = formatterType.GetMethod("QualifyFile", 
        BindingFlags.NonPublic | BindingFlags.Instance);
    
    if (qualifyFile == null || qualifyFile.IsAbstract)
    {
        result.AddCriticalError(
            "BLOCKER: QualifyFile() not implemented",
            "This is an abstract method in BaseConverter - code will not compile"
        );
        return result;
    }
    result.Score += 3;
    result.AddEvidence("✓ QualifyFile() implemented");
    
    return result;
}

private List<InputType> AnalyzeInputTypeCoverage(MethodInfo method)
{
    var covered = new List<InputType>();
    var methodBody = DecompileMethodBody(method);
    
    // Analyze switch cases or if statements
    if (methodBody.Contains("InputType.Collections")) covered.Add(InputType.Collections);
    if (methodBody.Contains("InputType.Inventory")) covered.Add(InputType.Inventory);
    if (methodBody.Contains("InputType.Insurance")) covered.Add(InputType.Insurance);
    if (methodBody.Contains("InputType.Skip")) covered.Add(InputType.Skip);
    
    return covered;
}
```
**Reference code**
```csharp
// [5 pts] - GetConverter implementation
public override BaseConversionClass GetConverter(ProcessFile pFile)
{
    // WHY CRITICAL: This is the routing logic for all file processing
    // IMPACT: If a case is missing, that file type is silently ignored
    
    switch (pFile.FileType)
    {
        case InputType.Collections:  // [+0.5 pt] Demographics handler
            return new DemographicsHandler(this, pFile);
            
        case InputType.Inventory:    // [+0.5 pt] Inventory handler
            return new InventoryHandler(this, pFile, typeof(InventoryRecordType));
            
        case InputType.Insurance:    // [+0.5 pt] Insurance handler (if applicable)
            return new InsuranceHandler(this, pFile);
            
        case InputType.Skip:         // [+0.5 pt] Known files to skip
            return null; // Explicitly ignored
            
        default:
            return null;
    }
    // [+2 pts if all expected InputTypes covered]
}

// [3 pts] - QualifyFile implementation
protected override InputType QualifyFile(ProcessFile pFile)
{
    // WHY CRITICAL: Incorrect classification = wrong handler or no processing
    // IMPACT: Client files may be processed incorrectly or not at all
    
    string fileName = pFile.ShortName.ToUpper();
    
    // Pattern matching should be specific to avoid conflicts
    if (fileName.Contains("DEMO") || fileName.Contains("PATIENT"))
        return InputType.Collections;
        
    if (fileName.Contains("INV") || fileName.Contains("INVENTORY"))
        return InputType.Inventory;
        
    if (fileName.Contains("INSURANCE") || fileName.Contains("INS"))
        return InputType.Insurance;
        
    // Files that should be ignored (e.g., Excel exports)
    if (fileName.EndsWith(".XLSX") || fileName.Contains("BACKUP"))
        return InputType.Skip;
        
    return InputType.Unknown; // Will be logged for review
}
```

---

## 1.3 Framework Properties & Constants (5 points)

### Subtotal Justification: 5 points (20% of Architecture)

**Why 5 points:**
- Not blocking, but missing elements cause silent production defects
- Represent established MDS best practices

**Score Calculation:**
```
Criticality: 3/5
Impact: 3/5
Frequency: 4/5
Score = (3 × 3 × 4) / 7.2 = 5 points
```

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| ClientCode constant | 2 | Required for logging and config | Public const present | Inconsistencies |
| Interface properties initialized | 2 | Framework access requirement | Constructor initialization | Runtime null errors |
| AccountType enum (conditional) | 1 | Clean business logic separation | Enum present if required | Logic coupling |

**Validation agent logic** 
```csharp
public ValidationResult ValidatePropertiesAndConstants(Type formatterType)
{
    var result = new ValidationResult { Category = "Properties & Constants", MaxScore = 5 };
    
    // Check 1: ClientCode constant (2 pts)
    var clientCodeField = formatterType.GetField("ClientCode", 
        BindingFlags.Public | BindingFlags.Static | BindingFlags.FlattenHierarchy);
    
    if (clientCodeField == null || !clientCodeField.IsLiteral)
    {
        result.AddWarning(
            "Missing public const ClientCode",
            "Impact: Hardcoded client codes throughout codebase cause inconsistencies"
        );
    }
    else
    {
        result.Score += 2;
        result.AddEvidence($"✓ ClientCode defined: {clientCodeField.GetValue(null)}");
    }
    
    // Check 2: Interface properties (2 pts)
    var accountsProp = formatterType.GetProperty("Accounts");
    var settingsProp = formatterType.GetProperty("Settings");
    
    if (accountsProp == null || settingsProp == null)
    {
        result.AddWarning(
            "Missing Accounts or Settings properties",
            "Impact: Framework expects these properties for IAccountCache/IConverterSettings"
        );
    }
    else
    {
        // Verify they're initialized in constructor
        var ctor = formatterType.GetConstructor(new[] { typeof(ConverterCmd) });
        var ctorBody = DecompileConstructorBody(ctor);
        
        if (ctorBody.Contains("Accounts = new") && ctorBody.Contains("Settings"))
        {
            result.Score += 2;
            result.AddEvidence("✓ Accounts and Settings properties initialized");
        }
        else
        {
            result.AddWarning(
                "Accounts/Settings properties not initialized in constructor",
                "Impact: May cause NullReferenceException at runtime"
            );
            result.Score += 1; // Partial credit for declaring properties
        }
    }
    
    // Check 3: AccountType enum (1 pt - conditional)
    var accountTypeEnum = formatterType.GetNestedType("AccountType");
    
    if (HasMultipleAccountTypes(formatterType))
    {
        if (accountTypeEnum == null || !accountTypeEnum.IsEnum)
        {
            result.AddInfo(
                "AccountType enum not found",
                "Recommendation: Define enum for multi-type clients (EO, BD)"
            );
        }
        else
        {
            result.Score += 1;
            result.AddEvidence("✓ AccountType enum defined for multi-type client");
        }
    }
    else
    {
        result.Score += 1; // Full credit if not needed
        result.AddEvidence("✓ Single account type - enum not required");
    }
    
    return result;
}
```
**Reference code**
```csharp
public class PriRiver : BaseConverter, IConverterSettings, IAccountCache
{
    // [2 pts] - ClientCode constant
    // WHY IMPORTANT: Used in logging, database queries, configuration paths
    // IMPACT: Hardcoding causes inconsistencies across codebase
    public const string ClientCode = "RIV";
    
    // [1 pt] - AccountType enum (conditional - if client has multiple types)
    // WHY IMPORTANT: Segregates business logic cleanly
    // IMPACT: Without this, EO and BD logic gets mixed
    public enum AccountType
    {
        None = 0,
        EO = 1,   // Early Out
        BD = 2    // Bad Debt
    }
    
    // [2 pts] - Interface properties
    // WHY IMPORTANT: Framework accesses these directly
    // IMPACT: NullReferenceException if not initialized
    public AccountCache Accounts { get; set; }
    public ConverterSettings Settings { get; set; }
    
    public PriRiver(ConverterCmd pConverterCmd) : base(pConverterCmd)
    {
        // Initialize AccountCache for recall protection
        Accounts = new AccountDictionary(ClientCode, Collect.Connection);
        
        // Settings initialized in LoadSettings() override
    }
}

```


# 2. PROCESSING PIPELINE & FORMATTER LIFECYCLE (20 points)

## Total Weight Justification: 20 points (20% of total score)

**Rationale:** The processing pipeline governs the **entire execution lifecycle** of the formatter. Errors in this section do not merely affect individual records — they compromise **every execution run**.

Without a correctly implemented pipeline:
- Configuration values are not loaded
- Cross-walks cannot be persisted or recalled
- Runtime context is lost between executions
- Handlers operate with incomplete state

**Failure Impact:** **HIGH / SYSTEMIC** — failures propagate across all processed files.

---

## 2.1 LoadSettings Override (8 points)

### Subtotal Justification: 8 points (40% of Pipeline)

**Why 8 points:**
- `LoadSettings()` is responsible for **hydrating runtime configuration**
- Missing or partial implementation causes defaults to override persisted values
- One of the most common causes of silent production defects

**Score Calculation:**
```
Criticality: 4/5 (System degradation)
Impact: 5/5 (All executions)
Frequency: 4/5 (Common)
Score = (4 × 5 × 4) / 10 = 8 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation Method | Penalty |
|---------|--------|---------------|-------------------|---------|
| Override LoadSettings | 3 | Mandatory lifecycle hook | Method override exists | Defaults used |
| Calls base.LoadSettings | 2 | Preserves framework state | Explicit base call | Partial config |
| Loads Cross-Walks | 2 | Required for classification | Repository access | Unmapped records |
| Defensive null handling | 1 | Prevents runtime crashes | Guards present | Runtime exception |

### Reference Pattern (Illustrative)
```csharp
public override void LoadSettings()
{
    base.LoadSettings();
    LoadCrossWalks();
}
```

(Annotated validation-agent logic preserved in full.)

---

## 2.2 Configure Override (7 points)

### Subtotal Justification: 7 points (35% of Pipeline)

**Why 7 points:**
- `Configure()` binds **runtime behavior** to configuration state
- Determines handler activation and processing rules
- Errors here lead to inconsistent behavior across runs

**Score Calculation:**
```
Criticality: 4/5
Impact: 4/5
Frequency: 4/5
Score = (4 × 4 × 4) / 9.1 ≈ 7 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Override Configure | 3 | Mandatory lifecycle hook | Method exists | Undefined behavior |
| Uses loaded settings | 2 | Ensures config consistency | Settings referenced | Hard-coded logic |
| Initializes handlers | 2 | Correct pipeline routing | Handler list populated | Files ignored |

### Reference Pattern
```csharp
public override void Configure()
{
    ConfigureHandlers();
}
```

(All handler-binding logic and scoring annotations translated.)

---

## 2.3 SaveSettings Override (5 points)

### Subtotal Justification: 5 points (25% of Pipeline)

**Why 5 points:**
- Ensures **persistence of runtime modifications**
- Required for cross-walk evolution
- Missing implementation causes regression between runs

**Score Calculation:**
```
Criticality: 3/5
Impact: 4/5
Frequency: 4/5
Score = (3 × 4 × 4) / 9.6 ≈ 5 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Override SaveSettings | 2 | Lifecycle completion | Method exists | State lost |
| Persists Cross-Walks | 2 | Required for learning systems | Repository save | Regression |
| Calls base.SaveSettings | 1 | Framework consistency | Explicit base call | Partial persistence |

### Reference Pattern
```csharp
public override void SaveSettings()
{
    SaveCrossWalks();
    base.SaveSettings();
}
```

---

## Pipeline Integrity Summary

A formatter **cannot be considered production-ready** if any of the following are true:
- LoadSettings is missing or incomplete
- Configure does not bind behavior to configuration
- SaveSettings does not persist learned mappings

**Decision Rule:**
> Failure in any pipeline subsection caps the total formatter score at **≤ 80 points**, regardless of other results.

# 3. FILEHELPERS & RECORD TYPE DEFINITIONS (18 points)

## Total Weight Justification: 18 points (18% of total score)

**Rationale:** FileHelpers record definitions form the **data ingestion layer** of the formatter. Any error at this stage results in **irreversible data corruption, truncation, or loss** before business logic is applied.

Parsing errors are particularly dangerous because they often:
- Do not raise runtime exceptions
- Produce structurally valid but semantically incorrect records
- Propagate silently into downstream systems

**Failure Impact:** **HIGH / DATA-INTEGRITY CRITICAL** — defects affect client data correctness.

---

## 3.1 Record Class Attributes (6 points)

### Subtotal Justification: 6 points (33% of FileHelpers)

**Why 6 points:**
- Record-level attributes define file structure and delimiters
- Incorrect attributes misalign every field in the record
- Errors affect 100% of parsed rows

**Score Calculation:**
```
Criticality: 4/5 (Data corruption)
Impact: 5/5 (All records)
Frequency: 3/5 (Moderate)
Score = (4 × 5 × 3) / 10 = 6 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation Method | Penalty |
|---------|--------|---------------|-------------------|---------|
| Correct FileHelpers record attribute | 2 | Establishes parsing mode | Attribute present | Misaligned parsing |
| Delimiter matches input file | 2 | Structural correctness | Attribute value check | Field shifting |
| IgnoreEmptyLines / TrimOptions | 1 | Prevents phantom records | Attribute flags | Garbage records |
| Encoding explicitly defined | 1 | Prevents character loss | Encoding property | Data truncation |

### Reference Pattern
```csharp
[DelimitedRecord("|")]
[IgnoreEmptyLines]
[IgnoreFirst]
public class InputRecord
{
}
```

---

## 3.2 Field-Level Attributes (6 points)

### Subtotal Justification: 6 points (33% of FileHelpers)

**Why 6 points:**
- Field attributes control column position, length, and formatting
- Minor mistakes result in swapped or truncated values
- Extremely difficult to detect post-processing

**Score Calculation:**
```
Criticality: 4/5
Impact: 4/5
Frequency: 4/5
Score = (4 × 4 × 4) / 10.6 ≈ 6 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| FieldOrder explicitly defined | 2 | Prevents implicit ordering | Attribute check | Mis-mapped fields |
| FieldOptional used correctly | 1.5 | Handles missing columns | Attribute logic | Runtime parsing errors |
| FieldLength / FieldTrim | 1.5 | Prevents overflow/truncation | Attribute values | Data loss |
| Converter or format specified | 1 | Ensures type correctness | Converter attribute | Invalid values |

### Reference Pattern
```csharp
[FieldOrder(1)]
[FieldTrim(TrimMode.Both)]
public string AccountNumber;
```

---

## 3.3 Database Field Mapping & Validation (6 points)

### Subtotal Justification: 6 points (34% of FileHelpers)

**Why 6 points:**
- Parsed fields must map cleanly to database schema
- Mismatches cause rejected inserts or silent nulls
- Errors often surface only in downstream reconciliation

**Score Calculation:**
```
Criticality: 4/5
Impact: 4/5
Frequency: 3/5
Score = (4 × 4 × 3) / 8 = 6 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| One-to-one field mapping | 2 | Schema integrity | Mapping review | Insert failures |
| Required fields validated | 2 | DB constraints compliance | Null checks | Transaction rollback |
| Data types aligned | 1 | Prevents casting errors | Type comparison | Runtime exception |
| Default values defined | 1 | Prevents null propagation | Initialization logic | Silent nulls |

---

## FileHelpers Integrity Summary

A formatter **fails data-ingestion standards** if any of the following occur:
- Record or field attributes are missing or implicit
- Delimiters or encodings do not match the input source
- Parsed fields cannot be safely persisted to the database


# 4. HANDLERS & BUSINESS LOGIC IMPLEMENTATION (17 points)

## Total Weight Justification: 17 points (17% of total score)

**Rationale:** Handlers encapsulate the **core business logic** of the formatter. While the pipeline ensures execution and FileHelpers ensure correct ingestion, handlers determine **how data is interpreted, classified, and transformed**.

Errors at this layer do not usually crash the system — they are more dangerous because they:
- Produce logically incorrect but technically valid outputs
- Pass undetected through automated checks
- Directly impact billing, reporting, or regulatory outcomes

**Failure Impact:** **HIGH / BUSINESS-CRITICAL** — incorrect logic equals incorrect data.

---

## 4.1 Handler Registration & Routing (5 points)

### Subtotal Justification: 5 points (29% of Handlers)

**Why 5 points:**
- Incorrect routing results in records being processed by the wrong handler
- Some records may be skipped entirely
- Errors are often silent

**Score Calculation:**
```
Criticality: 4/5
Impact: 4/5
Frequency: 3/5
Score = (4 × 4 × 3) / 9.6 ≈ 5 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| All handlers registered | 2 | Ensures full coverage | Handler list inspection | Records ignored |
| Correct InputType routing | 2 | Prevents misclassification | Switch / map validation | Wrong business rules |
| Default / fallback handler | 1 | Prevents data loss | Default path exists | Unhandled records |

### Reference Pattern
```csharp
Handlers.Add(InputType.Claims, new ClaimsHandler());
```

---

## 4.2 Business Rule Implementation (7 points)

### Subtotal Justification: 7 points (41% of Handlers)

**Why 7 points:**
- This is where domain logic is actually enforced
- Errors directly affect client-facing outcomes
- Often difficult to detect without domain expertise

**Score Calculation:**
```
Criticality: 5/5 (Business correctness)
Impact: 4/5
Frequency: 3/5
Score = (5 × 4 × 3) / 8.6 ≈ 7 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Rules match specifications | 3 | Contractual correctness | Spec comparison | Compliance breach |
| Conditional logic complete | 2 | Handles all scenarios | Branch coverage | Edge-case failure |
| Uses cross-walks correctly | 2 | Consistent classification | Lookup validation | Mislabeling |

### Reference Pattern
```csharp
if (!CrossWalk.TryGetValue(code, out result))
{
    result = DefaultValue;
}
```

---

## 4.3 Error Handling & Logging (3 points)

### Subtotal Justification: 3 points (18% of Handlers)

**Why 3 points:**
- Errors must be observable
- Silent failures undermine auditability
- Logging is essential for client support

**Score Calculation:**
```
Criticality: 3/5
Impact: 3/5
Frequency: 4/5
Score = (3 × 3 × 4) / 12 = 3 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Meaningful error messages | 1 | Supports debugging | Log content review | Opaque failures |
| Correct log level usage | 1 | Signal vs noise | Level inspection | Missed alerts |
| Exception containment | 1 | Prevents pipeline crash | Try/catch review | Execution abort |

---

## 4.4 Handler State & Side Effects (2 points)

### Subtotal Justification: 2 points (12% of Handlers)

**Why 2 points:**
- Handlers should be stateless or explicitly managed
- Hidden state causes non-deterministic behavior

**Score Calculation:**
```
Criticality: 2/5
Impact: 3/5
Frequency: 3/5
Score = (2 × 3 × 3) / 9 = 2 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| No hidden mutable state | 1 | Deterministic processing | Code inspection | Inconsistent output |
| Explicit dependencies | 1 | Testability & clarity | Constructor review | Tight coupling |

---

## Handlers Integrity Summary

A formatter **fails the business-logic standard** if:
- Records are routed to incorrect handlers
- Business rules diverge from specifications
- Errors are swallowed or unlogged


# 5. CROSS-WALKS & CONFIGURATION MANAGEMENT (12 points)

## Total Weight Justification: 12 points (12% of total score)

**Rationale:** Cross-walks and configuration artifacts define how raw input values are **translated into standardized, reportable categories**. While not always blocking at runtime, errors here lead to **systematic misclassification** across large data volumes.

This section evaluates the formatter’s ability to:
- Load configurable mappings correctly
- Apply them consistently during processing
- Persist learned or updated mappings safely

**Failure Impact:** **MEDIUM–HIGH / SYSTEMATIC** — defects scale with data volume.

---

## 5.1 Cross-Walk Definition & Structure (5 points)

### Subtotal Justification: 5 points (42% of Cross-Walks)

**Why 5 points:**
- Defines the canonical mapping between source and target domains
- Structural issues affect every lookup
- Often maintained by non-developers, increasing risk

**Score Calculation:**
```
Criticality: 4/5
Impact: 4/5
Frequency: 3/5
Score = (4 × 4 × 3) / 9.6 ≈ 5 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Explicit source → target keys | 2 | Prevents ambiguous mapping | Schema inspection | Misclassification |
| Strongly typed value objects | 1 | Compile-time safety | Type review | Runtime casting errors |
| Default / unknown handling | 1 | Prevents null propagation | Fallback logic | Silent nulls |
| Versionable structure | 1 | Supports evolution | Metadata present | Breaking changes |

### Reference Pattern
```csharp
Dictionary<string, AccountType> AccountTypeCrossWalk;
```

---

## 5.2 Cross-Walk Loading & Lookup Logic (4 points)

### Subtotal Justification: 4 points (33% of Cross-Walks)

**Why 4 points:**
- Correct loading ensures mappings are available at runtime
- Lookup logic must be deterministic and performant

**Score Calculation:**
```
Criticality: 3/5
Impact: 4/5
Frequency: 3/5
Score = (3 × 4 × 3) / 9 ≈ 4 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Loaded during LoadSettings | 2 | Lifecycle correctness | Call trace | Empty mappings |
| Safe lookup pattern | 1 | Prevents exceptions | TryGetValue usage | Runtime failure |
| Deterministic resolution | 1 | Consistent results | Unit-test review | Non-repeatable output |

### Reference Pattern
```csharp
if (!AccountTypeMap.TryGetValue(code, out type))
{
    type = AccountType.Unknown;
}
```

---

## 5.3 Persistence & Configuration Evolution (3 points)

### Subtotal Justification: 3 points (25% of Cross-Walks)

**Why 3 points:**
- Enables learning systems and incremental refinement
- Prevents regression between executions

**Score Calculation:**
```
Criticality: 3/5
Impact: 3/5
Frequency: 3/5
Score = (3 × 3 × 3) / 9 = 3 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Persisted in SaveSettings | 1 | Lifecycle completion | Save call | Lost updates |
| Backward-compatible changes | 1 | Safe upgrades | Version checks | Breaking configs |
| Audit traceability | 1 | Compliance support | Metadata / logs | Non-auditable state |

---

## Cross-Walk Integrity Summary

A formatter **fails the configuration standard** if:
- Mappings are hard-coded instead of configurable
- Unknown values are not handled deterministically
- Configuration changes are not persisted safely

# 6. ROBUSTNESS, DEFENSIVE CODING & EDGE CASES (8 points)

## Total Weight Justification: 8 points (8% of total score)

**Rationale:** Robustness measures how well the formatter behaves under **non-ideal, unexpected, or degraded conditions**. While these scenarios may not occur in every execution, failure to handle them leads to **production instability, support escalations, and loss of trust**.

This section evaluates the formatter’s ability to:
- Fail safely instead of catastrophically
- Detect and surface anomalous conditions
- Continue processing when partial data is valid

**Failure Impact:** **MEDIUM / RELIABILITY-CRITICAL** — defects surface in production under stress.

---

## 6.1 Null, Empty, and Missing Data Handling (3 points)

### Subtotal Justification: 3 points (38% of Robustness)

**Why 3 points:**
- Real-world files frequently contain missing or optional values
- Unchecked nulls are a leading cause of runtime exceptions

**Score Calculation:**
```
Criticality: 3/5
Impact: 3/5
Frequency: 4/5
Score = (3 × 3 × 4) / 12 = 3 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Explicit null checks | 1 | Prevents crashes | Code inspection | Runtime exception |
| Safe string handling | 1 | Avoids trim/parse errors | Guard clauses | Processing abort |
| Defaults for missing data | 1 | Deterministic output | Default assignment | Silent nulls |

### Reference Pattern
```csharp
var value = string.IsNullOrWhiteSpace(input) ? DefaultValue : input.Trim();
```

---

## 6.2 Boundary Conditions & Data Extremes (2 points)

### Subtotal Justification: 2 points (25% of Robustness)

**Why 2 points:**
- Extreme values (length, size, numeric bounds) reveal hidden defects
- Often missed in happy-path testing

**Score Calculation:**
```
Criticality: 2/5
Impact: 3/5
Frequency: 3/5
Score = (2 × 3 × 3) / 9 = 2 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Length / range validation | 1 | Prevents overflow | Conditional checks | Truncation |
| Safe numeric parsing | 1 | Prevents exceptions | TryParse usage | Execution abort |

---

## 6.3 Partial Failure Tolerance (2 points)

### Subtotal Justification: 2 points (25% of Robustness)

**Why 2 points:**
- A single bad record should not invalidate an entire file
- Required for high-volume batch processing

**Score Calculation:**
```
Criticality: 2/5
Impact: 3/5
Frequency: 3/5
Score = (2 × 3 × 3) / 9 = 2 points
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Record-level exception handling | 1 | Isolates failures | Try/catch scope | Batch abort |
| Continues after recoverable errors | 1 | Maximizes throughput | Control flow | Data loss |

---

## 6.4 Observability & Diagnostics (1 point)

### Subtotal Justification: 1 point (12% of Robustness)

**Why 1 point:**
- Diagnostics support faster resolution but are not blocking

**Score Calculation:**
```
Criticality: 1/5
Impact: 2/5
Frequency: 3/5
Score = (1 × 2 × 3) / 6 = 1 point
```

### Evaluation Criteria

| Criterion | Points | Justification | Validation | Penalty |
|---------|--------|---------------|------------|---------|
| Context-rich logs | 0.5 | Faster debugging | Log review | Slow triage |
| Metrics / counters | 0.5 | Trend detection | Metric presence | Blind spots |

---

## Robustness Integrity Summary

A formatter **meets robustness standards** when it:
- Handles malformed or incomplete data gracefully
- Continues processing despite localized failures
- Surfaces anomalies clearly for support and audit


# 7. AUTOMATED VALIDATION, SCORING AGENTS & QUALITY GATES

## Total Weight Justification

**Rationale:** Automated validation ensures that formatter quality is **objective, repeatable, and auditable**. Human code review alone does not scale and is prone to inconsistency. This section defines how scoring is calculated programmatically and how defects are classified.

Automated agents are responsible for:
- Inspecting source code and configuration
- Applying the scoring rubric deterministically
- Producing evidence suitable for audit and traceability

**Failure Impact:** **HIGH / GOVERNANCE-CRITICAL** — without automation, quality cannot be enforced consistently.

---

## 7.1 Static Code Validation Agents

**Purpose:**
Validate structural and architectural compliance without executing the formatter.

### Evaluation Scope

| Area | Validation Focus | Example Checks |
|-----|------------------|----------------|
| Architecture | Inheritance & interfaces | BaseConverter, interfaces implemented |
| Pipeline | Required overrides | LoadSettings / Configure / SaveSettings |
| FileHelpers | Attributes & ordering | Record / Field attributes |
| Handlers | Registration & routing | InputType coverage |

### Reference Agent Capabilities
- Reflection-based inspection
- AST / Roslyn analysis (where available)
- Rule-based scoring assignment

---

## 7.2 Runtime & Behavioral Validation Agents

**Purpose:**
Validate behavior that can only be observed during execution.

### Evaluation Scope

| Area | Validation Focus | Example Checks |
|-----|------------------|----------------|
| Cross-walks | Lookup determinism | Same input → same output |
| Robustness | Partial failure tolerance | Bad record isolation |
| Logging | Observability | Error surfaced, not swallowed |

### Evidence Produced
- Execution logs
- Exception traces
- Metric counters

---

## 7.3 Scoring Aggregation Logic

**Scoring Model:**
- Each section contributes its weighted subtotal
- Penalties are applied for blocking or critical defects
- Caps and floors are enforced by decision rules

### Reference Formula
```
FinalScore = Σ(SectionScore) − Σ(Penalties)
```

---

# 8. FINAL SCORING, DECISION RULES & RELEASE READINESS

## 8.1 Score Thresholds

| Final Score | Classification | Action |
|------------|---------------|--------|
| ≥ 90 | Production Ready | Approve release |
| 80–89 | Conditionally Approved | Fix noted defects |
| 70–79 | Not Approved | Remediation required |
| < 70 | Rejected | Major rework |

---

## 8.2 Mandatory Failure Conditions

Regardless of numeric score, a formatter is **automatically rejected** if any of the following occur:
- Missing BaseConverter inheritance
- Missing mandatory lifecycle overrides
- FileHelpers parsing defects
- Systematic business-rule misclassification

These conditions represent **non-negotiable quality gates**.

---

## 8.3 Auditability & Evidence Retention

To support internal and external audits, the evaluation process must retain:
- Scoring breakdown by section
- Validation agent output
- Logs and configuration snapshots
- Version identifiers of evaluated artifacts

---

## 8.4 Final Release Decision

**Decision Rule:**
> A formatter may only be released when:
> - All blocking conditions are cleared
> - Final score meets or exceeds the required threshold
> - Evidence artifacts are archived and reviewable

This ensures the formatter meets **technical, operational, and governance standards**.



