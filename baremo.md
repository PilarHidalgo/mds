
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

**Validation agent logic** 
```csharp
public ValidationResult ValidateLoadSettings(Type formatterType)
{
    var result = new ValidationResult 
    { 
        Category = "LoadSettings", 
        MaxScore = 8 
    };
    
    // Check 1: Override exists (3 pts)
    var method = formatterType.GetMethod("LoadSettings", 
        BindingFlags.NonPublic | BindingFlags.Instance);
    
    if (method == null || method.DeclaringType == typeof(BaseConverter))
    {
        result.AddCriticalError(
            "BLOCKER: LoadSettings() not overridden",
            "Impact: Cross-walks will not load, all transactions will be unmapped",
            "Fix: Add 'protected override void LoadSettings(ConverterCmd cmd)'"
        );
        return result; // Don't continue - nothing to validate
    }
    result.Score += 3;
    result.AddEvidence("✓ LoadSettings() override present");
    
    // Decompile method body for analysis
    var methodBody = DecompileMethodBody(method);
    
    // Check 2: TransCodeCrossWalkStore initialization (2.5 pts)
    if (!methodBody.Contains("TransCodeCrossWalkStore"))
    {
        result.AddCriticalError(
            "BLOCKER: TransCodeCrossWalkStore not initialized",
            "Impact: ALL transaction codes will be unmapped",
            "Error in production: 'Transaction code not found in cross-walk'",
            "Fix: Add 'Meditech.TransCodeCrossWalkStore = new TransactionCodeCrossWalkBase(...)'"
        );
    }
    else
    {
        // Verify it's using TransactionCodeCrossWalkBase
        if (methodBody.Contains("new TransactionCodeCrossWalkBase"))
        {
            result.Score += 2.5;
            result.AddEvidence("✓ TransCodeCrossWalkStore initialized correctly");
        }
        else
        {
            result.AddWarning(
                "TransCodeCrossWalkStore initialized but not using TransactionCodeCrossWalkBase",
                "May cause compatibility issues with framework"
            );
            result.Score += 1.5; // Partial credit
        }
    }
    
    // Check 3: FinancialClassCrossWalkStore initialization (2.5 pts)
    if (!methodBody.Contains("FinancialClassCrossWalkStore"))
    {
        result.AddCriticalError(
            "BLOCKER: FinancialClassCrossWalkStore not initialized",
            "Impact: Financial classes won't map to MDS codes (SP, COM, MCR, MCD)",
            "Business Impact: Accounts assigned to wrong collection queues",
            "Fix: Add 'Meditech.FinancialClassCrossWalkStore = new MiscCodeCrossWalkBase(...)'"
        );
    }
    else
    {
        if (methodBody.Contains("new MiscCodeCrossWalkBase"))
        {
            result.Score += 2.5;
            result.AddEvidence("✓ FinancialClassCrossWalkStore initialized correctly");
        }
        else
        {
            result.AddWarning(
                "FinancialClassCrossWalkStore initialized but not using MiscCodeCrossWalkBase"
            );
            result.Score += 1.5; // Partial credit
        }
    }
    
    return result;
}
```
**Reference code**
```csharp
// [3 pts] - Override declaration
protected override void LoadSettings(ConverterCmd pConverterCmd)
{
    // WHY CRITICAL: This is called once at the start of each formatter execution
    // IMPACT: Without this override, cross-walks are never loaded
    // FREQUENCY: Very common error - MVP formatters often missing this
    
    // [2.5 pts] - Transaction codes initialization
    // WHY CRITICAL: Every transaction in demographic files needs code mapping
    // EXAMPLES: "CHG" → Charge, "PSP" → Payment Self Pay, "PMCR" → Payment Medicare
    // IMPACT: Without this, ALL transactions fail validation at Cupload insert
    // ERROR MESSAGE: "Transaction code 'CHG' not found in cross-walk"
    Meditech.Meditech.TransCodeCrossWalkStore = new TransactionCodeCrossWalkBase(
        pConverterCmd.ProfileID,      // Links to client-specific configuration
        "TransactionCodes",            // XML node name in config file
        GetConfigFileSpec(),           // Path: C:\MDS\Configs\ClientXXX.xml
        TransactionCodeItems           // Static array defined in this class
    );
    
    // [2.5 pts] - Financial class initialization
    // WHY CRITICAL: Determines queue assignment (SP, COM, MCR, MCD)
    // BUSINESS IMPACT: Wrong queue = wrong collection strategy = revenue loss
    // EXAMPLES: "SP" → Self Pay, "BC" → Blue Cross, "MCR" → Medicare
    // IMPACT: Accounts classified incorrectly, sent to wrong collectors
    Meditech.Meditech.FinancialClassCrossWalkStore = new MiscCodeCrossWalkBase(
        pConverterCmd.ProfileID,
        "FinancialClasses",
        GetConfigFileSpec(),
        FinancialClassItems
    );
    
    // Initialize Settings object (not scored separately, but required)
    Settings = new ConverterSettings(GetConfigFileSpec(), pConverterCmd);
}
```

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

**Validation agent logic** 
```csharp
public ValidationResult ValidateConfigure(Type formatterType)
{
    var result = new ValidationResult 
    { 
        Category = "Configure", 
        MaxScore = 7 
    };
    
    var method = formatterType.GetMethod("Configure", 
        BindingFlags.NonPublic | BindingFlags.Instance);
    
    if (method == null)
    {
        result.AddWarning(
            "Configure() not overridden - using base implementation",
            "Impact: Default configuration UI only, may miss client-specific needs"
        );
        return result;
    }
    
    var methodBody = DecompileMethodBody(method);
    
    // Check 1: BaseConfigurationForm usage (3 pts)
    if (!methodBody.Contains("BaseConfigurationForm"))
    {
        // Check if using custom Windows Forms (anti-pattern)
        if (methodBody.Contains("new Form") || methodBody.Contains(": Form"))
        {
            result.AddError(
                "Using custom Windows Forms instead of BaseConfigurationForm",
                "Impact: Inconsistent UI, potential bugs in persistence",
                "MVP Feedback: Custom forms had 'confusing variable naming'",
                "Fix: Use BaseConfigurationForm for standard UI"
            );
        }
        else
        {
            result.AddWarning("Cannot determine configuration form type");
        }
    }
    else
    {
        result.Score += 3;
        result.AddEvidence("✓ Using BaseConfigurationForm (standard UI)");
    }
    
    // Check 2: Transaction Codes tab (2 pts)
    bool hasTransCodesTab = 
        methodBody.Contains("TransCodeBrowseControl") || 
        (methodBody.Contains("AddCrossWalk") && methodBody.Contains("Transaction"));
    
    if (!hasTransCodesTab)
    {
        result.AddCriticalError(
            "Transaction Codes cross-walk tab missing",
            "Impact: Users (Mitch/Shawna) cannot configure transaction code mappings",
            "Result: Formatter unusable without developer intervention",
            "Fix: Add 'ConfigForm.AddCrossWalk(TransCodes, \"Transaction Codes\")'"
        );
    }
    else
    {
        result.Score += 2;
        result.AddEvidence("✓ Transaction Codes cross-walk tab present");
    }
    
    // Check 3: Financial Classes tab (2 pts)
    bool hasFinClassesTab = 
        methodBody.Contains("MiscCodeBrowseControl") || 
        (methodBody.Contains("AddCrossWalk") && methodBody.Contains("Financial"));
    
    if (!hasFinClassesTab)
    {
        result.AddCriticalError(
            "Financial Classes cross-walk tab missing",
            "Impact: Users cannot configure financial class mappings",
            "Business Impact: Account classification will fail",
            "Fix: Add 'ConfigForm.AddCrossWalk(MiscCodes, \"Financial Classes\")'"
        );
    }
    else
    {
        result.Score += 2;
        result.AddEvidence("✓ Financial Classes cross-walk tab present");
    }
    
    return result;
}
```
**Reference code**
```csharp
// [7 pts total] - Configure override
protected override bool Configure(ConverterCmd pConverterCmd)
{
    // [3 pts] - Using BaseConfigurationForm (not custom Windows Forms)
    // WHY IMPORTANT: Standard UI, logging, validation, persistence all built-in
    // ANTI-PATTERN: Creating custom Form causes bugs (per MVP feedback)
    // MVP ISSUE: "Confusing variable naming" in custom UIs
    BaseConfigurationForm ConfigForm = new BaseConfigurationForm(pConverterCmd);
    
    // [2 pts] - Transaction Codes cross-walk tab
    // WHO USES: Mitch/Shawna during client setup
    // WHAT IT DOES: Allows mapping client codes → MDS standard codes
    // EXAMPLE: Client code "CHG001" → MDS code "C" (Charge)
    // WITHOUT THIS: Developers must manually edit XML files (not scalable)
    TransCodeBrowseControl TransCodes = new TransCodeBrowseControl(
        pConverterCmd, 
        TransTypeSelections // Array of P, A, I, C types
    );
    ConfigForm.AddCrossWalk(TransCodes, "Transaction Codes");
    
    // [2 pts] - Financial Classes cross-walk tab
    // WHO USES: Mitch/Shawna during client setup
    // WHAT IT DOES: Maps client financial classes → MDS queues (SP, COM, MCR, MCD)
    // BUSINESS IMPACT: Determines collection strategy per account
    // EXAMPLE: Client "BLUECROSS" → MDS "BC" (Commercial)
    MiscCodeBrowseControl MiscCodes = new MiscCodeBrowseControl(pConverterCmd);
    ConfigForm.AddCrossWalk(MiscCodes, "Financial Classes");
    
    // Show configuration dialog to user
    return Configure(pConverterCmd, ConfigForm);
}
```

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

**Validation agent logic** 
```csharp
public ValidationResult ValidateSaveSettings(Type formatterType)
{
    var result = new ValidationResult 
    { 
        Category = "SaveSettings", 
        MaxScore = 5 
    };
    
    // Check 1: Override exists (2 pts)
    var method = formatterType.GetMethod("SaveSettings", 
        BindingFlags.NonPublic | BindingFlags.Instance);
    
    if (method == null || method.DeclaringType == typeof(BaseConverter))
    {
        result.AddWarning(
            "SaveSettings() not overridden",
            "Impact: Configuration changes will not persist between executions",
            "User Impact: Must reconfigure every time",
            "Fix: Add 'protected override void SaveSettings()'"
        );
        return result;
    }
    result.Score += 2;
    result.AddEvidence("✓ SaveSettings() override present");
    
    var methodBody = DecompileMethodBody(method);
    
    // Check 2: TransCodeCrossWalkStore saved (1.5 pts)
    if (!methodBody.Contains("TransCodeCrossWalkStore") || !methodBody.Contains(".Save()"))
    {
        result.AddWarning(
            "TransCodeCrossWalkStore not saved",
            "Impact: Transaction code mappings will be lost after Configure dialog closes"
        );
    }
    else
    {
        result.Score += 1.5;
        result.AddEvidence("✓ TransCodeCrossWalkStore.Save() called");
    }
    
    // Check 3: FinancialClassCrossWalkStore saved (1.5 pts)
    if (!methodBody.Contains("FinancialClassCrossWalkStore") || !methodBody.Contains(".Save()"))
    {
        result.AddWarning(
            "FinancialClassCrossWalkStore not saved",
            "Impact: Financial class mappings will be lost after Configure dialog closes"
        );
    }
    else
    {
        result.Score += 1.5;
        result.AddEvidence("✓ FinancialClassCrossWalkStore.Save() called");
    }
    
    return result;
}
```
**Reference code**
```csharp
// [2 pts] - SaveSettings override
protected override void SaveSettings()
{
    // WHY IMPORTANT: Called after user clicks "Save" in Configure dialog
    // IMPACT: Without this, all configuration changes are lost
    // USER IMPACT: Mitch/Shawna must reconfigure every time
    
    // [1.5 pts] - Save transaction codes
    // PERSISTS TO: C:\MDS\Configs\ClientXXX.xml (TransactionCodes node)
    if (Meditech.Meditech.TransCodeCrossWalkStore != null)
    {
        Meditech.Meditech.TransCodeCrossWalkStore.Save();
    }
    
    // [1.5 pts] - Save financial classes
    // PERSISTS TO: C:\MDS\Configs\ClientXXX.xml (FinancialClasses node)
    if (Meditech.Meditech.FinancialClassCrossWalkStore != null)
    {
        Meditech.Meditech.FinancialClassCrossWalkStore.Save();
    }
    
    // Save other settings (if any)
    if (Settings != null)
    {
        Settings.Save();
    }
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

**Validation agent logic** 
```csharp
public ValidationResult ValidateRecordTypeClassAttributes(Type recordType)
{
    var result = new ValidationResult 
    { 
        Category = "Record Type Class Attributes", 
        MaxScore = 6 
    };
    
    // Check 1: DelimitedRecord or FixedLengthRecord (3 pts)
    var delimitedAttr = recordType.GetCustomAttribute<DelimitedRecordAttribute>();
    var fixedLengthAttr = recordType.GetCustomAttribute<FixedLengthRecordAttribute>();
    
    if (delimitedAttr == null && fixedLengthAttr == null)
    {
        result.AddCriticalError(
            "BLOCKER: Missing [DelimitedRecord] or [FixedLengthRecord] attribute",
            "Impact: FileHelpers cannot parse the file",
            "Exception: 'Type must have DelimitedRecord or FixedLengthRecord attribute'",
            "Fix: Add [DelimitedRecord(\"\\t\")] or [FixedLengthRecord] to class"
        );
        return result; // Don't continue - this is blocker
    }
    
    result.Score += 3;
    if (delimitedAttr != null)
    {
        result.AddEvidence($"✓ [DelimitedRecord] with delimiter: '{delimitedAttr.Delimiter}'");
    }
    else
    {
        result.AddEvidence($"✓ [FixedLengthRecord] format");
    }
    
    // Check 2: IgnoreEmptyLines (2 pts) - only for delimited files
    if (delimitedAttr != null)
    {
        var ignoreEmptyAttr = recordType.GetCustomAttribute<IgnoreEmptyLinesAttribute>();
        
        if (ignoreEmptyAttr == null)
        {
            result.AddWarning(
                "Missing [IgnoreEmptyLines] attribute",
                "Impact: Blank lines in file will cause parsing errors",
                "Common in medical exports: Meditech includes blank lines every 100 records",
                "Fix: Add [IgnoreEmptyLines] to class"
            );
        }
        else
        {
            result.Score += 2;
            result.AddEvidence("✓ [IgnoreEmptyLines] present - handles blank lines");
        }
    }
    else
    {
        // Fixed-length files don't need IgnoreEmptyLines (full credit)
        result.Score += 2;
        result.AddEvidence("✓ Fixed-length format - blank line handling not needed");
    }
    
    // Check 3: Sealed class (1 pt)
    if (!recordType.IsSealed)
    {
        result.AddInfo(
            "Class not sealed",
            "Best Practice: Record types should be sealed (prevents inheritance)",
            "Fix: Change 'public class' to 'public sealed class'"
        );
    }
    else
    {
        result.Score += 1;
        result.AddEvidence("✓ Class is sealed (best practice)");
    }
    
    return result;
}
```
**Reference code**
```csharp
// [3 pts] - DelimitedRecord attribute (CRITICAL)
// WHY CRITICAL: Tells FileHelpers this is a tab-delimited file
// ALTERNATIVES: [FixedLengthRecord] for fixed-width files
// IMPACT: Without this, FileHelpers cannot parse the file at all
// EXCEPTION: "Type must have DelimitedRecord or FixedLengthRecord attribute"
[DelimitedRecord("\t")]  // Tab-delimited (common in medical exports)
// [DelimitedRecord(",")]  // CSV
// [DelimitedRecord("|")]  // Pipe-delimited
// [FixedLengthRecord()]   // Fixed-width format

// [2 pts] - IgnoreEmptyLines attribute (IMPORTANT)
// WHY IMPORTANT: Medical system exports often have blank lines between records
// REAL EXAMPLE: Meditech exports include blank line after every 100 records
// WITHOUT THIS: FileHelpers tries to parse blank line →
// WITHOUT THIS: FileHelpers tries to parse blank line → "Line too short" error
// IMPACT: Data loss - processing stops at first blank line
[IgnoreEmptyLines]

// [1 pt] - Sealed class (BEST PRACTICE)
// WHY SEALED: Record types are DTOs (Data Transfer Objects), not meant for inheritance
// BENEFIT: Prevents future developers from incorrectly extending this class
// PERFORMANCE: Minor performance improvement (virtual call optimization)
public sealed class InventoryRecordType
{
    // Field definitions...
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


**Validation agent logic** 
```csharp
public ValidationResult ValidateRecordTypeFieldAttributes(Type recordType)
{
    var result = new ValidationResult 
    { 
        Category = "Field Attributes", 
        MaxScore = 6 
    };
    
    var fields = recordType.GetFields(BindingFlags.Public | BindingFlags.Instance);
    
    // Scoring accumulators
    double lengthAttributeScore = 0;
    double trimAttributeScore = 0;
    double converterScore = 0;
    double nullValueScore = 0;
    
    foreach (var field in fields)
    {
        // Check 1: FieldQuoted/FieldFixedLength (max 2 pts total)
        bool hasLengthAttribute = 
            field.GetCustomAttribute<FieldQuotedAttribute>() != null ||
            field.GetCustomAttribute<FieldFixedLengthAttribute>() != null;
        
        if (!hasLengthAttribute)
        {
            result.AddWarning(
                $"Field '{field.Name}' missing length attribute",
                "Expected: [FieldQuoted] for delimited or [FieldFixedLength] for fixed-width"
            );
        }
        else
        {
            // Award 0.1 pt per field, up to max 2 pts
            lengthAttributeScore += Math.Min(0.1, 2.0 - lengthAttributeScore);
        }
        
        // Check 2: FieldTrim for string fields (max 1.5 pts total)
        if (field.FieldType == typeof(string))
        {
            var trimAttr = field.GetCustomAttribute<FieldTrimAttribute>();
            
            if (trimAttr == null)
            {
                result.AddWarning(
                    $"String field '{field.Name}' missing [FieldTrim]",
                    "Impact: Spaces will cause lookup failures",
                    "Example: 'SMITH ' won't match 'SMITH' in database"
                );
            }
            else if (trimAttr.TrimMode != TrimMode.Both)
            {
                result.AddWarning(
                    $"Field '{field.Name}' uses TrimMode.{trimAttr.TrimMode}",
                    "Recommendation: Use TrimMode.Both for safety"
                );
                trimAttributeScore += Math.Min(0.05, 1.5 - trimAttributeScore); // Partial credit
            }
            else
            {
                // Award 0.075 pt per string field, up to max 1.5 pts
                trimAttributeScore += Math.Min(0.075, 1.5 - trimAttributeScore);
            }
        }
        
        // Check 3: FieldConverter for date/decimal fields (max 1.5 pts total)
        if (field.FieldType == typeof(DateTime) || 
            field.FieldType == typeof(DateTime?) ||
            field.FieldType == typeof(decimal) || 
            field.FieldType == typeof(decimal?))
        {
            var converterAttr = field.GetCustomAttribute<FieldConverterAttribute>();
            
            if (converterAttr == null)
            {
                result.AddCriticalError(
                    $"CRITICAL: Field '{field.Name}' ({field.FieldType.Name}) missing [FieldConverter]",
                    "Impact: Parsing will fail or produce incorrect values",
                    field.FieldType == typeof(decimal) || field.FieldType == typeof(decimal?) 
                        ? "Medical systems often store amounts without decimal point (12345 = $123.45)"
                        : "Medical systems use non-standard date formats (20231215 = 2023-12-15)",
                    $"Fix: Add [FieldConverter(typeof({GetRecommendedConverter(field.FieldType)}))]"
                );
            }
            else
            {
                // Validate correct converter type
                Type expectedConverter = GetExpectedConverterType(field.FieldType);
                
                if (converterAttr.Converter == expectedConverter ||
                    IsAcceptableConverter(converterAttr.Converter, field.FieldType))
                {
                    // Award 0.15 pt per special field, up to max 1.5 pts
                    converterScore += Math.Min(0.15, 1.5 - converterScore);
                }
                else
                {
                    result.AddWarning(
                        $"Field '{field.Name}' uses unexpected converter: {converterAttr.Converter.Name}",
                        $"Expected: {expectedConverter.Name} or compatible"
                    );
                    converterScore += Math.Min(0.08, 1.5 - converterScore); // Partial credit
                }
            }
        }
        
        // Check 4: FieldNullValue for nullable/optional fields (max 1 pt total)
        if (IsNullableOrOptional(field.FieldType))
        {
            var nullValueAttr = field.GetCustomAttribute<FieldNullValueAttribute>();
            
            if (nullValueAttr != null)
            {
                // Award 0.05 pt per field, up to max 1 pt
                nullValueScore += Math.Min(0.05, 1.0 - nullValueScore);
            }
            // Note: Not warning if missing - this is optional best practice
        }
    }
    
    // Add accumulated scores
    result.Score += lengthAttributeScore;
    result.Score += trimAttributeScore;
    result.Score += converterScore;
    result.Score += nullValueScore;
    
    // Add summary evidence
    result.AddEvidence($"✓ Length attributes: {lengthAttributeScore:F2}/2.0 pts");
    result.AddEvidence($"✓ Trim attributes: {trimAttributeScore:F2}/1.5 pts");
    result.AddEvidence($"✓ Converters: {converterScore:F2}/1.5 pts");
    result.AddEvidence($"✓ Null values: {nullValueScore:F2}/1.0 pts");
    
    return result;
}

private string GetRecommendedConverter(Type fieldType)
{
    if (fieldType == typeof(decimal) || fieldType == typeof(decimal?))
        return "MDSDecimalConverter";
    if (fieldType == typeof(DateTime) || fieldType == typeof(DateTime?))
        return "MDSDateConverter or ClarioneseDateConverter";
    return "unknown";
}

private Type GetExpectedConverterType(Type fieldType)
{
    if (fieldType == typeof(decimal) || fieldType == typeof(decimal?))
        return typeof(MDSDecimalConverter);
    if (fieldType == typeof(DateTime) || fieldType == typeof(DateTime?))
        return typeof(MDSDateConverter);
    return null;
}

private bool IsAcceptableConverter(Type converterType, Type fieldType)
{
    // Accept multiple valid converters for DateTime
    if (fieldType == typeof(DateTime) || fieldType == typeof(DateTime?))
    {
        return converterType == typeof(MDSDateConverter) ||
               converterType == typeof(ClarioneseDateConverter);
    }
    return false;
}

private bool IsNullableOrOptional(Type fieldType)
{
    return Nullable.GetUnderlyingType(fieldType) != null || 
           !fieldType.IsValueType;
}
```
**Reference code**
```csharp
[IgnoreEmptyLines]
[DelimitedRecord("\t")]
public sealed class InventoryRecordType
{
    // FIELD 1: Account Number (string field example)
    
    // [0.1 pt] - FieldQuoted (handles quotes in CSV)
    // WHY: If account number contains delimiter (rare but possible)
    // EXAMPLE: "ACC,123" in CSV would break without QuoteMode
    [FieldQuoted(QuoteMode.OptionalForRead)]
    
    // [0.075 pt] - FieldTrim (removes spaces)
    // WHY CRITICAL: DB lookup for "ACC123 " won't find "ACC123"
    // REAL IMPACT: Account not found → creates duplicate instead of update
    // MEDITECH ISSUE: Exports add trailing spaces to fixed-width fields
    [FieldTrim(TrimMode.Both)]
    public string AccountNum;
    
    // FIELD 2: Bill Balance (decimal field example)
    
    [FieldQuoted(QuoteMode.OptionalForRead)]
    [FieldTrim(TrimMode.Both)]
    
    // [0.15 pt] - FieldConverter for decimal (CRITICAL)
    // WHY CRITICAL: Medical systems store amounts without decimal point
    // EXAMPLE: "12345" in file represents $123.45
    // WITHOUT THIS: Would parse as 12345.00 (100x error!)
    // MDSDecimalConverter divides by 100
    [FieldConverter(typeof(MDSDecimalConverter))]
    
    // [0.05 pt] - FieldNullValue (handles missing values)
    // WHY: Empty balance in file should default to 0, not null
    // WITHOUT THIS: NullReferenceException when summing balances
    [FieldNullValue(typeof(decimal), "0")]
    public decimal BillBal;
    
    // FIELD 3: Service Date (DateTime field example)
    
    [FieldQuoted(QuoteMode.OptionalForRead)]
    [FieldTrim(TrimMode.Both)]
    
    // [0.15 pt] - FieldConverter for DateTime (CRITICAL)
    // WHY CRITICAL: Medical systems use non-standard date formats
    // MEDITECH: "20231215" (yyyyMMdd) - no separators
    // CLARION: Custom format with different separators
    // WITHOUT THIS: FormatException "String was not recognized as valid DateTime"
    [FieldConverter(typeof(MDSDateConverter))]
    // OR for Clarion systems:
    // [FieldConverter(typeof(ClarioneseDateConverter))]
    
    // [0.05 pt] - FieldNullValue for optional dates
    // WHY: Missing date should default to MinValue or specific sentinel
    [FieldNullValue(typeof(DateTime), "1900-01-01")]
    public DateTime ServiceDate;
    
    // FIELD 4: Transaction Code (string, simple)
    
    [FieldQuoted(QuoteMode.OptionalForRead)]
    [FieldTrim(TrimMode.Both)]
    public string TransCode; // No converter needed - simple string
    
    // FIELD 5: Financial Class (string with special handling)
    
    [FieldQuoted(QuoteMode.OptionalForRead)]
    [FieldTrim(TrimMode.Both)]
    
    // Optional: FieldNullValue for empty financial class
    [FieldNullValue(typeof(string), "SP")] // Default to Self Pay if empty
    public string FinClass;
}
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

**Validation agent logic** 
```csharp
// Simplified Cupload schema for validation
public static class CuploadSchema
{
    public static readonly TableSchema MasterTable = new TableSchema
    {
        Name = "uplmaster",
        Columns = new[]
        {
            new Column("accountnumber", typeof(string), 20, required: true),
            new Column("membercode", typeof(string), 3, required: true),
            new Column("loadexe", typeof(string), 50, required: true),
            new Column("accountbalance", typeof(decimal)),
            new Column("patientname", typeof(string), 50),
            new Column("acctbaseclass", typeof(string), 10),
            new Column("originalcreditor", typeof(string), 50),
            new Column("ssn", typeof(string), 11),
            new Column("dateofbirth", typeof(DateTime)),
            // ... 51 more columns (total 60)
        }
    };
    
    public static readonly TableSchema TransTable = new TableSchema
    {
        Name = "upltrans",
        Columns = new[]
        {
            new Column("accountnumber", typeof(string), 20, required: true),
            new Column("transactioncode", typeof(string), 10, required: true),
            new Column("transactiondate", typeof(DateTime)),
            new Column("transactionamount", typeof(decimal)),
            new Column("transactiontype", typeof(string), 1),
            // ... more transaction columns
        }
    };
    
    public static readonly TableSchema InsuranceTable = new TableSchema
    {
        Name = "uplinsurance",
        Columns = new[]
        {
            new Column("accountnumber", typeof(string), 20, required: true),
            new Column("insurancename", typeof(string), 50),
            new Column("policynumber", typeof(string), 30),
            new Column("groupnumber", typeof(string), 30),
            new Column("insuredname", typeof(string), 50),
            // ... more insurance columns
        }
    };
}
```
**Reference code**
```csharp
[IgnoreEmptyLines]
[DelimitedRecord("\t")]
public sealed class InventoryRecordType
{
    // MASTER TABLE FIELDS (uplmaster - 60 columns)
    
    // [0.2 pt] - Account Number (PRIMARY KEY)
    // SCHEMA: uplmaster.accountnumber VARCHAR(20) NOT NULL
    // WHY CRITICAL: Primary key for account lookup
    [FieldQuoted(QuoteMode.OptionalForRead)]
    [FieldTrim(TrimMode.Both)]
    [FieldMapping("accountnumber", TableDestination.Master)]
    public string AccountNum;
    
    // [0.2 pt] - Member Code (REQUIRED)
    // SCHEMA: uplmaster.membercode VARCHAR(3) NOT NULL
    // WHY: Links to client in MDS system
    [FieldQuoted(QuoteMode.OptionalForRead)]
    [FieldTrim(TrimMode.Both)]
    [FieldMapping("membercode", TableDestination.Master)]
    public string MemberCode;
    
    // [0.2 pt] - Account Balance
    // SCHEMA: uplmaster.accountbalance DECIMAL(18,2)
    // NOTE: TableDestination.Master is correct for demographics/inventory
    [FieldQuoted(QuoteMode.OptionalForRead)]
    [FieldTrim(TrimMode.Both)]
    [FieldConverter(typeof(MDSDecimalConverter))]
    [FieldNullValue(typeof(decimal), "0")]
    [FieldMapping("accountbalance", TableDestination.Master)]
    public decimal BillBal;
    
    // [0.2 pt] - Patient Name
    // SCHEMA: uplmaster.patientname VARCHAR(50)
    [FieldQuoted(QuoteMode.OptionalForRead)]
    [FieldTrim(TrimMode.Both)]
    [FieldMapping("patientname", TableDestination.Master)]
    public string PatientName;
    
    // [0.2 pt] - Financial Class
    // SCHEMA: uplmaster.acctbaseclass VARCHAR(10)
    // WRONG DESTINATION EXAMPLE: TableDestination.Trans would corrupt data!
    [FieldQuoted(QuoteMode.OptionalForRead)]
    [FieldTrim(TrimMode.Both)]
    [FieldMapping("acctbaseclass", TableDestination.Master)]
    public string FinClass;
    
    // TRANSACTION TABLE FIELDS (upltrans - if this were a transaction file)
    // NOTE: This example shows WRONG usage - transactions should be in separate handler
    
    // ANTI-PATTERN (DO NOT DO THIS):
    // [FieldMapping("transactioncode", TableDestination.Trans)] // WRONG!
    // This field is in InventoryRecordType which maps to Master table
    // Transactions should be in separate TransactionRecordType
    
    // INSURANCE TABLE FIELDS (uplinsurance)
    // Similar principle - insurance data needs separate record type
    
    // UNMAPPED FIELD EXAMPLE (client-specific, not in Cupload schema)
    
    // [0 pts] - Not mapped (intentionally)
    // WHY: Client has custom field not in MDS schema
    // IMPACT: Data parsed but not persisted (acceptable if documented)
    [FieldQuoted(QuoteMode.OptionalForRead)]
    [FieldTrim(TrimMode.Both)]
    // NO [FieldMapping] attribute - this field is ignored
    public string ClientCustomField;
}
```



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

**Validation agent logic** 
```csharp
public ValidationResult ValidateDemographicsHandler(Type handlerType)
{
    var result = new ValidationResult 
    { 
        Category = "DemographicsHandler", 
        MaxScore = 7 
    };
    
    // Check 1: Inheritance (3 pts)
    if (!handlerType.IsSubclassOf(typeof(MedBaseCollectionsHandler)))
    {
        // Check if using wrong base class
        if (handlerType.IsSubclassOf(typeof(BaseCollectionsHandler)))
        {
            result.AddCriticalError(
                "Handler inherits from BaseCollectionsHandler instead of MedBaseCollectionsHandler",
                "Impact: Missing 20+ Meditech-specific methods",
                "Examples of missing logic: cross-walk application, field defaults, validations",
                "Fix: Change inheritance to 'MedBaseCollectionsHandler'"
            );
            result.Score += 1.0; // Partial credit - at least it compiles
        }
        else
        {
            result.AddCriticalError(
                "BLOCKER: Handler does not inherit from MedBaseCollectionsHandler",
                "Expected: class DemographicsHandler : MedBaseCollectionsHandler",
                "Impact: Compilation errors or completely missing business logic"
            );
            return result; // Blocker - don't continue
        }
    }
    else
    {
        result.Score += 3.0;
        result.AddEvidence("✓ Inherits from MedBaseCollectionsHandler");
    }
    
    // Check 2: Constructor (1.5 pts)
    var ctor = handlerType.GetConstructor(new[] 
    { 
        typeof(BaseConverter), 
        typeof(ProcessFile) 
    });
    
    if (ctor == null)
    {
        result.AddCriticalError(
            "Constructor signature incorrect",
            "Expected: public DemographicsHandler(BaseConverter conv, ProcessFile file)",
            "Impact: Framework cannot instantiate handler via reflection",
            "Runtime error: 'No suitable constructor found'"
        );
    }
    else
    {
        result.Score += 1.5;
        result.AddEvidence("✓ Constructor signature correct");
    }
    
    // Check 3: Recall protection (2.5 pts)
    var processRecordMethod = handlerType.GetMethod("ProcessRecord", 
        BindingFlags.NonPublic | BindingFlags.Instance);
    
    if (processRecordMethod != null)
    {
        var methodBody = DecompileMethodBody(processRecordMethod);
        
        bool hasRecallCheck = 
            methodBody.Contains("IsRecalled") ||
            (methodBody.Contains("Accounts") && methodBody.Contains("recall"));
        
        if (!hasRecallCheck)
        {
            result.AddCriticalError(
                "CRITICAL: No recall protection check in ProcessRecord",
                "Business Rule: Recalled accounts must not be reactivated",
                "Legal Impact: Violates compliance requirements",
                "Example Code: if (converter.Accounts.IsRecalled(acctNum)) return;",
                "Fix: Add recall check before processing account"
            );
        }
        else
        {
            result.Score += 2.5;
            result.AddEvidence("✓ Recall protection check present");
        }
    }
    else
    {
        result.AddWarning(
            "Cannot validate ProcessRecord method - not found or not accessible"
        );
    }
    
    return result;
}
```
**Reference code**
```csharp
// [3 pts] - Correct inheritance
// WHY CRITICAL: MedBaseCollectionsHandler provides 20+ methods specific to Meditech
// INCLUDES: ApplyCrossWalks(), SetDefaults(), ValidateRequiredFields()
// MVP ERROR: Inheriting from generic BaseCollectionsHandler = missing logic
// IMPACT: Manual implementation of 500+ lines of standard logic
public class DemographicsHandler : MedBaseCollectionsHandler
{
    // [1.5 pts] - Constructor with correct signature
    // WHY: Framework instantiates handlers via reflection using this signature
    // PARAMETERS:
    //   - pBaseConverter: Reference to parent formatter (access to Settings, Accounts)
    //   - pInputFile: File being processed (name, path, metadata)
    // IMPACT: Wrong signature = reflection fails = runtime exception
    public DemographicsHandler(BaseConverter pBaseConverter, ProcessFile pInputFile)
        : base(pBaseConverter, pInputFile)
    {
        // Constructor body usually empty - base handles initialization
    }
    
    // Override of ProcessRecord (called for each line in file)
    protected override void ProcessRecord(object pRecord)
    {
        // Cast to specific record type
        var record = pRecord as InventoryRecordType;
        if (record == null) return;
        
        // [2.5 pts] - Recall protection check (CRITICAL BUSINESS RULE)
        // WHY CRITICAL: Legal/compliance requirement
        // SCENARIO: Account deleted due to dispute/bankruptcy/legal issue
        // RULE: Once recalled, account cannot be reactivated by new file uploads
        // WITHOUT THIS: System automatically reactivates recalled accounts = VIOLATION
        var converter = BaseConverter as PriRiver; // Cast to access Accounts
        
        if (converter?.Accounts != null)
        {
            // Check if account is in recall cache (deleted accounts)
            if (converter.Accounts.IsRecalled(record.AccountNum))
            {
                // Log and skip - do NOT process this account
                LogWarning($"Account {record.AccountNum} is recalled - skipping");
                return; // Critical - exit without processing
            }
        }
        
        // Continue with normal processing...
        // Apply cross-walks (financial class mapping)
        ApplyCrossWalks(record);
        
        // Set defaults for required fields
        SetMandatoryDefaults(record);
        
        // Insert or update in Cupload database
        SaveToDatabase(record);
    }
}
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

**Validation agent logic** 
```csharp
public ValidationResult ValidateTransactionHandler(Type handlerType)
{
    var result = new ValidationResult 
    { 
        Category = "TransactionHandler", 
        MaxScore = 5 
    };
    
    // Check 1: Inheritance (2 pts)
    if (!handlerType.IsSubclassOf(typeof(MedBaseTransactionHandler)))
    {
        result.AddCriticalError(
            "Handler does not inherit from MedBaseTransactionHandler",
            "Impact: Batch loading logic missing",
            "Fix: class TransactionHandler : MedBaseTransactionHandler"
        );
        return result;
    }
    result.Score += 2.0;
    result.AddEvidence("✓ Inherits from MedBaseTransactionHandler");
    
    // Check 2: Constructor (1 pt)
    var ctor = handlerType.GetConstructor(new[] 
    { 
        typeof(BaseConverter), 
        typeof(ProcessFile) 
    });
    
    if (ctor == null)
    {
        result.AddError("Constructor signature incorrect");
    }
    else
    {
        result.Score += 1.0;
        result.AddEvidence("✓ Constructor signature correct");
    }
    
    // Check 3: LoadTransactions implementation (2 pts)
    var method = handlerType.GetMethod("LoadTransactions", 
        BindingFlags.NonPublic | BindingFlags.Instance);
    
    if (method == null || method.DeclaringType != handlerType)
    {
        result.AddCriticalError(
            "CRITICAL: LoadTransactions() not implemented",
            "Impact: Transactions will NOT be loaded - complete data loss",
            "This is the only method that persists transactions to upltrans table",
            "Fix: Override 'protected override DataTable LoadTransactions()'"
        );
    }
    else
    {
        // Verify it returns DataTable
        if (method.ReturnType != typeof(DataTable))
        {
            result.AddError(
                "LoadTransactions() has wrong return type",
                $"Expected: DataTable, Found: {method.ReturnType.Name}"
            );
            result.Score += 1.0; // Partial credit
        }
        else
        {
            result.Score += 2.0;
            result.AddEvidence("✓ LoadTransactions() implemented correctly");
        }
    }
    
    return result;
}
```
**Reference code**
```csharp
// [2 pts] - Correct inheritance
// WHY: MedBaseTransactionHandler provides batch loading infrastructure
// INCLUDES: CreateTransactionDataTable(), ApplyTransCrossWalks(), BulkInsert()
public class TransactionHandler : MedBaseTransactionHandler
{
    // [1 pt] - Constructor
    public TransactionHandler(BaseConverter pBaseConverter, ProcessFile pInputFile)
        : base(pBaseConverter, pInputFile)
    {
    }
    
    // [2 pts] - LoadTransactions implementation (CRITICAL)
    // WHY CRITICAL: This is THE method that actually persists transactions
    // FLOW:
    //   1. Create DataTable with upltrans schema
    //   2. For each transaction in file, add row to DataTable
    //   3. Return DataTable → framework does bulk insert
    // WITHOUT THIS: Transactions parsed but never saved = complete data loss
    protected override DataTable LoadTransactions()
    {
        // Create DataTable matching upltrans schema
        DataTable dt = CreateTransactionDataTable();
        
        // Read transaction file (could be separate file or embedded in demographics)
        foreach (var record in ReadTransactionRecords())
        {
            // Apply cross-walk: client code → MDS standard code
            string mdsTransCode = ApplyTransCodeCrossWalk(record.TransCode);
            
            // Create row
            DataRow row = dt.NewRow();
            row["accountnumber"] = record.AccountNum;
            row["transactioncode"] = mdsTransCode;
            row["transactiondate"] = record.TransDate;
            row["transactionamount"] = record.TransAmount;
            row["transactiontype"] = DetermineTransType(mdsTransCode); // P, A, I, C
            
            dt.Rows.Add(row);
        }
        
        // Framework will bulk insert this DataTable into upltrans
        return dt;
    }
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

  **Validation agent logic** 
```csharp
public ValidationResult ValidateInventoryHandler(Type handlerType)
{
    var result = new ValidationResult 
    { 
        Category = "InventoryHandler", 
        MaxScore = 5 
    };
    
    // Check 1: Inheritance from generic base (2 pts)
    Type baseType = handlerType.BaseType;
    
    if (baseType == null || !baseType.IsGenericType)
    {
        result.AddError(markdown
        result.AddError(
            "Handler does not inherit from MedBaseGenericFileHandler<T>",
            "Impact: Must implement FileHelpers parsing manually (300+ lines)",
            "Fix: class InventoryHandler : MedBaseGenericFileHandler<InventoryRecordType>"
        );
        return result;
    }
    
    Type genericTypeDef = baseType.GetGenericTypeDefinition();
    if (genericTypeDef != typeof(MedBaseGenericFileHandler<>))
    {
        result.AddWarning(
            $"Handler inherits from unexpected base: {baseType.Name}",
            "Expected: MedBaseGenericFileHandler<RecordType>"
        );
        result.Score += 1.0; // Partial credit
    }
    else
    {
        result.Score += 2.0;
        Type recordType = baseType.GetGenericArguments()[0];
        result.AddEvidence($"✓ Inherits from MedBaseGenericFileHandler<{recordType.Name}>");
    }
    
    // Check 2: Constructor passes record type (1.5 pts)
    var ctors = handlerType.GetConstructors();
    bool hasCorrectCtor = false;
    
    foreach (var ctor in ctors)
    {
        var parameters = ctor.GetParameters();
        
        // Check for constructor with typeof() call in base()
        // This is validated by decompiling constructor body
        var ctorBody = DecompileConstructorBody(ctor);
        
        if (ctorBody.Contains("typeof(") && ctorBody.Contains("RecordType"))
        {
            hasCorrectCtor = true;
            break;
        }
    }
    
    if (!hasCorrectCtor)
    {
        result.AddWarning(
            "Constructor may not be passing record type to base",
            "Expected pattern: base(conv, file, typeof(InventoryRecordType))",
            "Impact: FileHelpers may not know which record structure to use"
        );
    }
    else
    {
        result.Score += 1.5;
        result.AddEvidence("✓ Constructor passes record type to base");
    }
    
    // Check 3: ProcessRecord implementation (1.5 pts)
    var method = handlerType.GetMethod("ProcessRecord", 
        BindingFlags.NonPublic | BindingFlags.Instance);
    
    if (method == null || method.DeclaringType != handlerType)
    {
        result.AddWarning(
            "ProcessRecord() not overridden",
            "Impact: Parsed records won't be processed or persisted",
            "Fix: Override 'protected override void ProcessRecord(object pRecord)'"
        );
    }
    else
    {
        result.Score += 1.5;
        result.AddEvidence("✓ ProcessRecord() implemented");
    }
    
    return result;
}
```
**Reference code**
```csharp
// [2 pts] - Inheritance with generic record type
// WHY: MedBaseGenericFileHandler<T> provides automatic FileHelpers parsing
// BENEFIT: Framework automatically parses file using InventoryRecordType attributes
// ALTERNATIVE: Manual parsing with StreamReader (300+ lines of code)
public class InventoryHandler : MedBaseGenericFileHandler<InventoryRecordType>
{
    // [1.5 pts] - Constructor passes record type to base
    // WHY: Tells FileHelpers engine which record structure to use
    // IMPACT: Without typeof(), FileHelpers doesn't know how to parse
    public InventoryHandler(
        BaseConverter pBaseConverter, 
        ProcessFile pInputFile, 
        Type pRecordType) // Must be typeof(InventoryRecordType)
        : base(pBaseConverter, pInputFile, pRecordType)
    {
    }
    
    // Simplified constructor that hardcodes record type (common pattern)
    public InventoryHandler(
        BaseConverter pBaseConverter, 
        ProcessFile pInputFile)
        : base(pBaseConverter, pInputFile, typeof(InventoryRecordType)) // [1.5 pts]
    {
    }
    
    // [1.5 pts] - ProcessRecord override
    // WHY: Called by framework for each parsed record
    // PARAMETERS: pRecord is already parsed InventoryRecordType object
    // RESPONSIBILITY: Business logic, cross-walks, persistence
    protected override void ProcessRecord(object pRecord)
    {
        var record = pRecord as InventoryRecordType;
        if (record == null) return;
        
        // Apply cross-walks (financial class mapping)
        var converter = BaseConverter as PriRiver;
        record.FinClass = converter?.ApplyFinancialClassCrossWalk(record.FinClass) ?? record.FinClass;
        
        // Business logic: calculate fields, apply defaults
        if (record.BillBal == 0 && record.PaidAmount > 0)
        {
            record.BillBal = record.PaidAmount; // Example business rule
        }
        
        // Persist to database
        SaveRecordToMaster(record);
    }
}
```



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

**Validation agent logic** 
```csharp
public ValidationResult ValidateTransactionCodeItems(Type formatterType)
{
    var result = new ValidationResult 
    { 
        Category = "Transaction Code Cross-Walk", 
        MaxScore = 6 
    };
    
    // Check 1: Array exists (2 pts)
    var field = formatterType.GetField("TransactionCodeItems", 
        BindingFlags.NonPublic | BindingFlags.Static);
    
    if (field == null)
    {
        result.AddCriticalError(
            "BLOCKER: TransactionCodeItems array not found",
            "Impact: NullReferenceException in LoadSettings()",
            "Formatter will crash on startup",
            "Fix: Add 'private static TransactionCodeItem[] TransactionCodeItems'"
        );
        return result; // Blocker
    }
    
    result.Score += 2.0;
    result.AddEvidence("✓ TransactionCodeItems array exists");
    
    // Get array value
    var arrayValue = field.GetValue(null) as Array;
    
    if (arrayValue == null)
    {
        result.AddCriticalError(
            "TransactionCodeItems is null",
            "Impact: Cross-walk cannot be configured"
        );
        return result;
    }
    
    // Check 2: Array has elements (2 pts)
    int itemCount = arrayValue.Length;
    
    if (itemCount == 0)
    {
        result.AddCriticalError(
            "TransactionCodeItems array is empty",
            "Impact: Users have no sample codes to configure",
            "Best Practice: Include 5-10 most common client transaction codes",
            "Fix: Add sample codes like 'CHG' (Charge), 'PSP' (Payment Self Pay), etc."
        );
    }
    else if (itemCount < 3)
    {
        result.AddWarning(
            $"TransactionCodeItems has only {itemCount} items",
            "Recommendation: Include at least 5-10 common codes for easier setup"
        );
        result.Score += 1.0; // Partial credit
    }
    else
    {
        result.Score += 2.0;
        result.AddEvidence($"✓ TransactionCodeItems has {itemCount} initial codes");
    }
    
    // Check 3: Structure validation (2 pts)
    int validItems = 0;
    int totalItems = itemCount;
    var structureErrors = new List<string>();
    
    for (int i = 0; i < arrayValue.Length; i++)
    {
        var item = arrayValue.GetValue(i);
        if (item == null)
        {
            structureErrors.Add($"Item {i}: null");
            continue;
        }
        
        // Validate ClientCode
        var clientCodeProp = item.GetType().GetProperty("ClientCode");
        var clientCode = clientCodeProp?.GetValue(item) as string;
        
        if (string.IsNullOrWhiteSpace(clientCode))
        {
            structureErrors.Add($"Item {i}: ClientCode is null or empty");
            continue;
        }
        
        // Validate TransType
        var transTypeProp = item.GetType().GetProperty("TransType");
        var transType = transTypeProp?.GetValue(item) as string;
        
        if (transType == null || !IsValidTransType(transType))
        {
            structureErrors.Add($"Item {i} ('{clientCode}'): Invalid TransType '{transType}' (must be P, A, I, or C)");
            continue;
        }
        
        validItems++;
    }
    
    if (validItems == 0 && totalItems > 0)
    {
        result.AddCriticalError(
            "All TransactionCodeItems have invalid structure",
            "Errors: " + string.Join("; ", structureErrors.Take(5)),
            "Fix: Ensure format: new TransactionCodeItem(\"CODE\", \"P\", \"Description\")"
        );
    }
    else if (validItems < totalItems)
    {
        result.AddWarning(
            $"Only {validItems}/{totalItems} items have valid structure",
            "First errors: " + string.Join("; ", structureErrors.Take(3))
        );
        // Partial credit based on percentage valid
        result.Score += 2.0 * (validItems / (double)totalItems);
    }
    else
    {
        result.Score += 2.0;
        result.AddEvidence($"✓ All {validItems} items have correct structure (ClientCode, TransType, Description)");
    }
    
    return result;
}

private bool IsValidTransType(string transType)
{
    return transType == "P" || // Payment
           transType == "A" || // Adjustment
           transType == "I" || // Info
           transType == "C";   // Charge
}
```
**Reference code**
```csharp
public class PriRiver : BaseConverter, IConverterSettings, IAccountCache
{
    // [2 pts] - Array declaration (CRITICAL)
    // WHY CRITICAL: This array is referenced by LoadSettings()
    // USED BY: TransactionCodeCrossWalkStore initialization
    // WITHOUT THIS: NullReferenceException when formatter starts
    // STRUCTURE: Array of TransactionCodeItem objects
    private static TransactionCodeItem[] TransactionCodeItems =
    {
        // [Each item contributes to 2 pts for "has initial elements"]
        // [Each item validates structure for 2 pts "correct structure"]
        
        // CHARGE transactions (TransType = "C")
        // WHY INCLUDE: Most common transaction type in medical billing
        // CLIENT CODES: These are examples from client's system
        new TransactionCodeItem(
            "CHG",        // [Structure check] Client code (what appears in their file)
            "C",          // [Structure check] Trans type: C = Charge
            "Charge"      // Description (for UI display)
        ),
        new TransactionCodeItem("CHGADJ", "C", "Charge Adjustment"),
        new TransactionCodeItem("CHRG", "C", "Charge Alternate Code"),
        
        // PAYMENT transactions (TransType = "P")
        // EXAMPLES: Self Pay, Insurance, Medicare, Medicaid
        new TransactionCodeItem(
            "PSP",        // Payment Self Pay
            "P",          // Trans type: P = Payment
            "Payment Self Pay"
        ),
        new TransactionCodeItem("PINS", "P", "Payment Insurance"),
        new TransactionCodeItem("PMCR", "P", "Payment Medicare"),
        new TransactionCodeItem("PMCD", "P", "Payment Medicaid"),
        
        // ADJUSTMENT transactions (TransType = "A")
        // EXAMPLES: Write-offs, contractual adjustments
        new TransactionCodeItem(
            "ADJWO",      // Adjustment Write Off
            "A",          // Trans type: A = Adjustment
            "Adjustment Write Off"
        ),
        new TransactionCodeItem("ADJCN", "A", "Contractual Adjustment"),
        
        // INFO transactions (TransType = "I")
        // EXAMPLES: Balance forward, informational only
        new TransactionCodeItem(
            "BALFWD",     // Balance Forward
            "I",          // Trans type: I = Info
            "Balance Forward"
        ),
        
        // ANTI-PATTERNS TO AVOID:
        // ❌ new TransactionCodeItem("", "C", "Empty Code") // Empty ClientCode
        // ❌ new TransactionCodeItem("CHG", "X", "Invalid") // Invalid TransType (not P/A/I/C)
        // ❌ new TransactionCodeItem(null, "C", "Null")     // Null ClientCode
    };
    
    // Similar pattern for Financial Class Items (see next section)
    private static MiscCodeItem[] FinancialClassItems = 
    {
        new MiscCodeItem("SP", "Self Pay"),
        new MiscCodeItem("BC", "Blue Cross"),
        new MiscCodeItem("MCR", "Medicare"),
        new MiscCodeItem("MCD", "Medicaid"),
        new MiscCodeItem("COM", "Commercial"),
        new MiscCodeItem("WC", "Workers Comp"),
        new MiscCodeItem("AUTO", "Auto Insurance"),
    };
}
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

**Validation agent logic** 
```csharp
public ValidationResult ValidateFinancialClassItems(Type formatterType)
{
    var result = new ValidationResult 
    { 
        Category = "Financial Class Cross-Walk", 
        MaxScore = 6 
    };
    
    // Check 1: Array exists (2 pts)
    var field = formatterType.GetField("FinancialClassItems", 
        BindingFlags.NonPublic | BindingFlags.Static);
    
    if (field == null)
    {
        result.AddCriticalError(
            "BLOCKER: FinancialClassItems array not found",
            "Impact: Financial class cross-walk cannot be configured",
            "Business Impact: Accounts cannot be assigned to correct collection queues",
            "Fix: Add 'private static MiscCodeItem[] FinancialClassItems'"
        );
        return result;
    }
    
    result.Score += 2.0;
    result.AddEvidence("✓ FinancialClassItems array exists");
    
    // Get array value
    var arrayValue = field.GetValue(null) as Array;
    
    if (arrayValue == null)
    {
        result.AddCriticalError(
            "FinancialClassItems is null",
            "Impact: Cross-walk initialization will fail"
        );
        return result;
    }
    
    // Check 2: Array has elements (2 pts)
    int itemCount = arrayValue.Length;
    
    if (itemCount == 0)
    {
        result.AddCriticalError(
            "FinancialClassItems array is empty",
            "Impact: No sample classes for users to configure",
            "Best Practice: Include common classes (SP, COM, MCR, MCD, WC, AUTO)",
            "Fix: Add sample codes for Self Pay, Commercial, Medicare, Medicaid"
        );
    }
    else if (itemCount < 4)
    {
        result.AddWarning(
            $"FinancialClassItems has only {itemCount} items",
            "Recommendation: Include at least SP, COM, MCR, MCD (4 main types)"
        );
        result.Score += 1.0; // Partial credit
    }
    else
    {
        result.Score += 2.0;
        result.AddEvidence($"✓ FinancialClassItems has {itemCount} initial codes");
        
        // Bonus validation: Check if common codes are present
        var commonCodes = new[] { "SP", "COM", "MCR", "MCD" };
        var presentCodes = new List<string>();
        
        for (int i = 0; i < arrayValue.Length; i++)
        {
            var item = arrayValue.GetValue(i);
            var clientCodeProp = item?.GetType().GetProperty("ClientCode");
            var clientCode = clientCodeProp?.GetValue(item) as string;
            
            if (commonCodes.Contains(clientCode?.ToUpper()))
            {
                presentCodes.Add(clientCode.ToUpper());
            }
        }
        
        if (presentCodes.Count >= 3)
        {
            result.AddEvidence($"✓ Includes {presentCodes.Count}/4 common codes: {string.Join(", ", presentCodes)}");
        }
    }
    
    // Check 3: Structure validation (2 pts)
    int validItems = 0;
    int totalItems = itemCount;
    var structureErrors = new List<string>();
    
    for (int i = 0; i < arrayValue.Length; i++)
    {
        var item = arrayValue.GetValue(i);
        if (item == null)
        {
            structureErrors.Add($"Item {i}: null");
            continue;
        }
        
        // Validate ClientCode
        var clientCodeProp = item.GetType().GetProperty("ClientCode");
        var clientCode = clientCodeProp?.GetValue(item) as string;
        
        if (string.IsNullOrWhiteSpace(clientCode))
        {
            structureErrors.Add($"Item {i}: ClientCode is null or empty");
            continue;
        }
        
        // Validate Description (optional but recommended)
        var descriptionProp = item.GetType().GetProperty("Description");
        var description = descriptionProp?.GetValue(item) as string;
        
        if (string.IsNullOrWhiteSpace(description))
        {
            structureErrors.Add($"Item {i} ('{clientCode}'): Description is empty (not critical but recommended)");
            // Don't fail validation for missing description
        }
        
        validItems++;
    }
    
    if (validItems == 0 && totalItems > 0)
    {
        result.AddCriticalError(
            "All FinancialClassItems have invalid structure",
            "Errors: " + string.Join("; ", structureErrors.Take(5)),
            "Fix: Ensure format: new MiscCodeItem(\"CODE\", \"Description\")"
        );
    }
    else if (validItems < totalItems)
    {
        result.AddWarning(
            $"Only {validItems}/{totalItems} items have valid ClientCode",
            "First errors: " + string.Join("; ", structureErrors.Take(3))
        );
        result.Score += 2.0 * (validItems / (double)totalItems);
    }
    else
    {
        result.Score += 2.0;
        result.AddEvidence($"✓ All {validItems} items have valid structure (ClientCode, Description)");
    }
    
    return result;
}
```

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

**Validation agent logic** 
```csharp
public ValidationResult ValidateNullSafety(Type handlerType)
{
    var result = new ValidationResult 
    { 
        Category = "Null Safety & Defensive Coding", 
        MaxScore = 4 
    };
    
    var processRecordMethod = handlerType.GetMethod("ProcessRecord", 
        BindingFlags.NonPublic | BindingFlags.Instance);
    
    if (processRecordMethod == null)
    {
        result.AddWarning("ProcessRecord method not found - cannot validate null safety");
        return result;
    }
    
    var methodBody = DecompileMethodBody(processRecordMethod);
    
    // Check 1: Null checks (2 pts)
    bool hasNullChecks = 
        methodBody.Contains("== null") || 
        methodBody.Contains("!= null") ||
        methodBody.Contains("?.") || // Null-conditional operator
        methodBody.Contains("??");   // Null-coalescing operator
    
    if (!hasNullChecks)
    {
        result.AddWarning(
            "No null checks detected in ProcessRecord",
            "Impact: NullReferenceException risk in production",
            "Recommendation: Add null checks for record, converter, settings"
        );
    }
    else
    {
        result.Score += 2.0;
        result.AddEvidence("✓ Null safety patterns detected");
    }
    
    // Check 2: Required field validation (2 pts)
    bool hasAccountNumValidation = 
        methodBody.Contains("AccountNum") && 
        (methodBody.Contains("IsNullOrEmpty") || methodBody.Contains("IsNullOrWhiteSpace"));
    
    bool hasMemberCodeValidation = 
        methodBody.Contains("MemberCode") && 
        (methodBody.Contains("IsNullOrEmpty") || methodBody.Contains("IsNullOrWhiteSpace"));
    
    if (!hasAccountNumValidation && !hasMemberCodeValidation)
    {
        result.AddWarning(
            "No validation for required fields (AccountNum, MemberCode)",
            "Impact: DB constraint violations will crash processing"
        );
    }
    else if (hasAccountNumValidation && hasMemberCodeValidation)
    {
        result.Score += 2.0;
        result.AddEvidence("✓ Required fields validated");
    }
    else
    {
        result.Score += 1.0; // Partial credit
        result.AddEvidence("✓ Partial required field validation");
    }
    
    return result;
}
```
**Reference code**
```csharp
protected override void ProcessRecord(object pRecord)
{
    // [1 pt] - Null check on record
    var record = pRecord as InventoryRecordType;
    if (record == null)
    {
        LogWarning("Received null record - skipping");
        return; // Safe exit
    }
    
    // [0.5 pt] - Validate required fields (AccountNumber)
    if (string.IsNullOrWhiteSpace(record.AccountNum))
    {
        LogError("Record missing AccountNumber - skipping");
        CurrentCount++; // Still count the record
        return;
    }
    
    // [0.5 pt] - Validate MemberCode
    if (string.IsNullOrWhiteSpace(record.MemberCode))
    {
        LogError($"Account {record.AccountNum} missing MemberCode - skipping");
        return;
    }
    
    // [0.5 pt] - Safe access to converter with null check
    var converter = BaseConverter as PriRmarkdown
    var converter = BaseConverter as PriRiver;
    if (converter == null)
    {
        LogError("Unable to cast BaseConverter to PriRiver");
        return;
    }
    
    // [0.5 pt] - Safe access to Settings
    if (converter.Settings == null)
    {
        LogError("Converter Settings is null - cannot process");
        return;
    }
    
    // Safe to proceed with processing
    ProcessRecordInternal(record, converter);
}
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

**Validation agent logic** 
```csharp
public ValidationResult ValidateErrorHandling(Type handlerType)
{
    var result = new ValidationResult 
    { 
        Category = "Error Handling & Logging", 
        MaxScore = 4 
    };
    
    var processRecordMethod = handlerType.GetMethod("ProcessRecord", 
        BindingFlags.NonPublic | BindingFlags.Instance);
    
    if (processRecordMethod == null)
    {
        result.AddWarning("ProcessRecord method not found");
        return result;
    }
    
    var methodBody = DecompileMethodBody(processRecordMethod);
    
    // Check 1: Try-catch blocks (2 pts)
    bool hasTryCatch = 
        methodBody.Contains("try") && 
        methodBody.Contains("catch");
    
    if (!hasTryCatch)
    {
        result.AddWarning(
            "No try-catch blocks detected",
            "Impact: Unhandled exceptions will stop batch processing",
            "Recommendation: Wrap DB operations in try-catch"
        );
    }
    else
    {
        result.Score += 2.0;
        result.AddEvidence("✓ Try-catch blocks present");
    }
    
    // Check 2: Framework logging (2 pts)
    bool usesFrameworkLogging = 
        methodBody.Contains("LogError") || 
        methodBody.Contains("LogWarning") || 
        methodBody.Contains("LogInfo");
    
    bool usesConsoleLogging = 
        methodBody.Contains("Console.WriteLine") || 
        methodBody.Contains("Console.Write");
    
    if (usesConsoleLogging && !usesFrameworkLogging)
    {
        result.AddCriticalError(
            "CRITICAL: Using Console.WriteLine instead of framework logging",
            "Impact: Logs not captured in production environment",
            "MVP Error: This was identified in prototype review",
            "Fix: Replace Console.WriteLine with LogError()/LogWarning()"
        );
    }
    else if (!usesFrameworkLogging)
    {
        result.AddWarning(
            "No logging detected",
            "Impact: Debugging production issues will be impossible"
        );
    }
    else
    {
        result.Score += 2.0;
        result.AddEvidence("✓ Uses framework logging (LogError/LogWarning)");
        
        if (usesConsoleLogging)
        {
            result.AddWarning(
                "Mix of Console.WriteLine and framework logging detected",
                "Recommendation: Use only framework logging for consistency"
            );
        }
    }
    
    return result;
}

```
**Reference code**
```csharp
protected override void ProcessRecord(object pRecord)
{
    var record = pRecord as InventoryRecordType;
    if (record == null) return;
    
    // [1 pt] - Try-catch around critical operations
    try
    {
        // Validate required fields
        if (string.IsNullOrWhiteSpace(record.AccountNum))
        {
            // [1 pt] - Use LogWarning (not Console.WriteLine)
            LogWarning($"Line {CurrentCount}: Missing AccountNumber - skipping record");
            return;
        }
        
        // Apply business logic
        ApplyCrossWalks(record);
        
        // [1 pt] - Try-catch specifically around DB operation
        try
        {
            SaveToDatabase(record);
        }
        catch (SqlException ex)
        {
            // [1 pt] - LogError with details
            LogError($"DB error saving account {record.AccountNum}: {ex.Message}");
            // Don't rethrow - continue processing other records
        }
    }
    catch (Exception ex)
    {
        // [1 pt] - Outer catch for unexpected errors
        LogError($"Unexpected error processing record {CurrentCount}: {ex.Message}");
        // Log but don't crash - allow processing to continue
    }
    finally
    {
        CurrentCount++; // Always increment counter
    }
}
```


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

```markdown
# PriRiver - River Medical Center Formatter

## Client Information
- **Client Name**: River Medical Center
- **Member Code**: RIV
- **Host System**: Meditech
- **Effective Date**: 2024-01-15
- **Contact**: Jane Smith (jane.smith@rivermedical.com)

## File Specifications

### Demographics/Inventory File
- **Filename Pattern**: `RIV_DEMO_*.txt`
- **Format**: Tab-delimited
- **Record Type**: `InventoryRecordType`
- **Expected Columns**: 25
- **Sample Location**: `\\mds-share\samples\River\demo_sample.txt`

### Transaction File (if separate)
- **Filename Pattern**: `RIV_TRANS_*.txt`
- **Format**: Tab-delimited
- **Record Type**: `TransactionRecordType`

## Cross-Walk Configuration

### Financial Class Mapping
Use ConverterManager UI to configure mappings:

1. Open ConverterManager
2. Select "River Medical" (RIV)
3. Navigate to "Financial Class Cross-Walks"
4. Map client codes to MDS queues:

Client Code → MDS Queue
SP → Self Pay BCBS → Commercial MEDICARE → Medicare MEDICAID → Medicaid WC → Workers Comp


### Transaction Code Mapping
Navigate to "Transaction Code Cross-Walks":

Client Code → Trans Type → Description
CHG → C → Charge PSP → P → Payment Self Pay PINS → P → Payment Insurance ADJWO → A → Adjustment Write Off


## Special Business Rules
- **Recall Protection**: Enabled (standard)
- **Bill Number**: Generated from Account Number
- **Custom Logic**: None

## Testing
1. Place test file in: `C:\MDS\Input\RIV\`
2. Run formatter via TaskLauncher
3. Verify output in: `C:\MDS\Output\RIV\`
4. Check logs: `C:\MDS\Logs\PriRiver_YYYYMMDD.log`

## Support
- **Developer**: [Your Name]
- **Date Created**: 2024-01-15
- **Last Updated**: 2024-01-15
```

**Validation agent logic** 
```csharp
public ValidationResult ValidateReadme(string formatterPath)
{
    var result = new ValidationResult 
    { 
        Category = "README Documentation", 
        MaxScore = 3 
    };
    
    string readmePath = Path.Combine(formatterPath, "README.md");
    
    // Check 1: README exists (1.5 pts)
    if (!File.Exists(readmePath))
    {
        result.AddWarning(
            "README.md not found",
            "Impact: Mitch/Shawna won't have configuration instructions",
            "Recommendation: Create README with client info, file specs, cross-walk setup"
        );
        return result;
    }
    
    result.Score += 0.5; // File exists
    
    string content = File.ReadAllText(readmePath);
    
    // Check required sections
    var requiredSections = new[]
    {
        "Client Information",
        "File Specifications",
        "Cross-Walk Configuration"
    };
    
    int sectionsFound = 0;
    foreach (var section in requiredSections)
    {
        if (content.Contains(section, StringComparison.OrdinalIgnoreCase))
        {
            sectionsFound++;
        }
    }
    
    if (sectionsFound < 2)
    {
        result.AddWarning(
            $"README missing key sections (found {sectionsFound}/3)",
            $"Expected: {string.Join(", ", requiredSections)}"
        );
        result.Score += 0.5; // Partial credit
    }
    else
    {
        result.Score += 1.0; // Full credit for structure
        result.AddEvidence($"✓ README has {sectionsFound}/3 required sections");
    }
    
    // Check 2: Cross-walk configuration documented (1.5 pts)
    bool hasCrossWalkDocs = 
        content.Contains("Cross-Walk", StringComparison.OrdinalIgnoreCase) &&
        (content.Contains("Financial Class") || content.Contains("Transaction Code"));
    
    bool hasExamples = 
        content.Contains("→") || // Arrow notation for mappings
        content.Contains("->") ||
        content.Contains("Client Code");
    
    if (!hasCrossWalkDocs)
    {
        result.AddWarning(
            "Cross-walk configuration not documented",
            "Impact: Users won't know how to configure mappings",
            "Recommendation: Add section explaining ConverterManager UI usage"
        );
    }
    else if (!hasExamples)
    {
        result.AddWarning(
            "Cross-walk documentation lacks examples",
            "Recommendation: Show sample mappings (SP → Self Pay, etc.)"
        );
        result.Score += 0.75; // Partial credit
    }
    else
    {
        result.Score += 1.5; // Full credit
        result.AddEvidence("✓ Cross-walk configuration documented with examples");
    }
    
    return result;
}
```
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

**Validation agent logic** 
```csharp
public ValidationResult ValidateCodeQuality(Type handlerType)
{
    var result = new ValidationResult 
    { 
        Category = "Code Comments & Naming", 
        MaxScore = 2 
    };
    
    // Check 1: XML comments (1 pt)
    var xmlDocs = handlerType.GetCustomAttributes<System.ComponentModel.DescriptionAttribute>();
    
    // Check if type has XML summary
    bool hasClassXmlComment = HasXmlComment(handlerType);
    
    var publicMethods = handlerType.GetMethods(BindingFlags.Public | BindingFlags.Instance)
        .Where(m => m.DeclaringType == handlerType);
    
    int methodsWithXml = 0;
    int totalPublicMethods = publicMethods.Count();
    
    foreach (var method in publicMethods)
    {
        if (HasXmlComment(method))
        {
            methodsWithXml++;
        }
    }
    
    if (hasClassXmlComment && methodsWithXml > 0)
    {
        result.Score += 1.0;
        result.AddEvidence($"✓ XML comments present (class + {methodsWithXml} methods)");
    }
    else if (hasClassXmlComment || methodsWithXml > 0)
    {
        result.Score += 0.5;
        result.AddEvidence("✓ Partial XML documentation");
    }
    else
    {
        result.AddWarning(
            "No XML comments detected",
            "Recommendation: Add /// <summary> to class and public methods"
        );
    }
    
    // Check 2: Variable naming (1 pt)
    // This is harder to validate automatically, so we check method names
    var methods = handlerType.GetMethods(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);
    
    int descriptiveNames = 0;
    int poorNames = 0;
    
    foreach (var method in methods)
    {
        if (method.DeclaringType != handlerType) continue;
        
        string name = method.Name;
        
        // Good: PascalCase, descriptive
        if (name.Length > 3 && char.IsUpper(name[0]))
        {
            descriptiveNames++;
        }
        // Bad: single letter, tmp, etc.
        else if (name.Length <= 2 || name.Contains("tmp", StringComparison.OrdinalIgnoreCase))
        {
            poorNames++;
        }
    }
    
    if (poorNames > 0)
    {
        result.AddWarning(
            $"Detected {poorNames} methods with poor naming",
            "MVP Feedback: 'Confusing variable naming'",
            "Recommendation: Use descriptive PascalCase names"
        );
        result.Score += 0.5;
    }
    else if (descriptiveNames > 0)
    {
        result.Score += 1.0;
        result.AddEvidence("✓ Descriptive naming conventions");
    }
    
    return result;
}

private bool HasXmlComment(MemberInfo member)
{
    // In real implementation, would parse XML documentation file
    // For now, simplified check
    return member.GetCustomAttributes().Any(a => 
        a.GetType().Name.Contains("Description") || 
        a.GetType().Name.Contains("Summary"));
}
```
### Reference code

```csharp
/// <summary>
/// Handles demographics/inventory file processing for River Medical Center.
/// Processes account information and applies Meditech-specific business rules.
/// </summary>
/// <remarks>
/// [1 pt for XML comments]
/// WHY: Provides IntelliSense documentation for other developers
/// INCLUDES: Summary, parameters, return values, exceptions
/// </remarks>
public class DemographicsHandler : MedBaseCollectionsHandler
{
    /// <summary>
    /// Initializes a new instance of the DemographicsHandler.
    /// </summary>
    /// <param name="pBaseConverter">Reference to parent formatter (PriRiver)</param>
    /// <param name="pInputFile">Input file metadata and path</param>
    public DemographicsHandler(BaseConverter pBaseConverter, ProcessFile pInputFile)
        : base(pBaseConverter, pInputFile)
    {
    }
    
    /// <summary>
    /// Processes a single demographic record from the input file.
    /// </summary>
    /// <param name="pRecord">Parsed record (InventoryRecordType)</param>
    protected override void ProcessRecord(object pRecord)
    {
        // [1 pt for descriptive naming]
        // GOOD: descriptive variable names
        var demographicRecord = pRecord as InventoryRecordType;
        var riverFormatter = BaseConverter as PriRiver;
        string accountNumber = demographicRecord?.AccountNum;
        
        // BAD (MVP error):
        // var r = pRecord as InventoryRecordType;
        // var c = BaseConverter as PriRiver;
        // string x = r?.AccountNum;
        
        // Validate required fields
        if (string.IsNullOrWhiteSpace(accountNumber))
        {
            LogWarning($"Line {CurrentCount}: Missing account number");
            return;
        }
        
        // Check recall status
        if (riverFormatter?.Accounts?.IsRecalled(accountNumber) == true)
        {
            LogInfo($"Account {accountNumber} is recalled - skipping");
            return;
        }
        
        // Process record...
    }
}
```


---

# 8. FINAL SCORING, DECISION RULES & RELEASE READINESS

## 8.1 Score Thresholds

```markdown

┌─────────────────────────────────────────────────────────────┐
│                    SCORING BREAKDOWN                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. PROJECT STRUCTURE (16 pts)                    ████████  │
│     1.1 .csproj File (8 pts)                                │
│     1.2 AssemblyInfo.cs (8 pts)                             │
│                                                             │
│  2. MAIN FORMATTER CLASS (35 pts)                ███████████│
│     2.1 Class Declaration (12 pts)                          │
│     2.2 Settings & Configuration (10 pts)                   │
│     2.3 LoadSettings Override (13 pts)                      │
│                                                             │
│  3. FILEHELPERS INTEGRATION (18 pts)             █████████  │
│     3.1 Record Type Class Attributes (6 pts)                │
│     3.2 Field Attributes (6 pts)                            │
│     3.3 Field Mapping to DB (6 pts)                         │
│                                                             │
│  4. HANDLERS & BUSINESS LOGIC (17 pts)           ████████   │
│     4.1 DemographicsHandler (7 pts)                         │
│     4.2 TransactionHandler (5 pts)                          │
│     4.3 InventoryHandler (5 pts)                            │
│                                                             │
│  5. CROSS-WALKS (12 pts)                         ██████     │
│     5.1 Transaction Code Items (6 pts)                      │
│     5.2 Financial Class Items (6 pts)                       │
│                                                             │
│  6. ROBUSTNESS & EDGE CASES (8 pts)              ████       │
│     6.1 Null Safety (4 pts)                                 │
│     6.2 Error Handling & Logging (4 pts)                    │
│                                                             │
│  7. DOCUMENTATION (5 pts)                        ██         │
│     7.1 README.md (3 pts)                                   │
│     7.2 Code Comments (2 pts)                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘

TOTAL: 100 points
```


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

## Validation template
```markdown
public class FormatterValidationReport
{
    public string FormatterName { get; set; }
    public DateTime ValidationDate { get; set; }
    public int TotalScore { get; set; }
    public int MaxScore { get; set; } = 100;
    public double Percentage => (TotalScore / (double)MaxScore) * 100;
    public string Tier => GetTier();
    
    public List<CategoryResult> CategoryResults { get; set; }
    public List<CriticalIssue> CriticalIssues { get; set; }
    public List<Warning> Warnings { get; set; }
    public List<string> Recommendations { get; set; }
    
    public string GetTier()
    {
        if (Percentage >= 90) return "🟢 PRODUCTION READY";
        if (Percentage >= 75) return "🟡 NEEDS MINOR FIXES";
        if (Percentage >= 60) return "🟠 NEEDS MODERATE FIXES";
        return "🔴 MAJOR REWORK REQUIRED";
    }
    
    public string GenerateMarkdownReport()
    {
        var sb = new StringBuilder();
        
        sb.AppendLine($"# Validation Report: {FormatterName}");
        sb.AppendLine($"**Date**: {ValidationDate:yyyy-MM-dd HH:mm:ss}");
        sb.AppendLine($"**Score**: {TotalScore}/{MaxScore} ({Percentage:F1}%)");
        sb.AppendLine($"**Tier**: {Tier}");
        sb.AppendLine();
        
        // Critical Issues
        if (CriticalIssues.Any())
        {
            sb.AppendLine("## 🔴 Critical Issues");
            sb.AppendLine();
            foreach (var issue in CriticalIssues)
            {
                sb.AppendLine($"### {issue.Title}");
                sb.AppendLine($"- **Impact**: {issue.Impact}");
                sb.AppendLine($"- **Fix**: {issue.Fix}");
                sb.AppendLine();
            }
        }
        
        // Warnings
        if (Warnings.Any())
        {
            sb.AppendLine("## ⚠️ Warnings");
            sb.AppendLine();
            foreach (var warning in Warnings)
            {
                sb.AppendLine($"- {warning.Message}");
            }
            sb.AppendLine();
        }
        
        // Category Breakdown
        sb.AppendLine("## 📊 Score Breakdown");
        sb.AppendLine();
        sb.AppendLine("| Category | Score | Max | % |");
        sb.AppendLine("|----------|-------|-----|---|");
        
        foreach (var category in CategoryResults)
        {
            double pct = (category.Score / category.MaxScore) * 100;
            string bar = GetProgressBar(pct);
            sb.AppendLine($"| {category.Name} | {category.Score:F1} | {category.MaxScore} | {bar} {pct:F1}% |");
        }
        
        sb.AppendLine();
        
        // Recommendations
        if (Recommendations.Any())
        {
            sb.AppendLine("## 💡 Recommendations");
            sb.AppendLine();
            foreach (var rec in Recommendations)
            {
                sb.AppendLine($"- {rec}");
            }
        }
        
        return sb.ToString();
    }
    
    private string GetProgressBar(double percentage)
    {
        int blocks = (int)(percentage / 10);
        return new string('█', blocks) + new string('░', 10 - blocks);
    }
}

```





