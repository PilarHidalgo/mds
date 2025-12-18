# Tabla de Baremos con Estándares y Justificación de Puntajes
## Sistema de Evaluación del Proyecto MDS Formatter Generator

---

## 📊 METODOLOGÍA DE PUNTUACIÓN

### Principios de Diseño del Sistema de Scoring

El sistema de puntuación está diseñado para sumar **exactamente 100 puntos**, distribuidos según la **criticidad operacional** de cada componente del formatter.

**Fórmula de Asignación de Puntaje:**
Puntaje de Categoría = (Criticidad × Impacto × Frecuencia de Error) / Factor de Normalización


Donde:
- **Criticidad**: ¿Qué tan grave es el fallo? (1=Warning, 5=Bloqueante)
- **Impacto**: ¿Cuántos componentes afecta? (1=Local, 5=Sistema completo)
- **Frecuencia**: ¿Qué tan común es este error en formatters reales? (1=Raro, 5=Muy común)

**Criterios de Diseño:**

1. **Criticidad Operacional**: Mayor puntaje a elementos que impactan directamente la funcionalidad core
2. **Impacto en Calidad de Datos**: Prioridad a validaciones que protegen integridad de datos PHI
3. **Conformidad con Framework**: Adherencia estricta a patrones MDS establecidos
4. **Mantenibilidad**: Código que facilita debugging, auditoría y evolución

---

## 🎯 DISTRIBUCIÓN TOTAL DE PUNTOS (100 puntos)

| Categoría | Puntos | % del Total | Justificación de Peso |
|-----------|--------|-------------|----------------------|
| **1. Arquitectura Base** | 25 | 25% | Sin arquitectura correcta, el formatter no es invocable por TaskLauncher |
| **2. Pipeline de Procesamiento** | 20 | 20% | Controla lifecycle completo - errores causan fallos en runtime |
| **3. FileHelpers / Record Types** | 18 | 18% | Parser de datos - errores causan pérdida o corrupción de datos del cliente |
| **4. Handlers** | 17 | 17% | Lógica de negocio principal - implementación incorrecta = datos incorrectos |
| **5. Cross-Walks y Configuración** | 12 | 12% | Mapeos de códigos - sin esto, transacciones y cuentas quedan sin clasificar |
| **6. Robustez y Edge Cases** | 8 | 8% | Previene errores silenciosos en producción - crítico para confiabilidad |
| **TOTAL** | **100** | **100%** | |

**Justificación de la Distribución:**

- **Arquitectura (25%)**: Mayor peso porque es pre-requisito para todo lo demás. Sin herencia correcta de `BaseConverter`, el formatter no funciona.
- **Pipeline (20%)**: Segundo mayor peso porque controla configuración y persistencia. Errores aquí afectan todas las ejecuciones.
- **FileHelpers (18%)**: Tercer peso porque el parsing incorrecto resulta en pérdida de datos del cliente.
- **Handlers (17%)**: Cuarto peso porque implementan la lógica de negocio específica.
- **Cross-Walks (12%)**: Menor peso relativo porque son configurables post-deployment.
- **Robustez (8%)**: Menor peso porque son optimizaciones, no requisitos bloqueantes.

---

# 1. ARQUITECTURA BASE Y CONFORMIDAD FRAMEWORK (25 puntos)

## 📌 Justificación del Peso Total: 25 puntos (25% del score)

**Razón:** La arquitectura base es el **fundamento contractual** entre el formatter y el framework MDS. Sin la estructura correcta:
- TaskLauncher no puede instanciar el formatter (runtime exception)
- El pipeline de configuración no funciona
- Cross-walks no se pueden persistir
- Recall protection no se puede implementar

**Impacto de Fallo:** BLOQUEANTE - El formatter no es ejecutable si falla esta sección.

**Evidencia de Criticidad del Contexto:**
> *"Major Issues: Handlers not inheriting from Meditech base classes"* (MDS Project Follow-UP)

---

## 1.1 Herencia Correcta de BaseConverter (10 puntos)

### Justificación del Subtotal: 10 puntos (40% de Arquitectura)

**Por qué 10 puntos:**
- Esta subsección valida el **contrato principal** con el framework
- Fallos aquí causan **compilation errors** o **runtime exceptions**
- Es el error más frecuente en el MVP (según feedback de MDS)

**Cálculo:**
```
Criticidad: 5/5 (BLOQUEANTE) 
Impacto: 5/5 (Afecta sistema completo) 
Frecuencia: 5/5 (Error común en MVP) 
Score = (5 × 5 × 5) / 12.5 = 10 puntos
```

| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| **Clase hereda de BaseConverter** | 4 | **40% del subtotal**<br>• CRÍTICO: Sin esta herencia, TaskLauncher no puede invocar el formatter<br>• El framework espera este contrato<br>• Causa: Runtime exception al intentar cargar el formatter | `class X : BaseConverter` presente en código | **BLOQUEANTE**: Formatter no ejecutable |
| **Constructor recibe ConverterCmd** | 3 | **30% del subtotal**<br>• CRÍTICO: ConverterCmd contiene ProfileID, paths, DB Connection<br>• Sin él no hay contexto de ejecución<br>• Todos los métodos del formatter dependen de este objeto | Constructor signature:<br>`public X(ConverterCmd cmd) : base(cmd)` | **BLOQUEANTE**: Runtime exception al instanciar |
| **Implementa IConverterSettings** | 1.5 | **15% del subtotal**<br>• IMPORTANTE: Permite configuración persistente de cross-walks<br>• Sin esto, cada ejecución requiere reconfiguración manual<br>• Impacto en UX del usuario final | Interface presente en declaración:<br>`class X : BaseConverter, IConverterSettings` | Pérdida de configuración entre ejecuciones |
| **Implementa IAccountCache** | 1.5 | **15% del subtotal**<br>• IMPORTANTE: Recall protection depende de esto<br>• Sin cache, cuentas eliminadas pueden recibir nuevas transacciones<br>• Violación de reglas de negocio médico | Interface presente +<br>propiedad `AccountCache Accounts` | Violación de reglas de negocio (recalled accounts) |

**Validación Automatizada (CODE VALIDATOR Agent):**

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

## 1.2 Implementación de Métodos Abstractos Obligatorios (10 puntos)
### Justificación del Subtotal: 10 puntos (40% de Arquitectura)
Por qué 10 puntos:

- GetConverter() y QualifyFile() son métodos abstractos en BaseConverter
- Sin implementación, el código no compila
- Son el punto de entrada al procesamiento de archivos
- Errores aquí resultan en archivos ignorados silenciosamente
  
Cálculo:

```Criticidad: 5/5 (Compilation error)
Impacto: 5/5 (Ningún archivo se procesa)
Frecuencia: 4/5 (Común en generación automática)
Score = (5 × 5 × 4) / 10 = 10 puntos
```

| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| GetConverter() implementado | 5 | 50% del subtotal<br>• CRÍTICO: Es el router que devuelve el handler apropiado<br>• Sin él, ningún archivo se procesa<br>• Llamado por el pipeline en ProcessFile()<br>• Método abstracto - obligatorio | Método existe, no es abstracto,<br>retorna BaseConversionClass<br>según InputType | BLOQUEANTE: Compilation error |
| GetConverter() cubre todos InputTypes | 2 | 20% del subtotal<br>• IMPORTANTE: Debe manejar todos los tipos de archivo del cliente<br>• InputTypes no manejados = archivos ignorados silenciosamente<br>• Común error: solo implementar Demographics, olvidar Inventory | Switch/if cubre todos los valores<br>retornados por QualifyFile() | Archivos del cliente no procesados sin error visible |
| QualifyFile() implementado | 3 | 30% del subtotal<br>• CRÍTICO: Clasifica archivos entrantes por nombre/contenido<br>• Sin él, todos los archivos son InputType.Unknown<br>• Método abstracto - obligatorio | Método existe,<br>retorna InputType<br>basado en análisis de archivo | BLOQUEANTE: Compilation error |

**Código de Referencia con Anotaciones:**

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

**Validación Automatizada:**

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


## 1.3 Propiedades y Constantes del Framework (5 puntos)
### Justificación del Subtotal: 5 puntos (20% de Arquitectura)

Por qué 5 puntos:

- Menor peso que subsecciones anteriores porque no son bloqueantes
- Sin embargo, su ausencia causa bugs silenciosos en producción
- Son best practices del framework MDS

Cálculo:
```
Criticidad: 3/5 (Bugs en producción, no bloqueante)
Impacto: 3/5 (Afecta logging y configuración)
Frecuencia: 4/5 (Frecuentemente olvidados)
Score = (3 × 3 × 4) / 7.2 = 5 puntos

```
| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| GetConverter() implementado | 5 | 50% del subtotal<br>• CRÍTICO: Es el router que devuelve el handler apropiado<br>• Sin él, ningún archivo se procesa<br>• Llamado por el pipeline en ProcessFile()<br>• Método abstracto - obligatorio | Método existe, no es abstracto,<br>retorna BaseConversionClass<br>según InputType | BLOQUEANTE: Compilation error |
| GetConverter() cubre todos InputTypes | 2 | 20% del subtotal<br>• IMPORTANTE: Debe manejar todos los tipos de archivo del cliente<br>• InputTypes no manejados = archivos ignorados silenciosamente<br>• Común error: solo implementar Demographics, olvidar Inventory | Switch/if cubre todos los valores<br>retornados por QualifyFile() | Archivos del cliente no procesados sin error visible |
| QualifyFile() implementado | 3 | 30% del subtotal<br>• CRÍTICO: Clasifica archivos entrantes por nombre/contenido<br>• Sin él, todos los archivos son InputType.Unknown<br>• Método abstracto - obligatorio | Método existe,<br>retorna InputType<br>basado en análisis de archivo | BLOQUEANTE: Compilation error |



**Código de Referencia:**

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
**Validación Automatizada:**

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

## 2. PIPELINE DE PROCESAMIENTO Y LIFECYCLE (20 puntos)
### 📌 Justificación del Peso Total: 20 puntos (20% del score)
Razón: El pipeline controla el lifecycle completo del formatter:

- Inicialización: LoadSettings() carga configuración y cross-walks
- Configuración: Configure() permite al usuario definir mapeos
- Persistencia: SaveSettings() guarda cambios

### Impacto de Fallo:
- Sin LoadSettings: Cross-walks no cargan → todas las transacciones sin mapear
- Sin Configure: Usuario no puede configurar el formatter
- Sin SaveSettings: Configuración se pierde entre ejecuciones

### Evidencia de Criticidad del Contexto:

"Missing LoadSettings() and required overrides" (MDS Project Follow-UP)

Cálculo:
```
Criticidad: 4/5 (Causa fallos en runtime)
Impacto: 5/5 (Afecta todas las ejecuciones)
Frecuencia: 5/5 (Error muy común en generación)
Score = (4 × 5 × 5) / 5 = 20 puntos
```

## 2.1 LoadSettings Override (8 puntos)
### Justificación del Subtotal: 8 puntos (40% de Pipeline)
Por qué 8 puntos:

- Es el punto de entrada del lifecycle
- Sin LoadSettings, los cross-walks nunca se cargan
- Resulta en 100% de transacciones sin mapear
- Error más común según feedback del MVP

Cálculo:
```
Criticidad: 5/5 (Sin cross-walks, procesamiento falla)
Impacto: 5/5 (Afecta todas las transacciones)
Frecuencia: 5/5 (Muy común olvidarlo)
Score = (5 × 5 × 5) / 15.6 = 8 puntos
```
| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| **Override de LoadSettings() presente** | 3 | **37.5% del subtotal**<br>• CRÍTICO: Llamado al inicio de cada ejecución por el framework<br>• Sin override, usa implementación base (vacía)<br>• Cross-walks no se cargan = todas las transacciones sin mapear<br>• Punto de entrada del lifecycle | Método override existe:<br>`protected override void LoadSettings(ConverterCmd cmd)` | **BLOQUEANTE**: Cross-walks no cargan,<br>procesamiento falla |
| **Inicializa TransCodeCrossWalkStore** | 2.5 | **31.25% del subtotal**<br>• CRÍTICO: Sin esto, TODOS los transaction codes quedan sin mapear<br>• Resulta en transacciones rechazadas en Cupload DB<br>• Afecta al 100% de las transacciones del cliente | Línea presente:<br>`Meditech.TransCodeCrossWalkStore =`<br>`new TransactionCodeCrossWalkBase(...)` | **BLOQUEANTE**: Todas las transacciones fallan validación en Cupload |
| **Inicializa FinancialClassCrossWalkStore** | 2.5 | **31.25% del subtotal**<br>• CRÍTICO: Sin esto, financial classes no se mapean a códigos MDS<br>• Cuentas se asignan incorrectamente a queues<br>• Impacto directo en estrategia de colección | Línea presente:<br>`Meditech.FinancialClassCrossWalkStore =`<br>`new MiscCodeCrossWalkBase(...)` | **BLOQUEANTE**: Asignación incorrecta de cuentas (impacto de negocio) |

**Código de Referencia con Anotaciones de Puntaje:**

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

**Validación Automatizada por CODE VALIDATOR:**
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
## 2.2 Configure Override (7 puntos)
### Justificación del Subtotal: 7 puntos (35% de Pipeline)
Por qué 7 puntos:

- Controla la UI de configuración que usan Mitch y Shawna
- Sin Configure, el formatter no es configurable por el usuario
- BaseConfigurationForm es el estándar MDS - custom forms causan bugs

**Cálculo:**

```
Criticidad: 4/5 (Formatter no configurable)
Impacto: 4/5 (Afecta experiencia de usuario)
Frecuencia: 4/5 (Común usar custom forms)
Score = (4 × 4 × 4) / 9.1 = 7 puntos
```

| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| Usa BaseConfigurationForm | 3 | 42.8% del subtotal<br>• IMPORTANTE: BaseConfigurationForm proporciona UI estándar<br>• Incluye logging, validación y persistencia automática<br>• Custom Windows Forms causan inconsistencias UX<br>• Feedback del MVP: "confusing variable naming" en UI custom | Instancia BaseConfigurationForm<br>en método Configure | UI inconsistente,<br>bugs de persistencia,<br>UX fragmentada |
| Agrega cross-walk de Transaction Codes | 2 | 28.6% del subtotal<br>• CRÍTICO: Sin este tab, usuarios no pueden configurar mapeos<br>• Mitch/Shawna necesitan esto para setup inicial del cliente<br>• Cada cliente tiene códigos únicos que deben mapearse | ConfigForm.AddCrossWalk(<br>TransCodes, "Transaction Codes") | BLOQUEANTE: Configuración imposible,<br>formatter inutilizable sin intervención del desarrollador |
| Agrega cross-walk de Financial Classes | 2 | 28.6% del subtotal<br>• CRÍTICO: Sin este tab, no se pueden mapear financial classes<br>• Requerido para clasificación correcta de cuentas<br>• Setup inicial del cliente | ConfigForm.AddCrossWalk(<br>MiscCodes, "Financial Classes") | BLOQUEANTE: Configuración imposible,<br>clasificación de cuentas falla |


**Código de Referencia con Anotaciones:**
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

**Validación Automatizada:**
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

## 2.3 SaveSettings Override (5 puntos)
### Justificación del Subtotal: 5 puntos (25% de Pipeline)
Por qué 5 puntos:

- Menor peso que LoadSettings (8) y Configure (7) porque no es bloqueante
- Sin SaveSettings, el formatter funciona pero pierde configuración
- Impacto en usabilidad, no en funcionalidad core

**Cálculo:**

```Criticidad: 3/5 (Pérdida de configuración, no bloqueante)
Impacto: 3/5 (Afecta solo persistencia)
Frecuencia: 4/5 (Frecuentemente olvidado)
Score = (3 × 3 × 4) / 7.2 = 5 puntos
```

| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| Override de SaveSettings() presente | 2 | 40% del subtotal<br>• IMPORTANTE: Sin override, configuración no se persiste<br>• Usuario debe reconfigurar en cada ejecución<br>• Afecta productividad de Mitch/Shawna | Método override existe:<br>protected override void SaveSettings() | Reconfiguración manual en cada run,<br>pérdida de tiempo |
| Guarda TransCodeCrossWalkStore | 1.5 | 30% del subtotal<br>• IMPORTANTE: Persistencia de mapeos de transaction codes<br>• Sin esto, mapeos configurados en UI se pierden | .Save() llamado en<br>TransCodeCrossWalkStore | Pérdida de configuración de transacciones entre ejecuciones |
| Guarda FinancialClassCrossWalkStore | 1.5 | 30% del subtotal<br>• IMPORTANTE: Persistencia de mapeos de financial classes<br>• Sin esto, mapeos de clases se pierden | .Save() llamado en<br>FinancialClassCrossWalkStore | Pérdida de configuración de clases entre ejecuciones |

**Código de Referencia:**
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

**Validación Automatizada:**

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

## 3. FILEHELPERS Y RECORD TYPES (18 puntos)
### 📌 Justificación del Peso Total: 18 puntos (18% del score)
Razón: FileHelpers es el motor de parsing que convierte archivos de texto del cliente en objetos C#. Errores aquí causan:

- Pérdida de datos: Campos no parseados
- Corrupción de datos: Parsing incorrecto de decimales/fechas
- Runtime exceptions: Tipos incompatibles

### Impacto de Fallo:
- Datos del cliente se pierden silenciosamente
- Datos incorrectos se insertan en Cupload (corrupción de DB)
- Exceptions durante procesamiento

### Evidencia de Criticidad del Contexto:

"Wrong FileHelpers package and decorators" (MDS Project Follow-UP) "Schema naming inconsistencies" (MDS Project Follow-UP)

**Cálculo:**
```
Criticidad: 5/5 (Pérdida/corrupción de datos)
Impacto: 4/5 (Afecta calidad de datos)
Frecuencia: 4/5 (Errores comunes en atributos)
Score = (5 × 4 × 4) / 4.4 = 18 puntos
```

## 3.1 Atributos de Clase Record (6 puntos)
## Justificación del Subtotal: 6 puntos (33.3% de FileHelpers)
Por qué 6 puntos:
- Atributos de clase definen el modo de parsing completo
- Sin [DelimitedRecord] o [FixedLengthRecord], FileHelpers no sabe cómo parsear
- [IgnoreEmptyLines] previene errores frecuentes en archivos médicos

**Cálculo:**
- Criticidad: 5/5 (BLOQUEANTE sin atributo principal)
- Impacto: 5/5 (Afecta parsing completo)
- Frecuencia: 3/5 (Moderado - generadores suelen incluirlo)
- Score = (5 × 5 × 3) / 12.5 = 6 puntos

| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| [DelimitedRecord] o [FixedLengthRecord] | 3 | 50% del subtotal<br>• CRÍTICO: Define el modo fundamental de parsing<br>• Sin esto, FileHelpers lanza exception al intentar parsear<br>• Debe coincidir con formato real del archivo del cliente | Atributo de clase presente:<br>[DelimitedRecord("\t")] o<br>[FixedLengthRecord] | BLOQUEANTE:<br>FileHelpers exception:<br>"Class must have DelimitedRecord or FixedLengthRecord attribute" |
| [IgnoreEmptyLines] | 2 | 33.3% del subtotal<br>• IMPORTANTE: Archivos médicos frecuentemente tienen líneas vacías<br>• Sin esto, FileHelpers intenta parsear líneas vacías y falla<br>• Previene errores de "line too short" | Atributo presente:<br>[IgnoreEmptyLines]<br>(para archivos delimitados) | Parsing errors en líneas vacías:<br>"Line X is too short",<br>datos perdidos |
| Clase sealed | 1 | 16.7% del subtotal<br>• BEST PRACTICE: Record types son DTOs, no deben heredarse<br>• sealed previene uso incorrecto en futuro<br>• Mejora rendimiento mínimo | public sealed class<br>en declaración | Ninguna (best practice),<br>posible uso incorrecto futuro |


**Código de Referencia con Anotaciones:**

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

**Validación Automatizada:**

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

## 3.2 Atributos de Campo (6 puntos)
### Justificación del Subtotal: 6 puntos (33.3% de FileHelpers)

Por qué 6 puntos:

- Los atributos de campo controlan el parsing de cada columna
- Configuración incorrecta = datos corruptos o exceptions
- Fechas y decimales en formatos médicos requieren conversión especial
- [FieldTrim] es crítico para matching de datos (lookups en DB)

**Cálculo:**
```
Criticidad: 5/5 (Datos corruptos/perdidos)
Impacto: 4/5 (Afecta cada campo)
Frecuencia: 5/5 (Muy común configurar mal)
Score = (5 × 4 × 5) / 16.7 = 6 puntos
```
| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| [FieldQuoted] o [FieldFixedLength] | 2 | 33.3% del subtotal<br>• CRÍTICO: Define cómo parsear cada campo individual<br>• [FieldQuoted] para CSV - maneja comillas y delimitadores dentro del valor<br>• [FieldFixedLength(X)] para fixed-width - define ancho de columna<br>• Configuración incorrecta = parsing incorrecto de datos | Atributo correcto por campo:<br>[FieldQuoted] para delimited<br>[FieldFixedLength(10)] para fixed | Datos parseados incorrectamente,<br>columnas desalineadas,<br>valores truncados |
| [FieldTrim(TrimMode.Both)] | 1.5 | 25% del subtotal<br>• IMPORTANTE: Archivos médicos tienen espacios extra en valores<br>• Sin trim, "SMITH " ≠ "SMITH" en lookups de DB<br>• Causa fallos en joins y búsquedas de cuentas<br>• Aplica a todos los campos string | Presente en campos string:<br>[FieldTrim(TrimMode.Both)] | Fallos en matching de datos,<br>lookups no encuentran registros,<br>duplicados por espacios |
| [FieldConverter] para tipos especiales | 1.5 | 25% del subtotal<br>• CRÍTICO: Fechas en formatos médicos son no-estándar<br>• Decimales pueden tener formato especial (sin punto decimal)<br>• Sin converter correcto = exception o valor incorrecto<br>• EJEMPLOS: "20231215" → DateTime, "12345" → 123.45m | [FieldConverter(typeof(MDSDecimalConverter))]<br>[FieldConverter(typeof(MDSDateConverter))]<br>[FieldConverter(typeof(ClarioneseDateConverter))] | BLOQUEANTE:<br>FormatException al parsear,<br>o datos incorrectos insertados silenciosamente |
| [FieldNullValue] para defaults | 1 | 16.7% del subtotal<br>• IMPORTANTE: Campos opcionales necesitan valores por defecto<br>• Sin esto, nulls causan exceptions en DB insert<br>• Previene NullReferenceException en handlers<br>• EJEMPLOS: decimal → 0, DateTime → DateTime.MinValue | [FieldNullValue(typeof(decimal), "0")]<br>[FieldNullValue(typeof(DateTime), "1900-01-01")] | NullReferenceException en handlers,<br>DB constraint violation,<br>datos inconsistentes |



**Código de Referencia Completo con Anotaciones:**

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
**Validación Automatizada Inteligente:**

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

## 3.3 Field Mapping a Base de Datos (6 puntos)
### Justificación del Subtotal: 6 puntos (33.3% de FileHelpers)
Por qué 6 puntos:

- [FieldMapping] es el puente crítico entre archivo parseado y DB
- Sin esto, datos parseados correctamente no se persisten (pérdida total)
- TableDestination incorrecto = corrupción silenciosa de datos
- Mapeo a columnas inexistentes = runtime exception

**Cálculo:**

```
Criticidad: 5/5 (Pérdida o corrupción de datos)
Impacto: 5/5 (Afecta persistencia completa)
Frecuencia: 4/5 (Común mapear incorrectamente)
Score = (5 × 5 × 4) / 16.7 = 6 puntos
```

| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| [FieldMapping] attributes presentes | 4 | 66.7% del subtotal<br>• CRÍTICO: Sin esto, datos parseados NO se escriben a DB<br>• Es el conector entre objeto C# y tabla Cupload<br>• Cobertura mínima: 80% de campos deben estar mapeados<br>• Campos no mapeados = pérdida silenciosa de datos del cliente | Atributo presente en >80% campos:<br>[FieldMapping("accountnumber",<br>TableDestination.Master)] | CRÍTICO:<br>Datos no se persisten,<br>pérdida total de información parseada |
| TableDestination correctos | 2 | 33.3% del subtotal<br>• CRÍTICO: Mapeo incorrecto = tabla equivocada<br>• EJEMPLO: Insurance data en Master table = corrupción<br>• Validar contra schema de Cupload (uplmaster, upltrans, uplinsurance)<br>• Error silencioso - no genera exception, corrompe datos | Valores correctos:<br>TableDestination.Master<br>TableDestination.Trans<br>TableDestination.Insurance | CRÍTICO:<br>Corrupción silenciosa de datos en Cupload,<br>impacto en negocio |


**Código de Referencia con Schema de Cupload:**

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

**Schema Cupload de Referencia (para validación):**

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

**Validación Automatizada con Schema Check:**

```csharp
public ValidationResult ValidateFieldMappings(Type recordType)
{
    var result = new ValidationResult 
    { 
        Category = "Field Mappings", 
        MaxScore = 6 
    };
    
    var fields = recordType.GetFields(BindingFlags.Public | BindingFlags.Instance);
    int totalFields = fields.Length;
    int mappedFields = 0;
    int correctDestination = 0;
    int totalMappings = 0;
    
    var schemaErrors = new List<string>();
    
    foreach (var field in fields)
    {
        var mappingAttr = field.GetCustomAttribute<FieldMappingAttribute>();
        
        if (mappingAttr != null)
        {
            mappedFields++;
            totalMappings++;
            
            // Validate column exists in target table
            TableSchema targetSchema = GetSchemaForDestination(mappingAttr.Destination);
            
            if (targetSchema == null)
            {
                schemaErrors.Add($"Field '{field.Name}': Invalid TableDestination '{mappingAttr.Destination}'");
                continue;
            }
            
            // Check if column exists in schema
            var column = targetSchema.FindColumn(mappingAttr.ColumnName);
            
            if (column == null)
            {
                result.AddCriticalError(
                    $"Field '{field.Name}' maps to non-existent column '{mappingAttr.ColumnName}'",
                    $"Table: {targetSchema.Name}",
                    "Impact: Runtime exception during DB insert",
                    $"Available columns: {string.Join(", ", targetSchema.GetColumnNames().Take(10))}..."
                );
                continue;
            }
            
            correctDestination++;
            
            // Validate type compatibility
            if (!IsTypeCompatible(field.FieldType, column.Type))
            {
                result.AddWarning(
                    $"Field '{field.Name}' type mismatch",
                    $"Field type: {field.FieldType.Name}, Column type: {column.Type.Name}",
                    "May cause data truncation or conversion errors"
                );
            }
            
            // Validate required fields
            if (column.Required && IsNullableOrOptional(field.FieldType))
            {
                result.AddWarning(
                    $"Field '{field.Name}' maps to required column but is nullable",
                    "Ensure FieldNullValue is set or handler provides default"
                );
            }
        }
    }
    
    // Check 1: FieldMapping coverage (4 pts)
    double mappingCoverage = (double)mappedFields / totalFields;
    
    if (mappingCoverage < 0.5)
    {
        result.AddCriticalError(
            $"Only {mappedFields}/{totalFields} fields ({mappingCoverage:P0}) have [FieldMapping]",
            "Impact: Majority of parsed data will NOT be persisted to database",
            "Minimum expected: 80% coverage",
            "Fix: Add [FieldMapping] attributes to fields that should be saved"
        );
        result.Score += mappingCoverage * 4.0; // Partial credit
    }
    else if (mappingCoverage < 0.8)
    {
        result.AddWarning(
            $"Only {mappedFields}/{totalFields} fields ({mappingCoverage:P0}) have [FieldMapping]",
            "Expected: At least 80% coverage",
            "Some parsed data may be lost"
        );
        result.Score += mappingCoverage * 4.0; // Partial credit
    }
    else
    {
        result.Score += 4.0; // Full credit
        result.AddEvidence($"✓ {mappedFields}/{totalFields} fields ({mappingCoverage:P0}) mapped to database");
    }
    
    // Check 2: TableDestination correctness (2 pts)
    if (totalMappings == 0)
    {
        // Already penalized in Check 1
    }
    else
    {
        double destinationAccuracy = (double)correctDestination / totalMappings;
        
        if (destinationAccuracy < 0.9)
        {
            result.AddCriticalError(
                $"Only {correctDestination}/{totalMappings} mappings ({destinationAccuracy:P0}) have correct TableDestination",
                "Impact: Data written to wrong tables = silent data corruption",
                "Example: Insurance data in Master table",
                schemaErrors.Count > 0 ? $"Errors: {string.Join("; ", schemaErrors)}" : ""
            );
            result.Score += destinationAccuracy * 2.0; // Partial credit
        }
        else
        {
            result.Score += 2.0; // Full credit
            result.AddEvidence($"✓ {correctDestination}/{totalMappings} mappings ({destinationAccuracy:P0}) have valid destinations");
        }
    }
    
    return result;
}

private TableSchema GetSchemaForDestination(TableDestination destination)
{
    switch (destination)
    {
        case TableDestination.Master:
            return CuploadSchema.MasterTable;
        case TableDestination.Trans:
            return CuploadSchema.TransTable;
        case TableDestination.Insurance:
            return CuploadSchema.InsuranceTable;
        default:
            return null;
    }
}

private bool IsTypeCompatible(Type fieldType, Type columnType)
{
    // Unwrap nullable types
    var actualFieldType = Nullable.GetUnderlyingType(fieldType) ?? fieldType;
    
    // Exact match
    if (actualFieldType == columnType)
        return true;
    
    // String is compatible with most types (will be converted)
    if (actualFieldType == typeof(string))
        return true;
    
    // Numeric compatibility
    if (IsNumericType(actualFieldType) && IsNumericType(columnType))
        return true;
    
    return false;
}

private bool IsNumericType(Type type)
{
    return type == typeof(int) || type == typeof(long) || 
           type == typeof(decimal) || type == typeof(double) || 
           type == typeof(float);
}
```

## 4. HANDLERS Y LÓGICA DE NEGOCIO (17 puntos)
### 📌 Justificación del Peso Total: 17 puntos (17% del score)
Razón: Los handlers implementan la lógica de negocio específica del formatter:

- DemographicsHandler: Procesa cuentas, actualiza balances, maneja recall protection
- TransactionHandler: Carga transacciones en batch
- InventoryHandler: Combina parsing FileHelpers + lógica de negocio

**Impacto de Fallo:**

- Herencia incorrecta = compilation errors
- Lógica faltante = datos incompletos o incorrectos
- Account cache no utilizado = violación de reglas de recalled accounts

**Evidencia de Criticidad del Contexto:**

"Handlers not inheriting from Meditech base classes" (MDS Project Follow-UP) "Incorrect transaction loading logic" (MDS Project Follow-UP)

**Cálculo:**

```
Criticidad: 4/5 (Impacta lógica de negocio)
Impacto: 4/5 (Afecta procesamiento de datos)
Frecuencia: 5/5 (Errores muy comunes en handlers)
Score = (4 × 4 × 5) / 4.7 = 17 puntos
```
### 4.1 DemographicsHandler (7 puntos)
Justificación del Subtotal: 7 puntos (41.2% de Handlers)
Por qué 7 puntos:

- Es el handler más crítico - procesa información de cuentas
- Recall protection es requisito legal/de negocio
- Errores aquí afectan a todas las cuentas del cliente

**Cálculo:**
```
Criticidad: 5/5 (Procesa datos críticos de cuentas)
Impacto: 5/5 (Afecta todas las cuentas)
Frecuencia: 4/5 (Común heredar de clase incorrecta)
Score = (5 × 5 × 4) / 14.3 = 7 puntos
```
| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| Hereda de MedBaseCollectionsHandler | 3 | 42.8% del subtotal<br>• CRÍTICO: BaseCollectionsHandler es abstracto - no usar clase base correcta<br>• MedBaseCollectionsHandler tiene lógica específica de Meditech<br>• Incluye: cross-walk application, field defaults, validations<br>• Error del MVP: heredar de clase genérica | `class DemographicsHandler : MedBaseCollectionsHandler` | BLOQUEANTE:<br>Compilation error o<br>lógica de negocio faltante |
| Constructor correcto | 1.5 | 21.4% del subtotal<br>• IMPORTANTE: Recibe converter reference + file context<br>• Constructor incorrecto = runtime exception al instanciar | `public DemographicsHandler(BaseConverter conv, ProcessFile file) : base(conv, file)` | Runtime exception:<br>"No suitable constructor found" |
| Usa AccountCache para recall check | 2.5 | 35.8% del subtotal<br>• CRÍTICO: Violación de reglas de negocio si no se verifica<br>• Recalled accounts no deben recibir nuevas transacciones<br>• Legal/compliance issue en industria médica<br>• EJEMPLO: Account deleted por disputa legal, no debe reactivarse | Código llama:<br>`if (converter.Accounts.IsRecalled(acctNum))`<br>`return; // Skip recalled account` | CRÍTICO:<br>Violación de reglas de negocio,<br>posibles problemas legales |

**Código de Referencia con Anotaciones:**

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

**Validación Automatizada:**

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

## 4.2 TransactionHandler (5 puntos)
Justificación del Subtotal: 5 puntos (29.4% de Handlers)
Por qué 5 puntos:

- Menor peso que Demographics (7) porque las transacciones son secundarias a las cuentas
- Sin embargo, errores aquí causan pérdida completa de historial de transacciones
- LoadTransactions() es el método crítico - debe estar implementado

**Cálculo:**

```
Criticidad: 4/5 (Pérdida de datos de transacciones)
Impacto: 4/5 (Afecta historial completo)
Frecuencia: 4/5 (Común no implementar correctamente)
Score = (4 × 4 × 4) / 12.8 = 5 puntos
```

| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| Hereda de MedBaseTransactionHandler | 2 | 40% del subtotal<br>• IMPORTANTE: Contiene lógica de batch loading de transacciones<br>• Incluye aplicación de cross-walks de trans codes<br>• Validaciones de tipos de transacción (P, A, I, C) | `class TransactionHandler : MedBaseTransactionHandler` | BLOQUEANTE:<br>Lógica de batch loading faltante |
| Constructor correcto | 1 | 20% del subtotal<br>• IMPORTANTE: Similar a Demographics handler | `public TransactionHandler(BaseConverter conv, ProcessFile file) : base(conv, file)` | Runtime exception al instanciar |
| Implementa LoadTransactions() | 2 | 40% del subtotal<br>• CRÍTICO: Es el método que carga transacciones en batch<br>• Sin esto, transacciones se pierden completamente<br>• Debe crear DataTable y llamar bulk insert | Override de LoadTransactions()<br>que retorna DataTable | CRÍTICO:<br>Pérdida total de transacciones,<br>solo demographics se procesan |

**Código de Referencia:**

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

**Validación Automatizada:**

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

## 4.3 InventoryHandler (si aplica) (5 puntos)
Justificación del Subtotal: 5 puntos (29.4% de Handlers)
Por qué 5 puntos:
- CONDICIONAL: Solo si el cliente tiene inventory files
- Combina FileHelpers parsing + handler logic
- Menor peso que otros handlers porque no todos los clientes lo requieren

**Cálculo:**
```
Criticidad: 3/5 (No todos los clientes lo necesitan)
Impacto: 4/5 (Si se necesita, es crítico)
Frecuencia: 3/5 (Moderado - ~50% clientes)
Score = (3 × 4 × 3) / 7.2 = 5 puntos
```

| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| Hereda de MedBaseGenericFileHandler | 2 | 40% del subtotal<br>• IMPORTANTE: Proporciona infraestructura de FileHelpers<br>• Maneja parsing automático de record types<br>• Incluye error handling y logging | `class InventoryHandler : MedBaseGenericFileHandler<RecordType>` | Lógica de parsing manual,<br>código duplicado |
| Constructor recibe typeof(RecordType) | 1.5 | 30% del subtotal<br>• IMPORTANTE: Le dice al handler qué record type usar para parsing<br>• Sin esto, FileHelpers no sabe qué estructura esperar | `public InventoryHandler(...) : base(conv, file, typeof(InventoryRecordType))` | Runtime exception:<br>"Record type not specified" |
| ProcessRecord implementado | 1.5 | 30% del subtotal<br>• IMPORTANTE: Lógica específica por registro parseado<br>• Aplicación de cross-walks, validaciones, inserts | Override de ProcessRecord<br>con lógica de negocio | Datos parseados pero no procesados,<br>no se persisten |

**Código de Referencia:**

- CONDICIONAL: Solo si el cliente tiene inventory files
- Combina FileHelpers parsing + handler logic
- Menor peso que otros handlers porque no todos los clientes lo requieren

**Cálculo:**
```
Criticidad: 3/5 (No todos los clientes lo necesitan)
Impacto: 4/5 (Si se necesita, es crítico)
Frecuencia: 3/5 (Moderado - ~50% clientes)
Score = (3 × 4 × 3) / 7.2 = 5 puntos
```

| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| Hereda de MedBaseGenericFileHandler | 2 | 40% del subtotal<br>• IMPORTANTE: Proporciona infraestructura de FileHelpers<br>• Maneja parsing automático de record types<br>• Incluye error handling y logging | `class InventoryHandler : MedBaseGenericFileHandler<RecordType>` | Lógica de parsing manual,<br>código duplicado |
| Constructor recibe typeof(RecordType) | 1.5 | 30% del subtotal<br>• IMPORTANTE: Le dice al handler qué record type usar para parsing<br>• Sin esto, FileHelpers no sabe qué estructura esperar | `public InventoryHandler(...) : base(conv, file, typeof(InventoryRecordType))` | Runtime exception:<br>"Record type not specified" |
| ProcessRecord implementado | 1.5 | 30% del subtotal<br>• IMPORTANTE: Lógica específica por registro parseado<br>• Aplicación de cross-walks, validaciones, inserts | Override de ProcessRecord<br>con lógica de negocio | Datos parseados pero no procesados,<br>no se persisten |


**Código de Referencia:**

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

**Validación Automatizada:**

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

## 5. CROSS-WALKS Y CONFIGURACIÓN (12 puntos)
📌 Justificación del Peso Total: 12 puntos (12% del score)
Razón: Cross-walks son mapeos configurables entre códigos del cliente y códigos estándar MDS:
- Transaction Codes: Mapean códigos de transacción del cliente a tipos MDS (P, A, I, C)
- Financial Classes: Mapean clases financieras del cliente a queues MDS (SP, COM, MCR, MCD)

**Impacto de Fallo:**

- Sin cross-walks: 100% de códigos sin mapear → datos rechazados
- Arrays vacíos: Configuración imposible → formatter inutilizable sin developer
- Estructura incorrecta: Runtime exceptions al cargar configuración

**Evidencia de Criticidad del Contexto:**

"Identifies required converters" (proc_mds2) 
"Verifies all sample codes are mapped" (mds_act2)

**Cálculo:**

```Criticidad: 4/5 (Datos sin mapear son rechazados)
Impacto: 4/5 (Afecta todas las transacciones/cuentas)
Frecuencia: 3/5 (Moderado - estructuras conocidas)
Score = (4 × 4 × 3) / 4 = 12 puntos
```

### 5.1 Transaction Code Items (6 puntos)
Justificación del Subtotal: 6 puntos (50% de Cross-Walks)
Por qué 6 puntos:
- Mayor peso que Financial Classes porque cada transacción requiere mapeo
- Sin estos mapeos, todas las transacciones fallan validación en Cupload
- Estructura incorrecta causa runtime exception al cargar cross-walks

**Cálculo:**
```
Criticidad: 5/5 (Todas las transacciones fallan sin esto)
Impacto: 5/5 (Afecta historial completo de transacciones)
Frecuencia: 3/5 (Estructura conocida pero errores comunes)
Score = (5 × 5 × 3) / 12.5 = 6 puntos
```

| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| Array TransactionCodeItems existe | 2 | 33.3% del subtotal<br>• CRÍTICO: Sin este array, cross-walk no se puede configurar<br>• LoadSettings lanza NullReferenceException<br>• Usado por TransCodeCrossWalkStore | Array estático presente:<br>`private static TransactionCodeItem[] TransactionCodeItems` | BLOQUEANTE:<br>NullReferenceException en LoadSettings,<br>formatter no arranca |
| Array tiene elementos iniciales | 2 | 33.3% del subtotal<br>• IMPORTANTE: Sin códigos sample, usuarios no saben qué mapear<br>• Best practice: incluir los 5-10 códigos más comunes del cliente<br>• Facilita configuración inicial | Array tiene al menos 3 elementos:<br>`new TransactionCodeItem[] {`<br>`  new(...), new(...), ...`<br>`}` | Configuración difícil para usuarios,<br>múltiples iteraciones de setup |
| Estructura correcta de items | 2 | 33.3% del subtotal<br>• IMPORTANTE: Cada item debe tener ClientCode y TransType<br>• TransType debe ser P, A, I, o C (Payment, Adjustment, Info, Charge)<br>• ClientCode debe ser string válido del sistema del cliente | Items con estructura:<br>`new TransactionCodeItem(`<br>`  "CHG", // ClientCode`<br>`  "C",   // TransType`<br>`  "Charge")` | Runtime exception al aplicar cross-walk,<br>validación falla |

**Código de Referencia con Anotaciones:**

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

**Validación Automatizada:**
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

## 5.2 Financial Class Items (6 puntos)
Justificación del Subtotal: 6 puntos (50% de Cross-Walks)
Por qué 6 puntos:
- Igual peso que Transaction Codes porque ambos son igualmente críticos
- Financial classes determinan queue assignment (impacto directo en negocio)
- Sin mapeos, cuentas no se clasifican correctamente → estrategia de colección incorrecta

**Cálculo:**
```
Criticidad: 5/5 (Clasificación incorrecta = estrategia incorrecta)
Impacto: 5/5 (Afecta todas las cuentas)
Frecuencia: 3/5 (Estructura conocida)
Score = (5 × 5 × 3) / 12.5 = 6 puntos
```
| Criterio | Puntos | Justificación del Puntaje | Forma de Validación | Penalización por Incumplimiento |
|----------|--------|---------------------------|---------------------|--------------------------------|
| Array FinancialClassItems existe | 2 | 33.3% del subtotal<br>• CRÍTICO: Sin este array, financial class cross-walk no funciona<br>• LoadSettings lanza NullReferenceException<br>• Usado por FinancialClassCrossWalkStore | Array estático presente:<br>`private static MiscCodeItem[] FinancialClassItems` | BLOQUEANTE:<br>NullReferenceException en LoadSettings |
| Array tiene elementos iniciales | 2 | 33.3% del subtotal<br>• IMPORTANTE: Debe incluir las clases más comunes<br>• EJEMPLOS: SP, COM, MCR, MCD<br>• Facilita setup inicial por parte de Mitch/Shawna | Array tiene al menos 4 elementos:<br>SP, COM, MCR, MCD | Configuración difícil,<br>múltiples iteraciones |
| Estructura correcta de items | 2 | 33.3% del subtotal<br>• IMPORTANTE: Cada item debe tener ClientCode y Description<br>• ClientCode se mapea a MDS standard codes | Items con estructura:<br>`new MiscCodeItem(`<br>`  "SP",        // ClientCode`<br>`  "Self Pay")  // Description` | Runtime exception al configurar,<br>mapeos no funcionan |

**Validación Automatizada:**

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

## 6. ROBUSTEZ Y EDGE CASES (8 puntos)
📌 Justificación del Peso Total: 8 puntos (8% del score)
**Razón:** Esta sección cubre **defensividad del código** y manejo de **casos extremos** que causan crashes en producción:
- **Null checks**: Previenen NullReferenceException
- **Empty file handling**: Archivos vacíos no deben crashear el sistema
- **Duplicate handling**: Misma cuenta procesada múltiples veces
- **Error logging**: Permite debugging en producción

**Impacto de Fallo:**
- Crashes en producción durante procesamiento nocturno
- Pérdida de datos cuando error no se captura
- Debugging imposible sin logs adecuados

**Evidencia de Criticidad del Contexto:**
> *"Use of non-existent logging"* (MDS Project Follow-UP)
> *"Some nonsensical or invalid code"* (MDS Project Follow-UP)

**Cálculo:**
```
Criticidad: 3/5 (No bloqueante pero causa crashes)
Impacto: 3/5 (Afecta estabilidad)
Frecuencia: 5/5 (Edge cases muy comunes en producción)
Score = (3 × 3 × 5) / 5.6 = 8 puntos
```

---

### 6.1 Null Safety y Defensive Coding (4 puntos)

| Criterio | Puntos | Justificación | Validación | Penalización |
|----------|--------|---------------|------------|--------------|
| **Null checks antes de acceder propiedades** | 2 | **50% del subtotal**<br>• IMPORTANTE: Previene NullReferenceException en producción<br>• EJEMPLOS: record null, converter null, Settings null | Uso de null-conditional operators:<br>`record?.Property`<br>`if (record == null) return;` | Runtime crashes en producción cuando datos inesperados |
| **Validación de campos requeridos** | 2 | **50% del subtotal**<br>• IMPORTANTE: AccountNumber, MemberCode son requeridos<br>• Sin validación, DB constraints fallan = crash | Checks explícitos:<br>`if (string.IsNullOrEmpty(record.AccountNum))`<br>`{ LogError(); return; }` | DB constraint violations,<br>procesamiento se detiene |

**Código de Referencia:**

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

**Validación Automatizada:**

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

### 6.2 Error Handling y Logging (4 puntos)

| Criterio | Puntos | Justificación | Validación | Penalización |
|----------|--------|---------------|------------|--------------|
| Try-catch en operaciones críticas | 2 | 50% del subtotal<br>• IMPORTANTE: DB operations pueden fallar (network, constraints)<br>• Sin try-catch, una falla detiene todo el batch | Try-catch alrededor de:<br>SaveToDatabase()<br>BulkInsert() | Batch completo falla por un registro malo |
| Logging con LogError/LogWarning | 2 | 50% del subtotal<br>• CRÍTICO: Sin logs, debugging en producción es imposible<br>• MVP error: usar Console.WriteLine en lugar de framework logging<br>• DEBE usar: LogError(), LogWarning(), LogInfo() | Llamadas a:<br>LogError(message)<br>LogWarning(message)<br>NO Console.WriteLine | Debugging imposible,<br>issues no detectables |

**Código de Referencia:**

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

**Validación Automatizada:**

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

## 7. DOCUMENTACIÓN Y LEGIBILIDAD (5 puntos)
📌 Justificación del Peso Total: 5 puntos (5% del score)
Razón: Código sin documentación es no mantenible por equipo MDS:

- README.md: Instrucciones de configuración para Mitch/Shawna
- XML comments: Documentación inline para developers
- Naming conventions: Código auto-documentado

**Impacto de Fallo:**

- Future developers no entienden la lógica
- Multiple support tickets por falta de documentación

**Evidencia de Criticidad del Contexto:**

"Overly detailed README; missing standard .csproj structure" (MDS Project Follow-UP) "Confusing variable naming" (MDS Project Follow-UP)

**Cálculo:**
```
Criticidad: 2/5 (No impacta funcionalidad)
Impacto: 3/5 (Impacta mantenibilidad)
Frecuencia: 5/5 (Muy común olvidar documentar)
Score = (2 × 3 × 5) / 6 = 5 puntos
```

### 7.1 README.md (3 puntos)

| Criterio | Puntos | Justificación | Validación | Penalización |
|----------|--------|---------------|------------|--------------|
| README exists con estructura estándar | 1.5 | 50% del subtotal<br>• IMPORTANTE: Mitch/Shawna necesitan instrucciones de configuración<br>• Debe incluir: Client info, File specs, Cross-walk setup | Archivo README.md existe<br>Secciones: Client Info, Files, Cross-Walks | Support tickets por falta de instrucciones |
| Cross-walk configuration documented | 1.5 | 50% del subtotal<br>• CRÍTICO: Sin esto, usuarios no saben cómo configurar códigos<br>• Debe explicar cómo usar ConverterManager UI | Sección "Cross-Walk Configuration"<br>con ejemplos de UI | Múltiples iteraciones de configuración |

**Template de README.md:**

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

**Validación Automatizada:**

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

### 7.2 Code Comments y Naming (2 puntos)

| Criterio | Puntos | Justificación | Validación | Penalización |
|----------|--------|---------------|------------|--------------|
| XML comments en métodos públicos | 1 | 50% del subtotal<br>• IMPORTANTE: Permite IntelliSense en Visual Studio<br>• Documenta propósito de handlers | XML comments /// <summary><br>en handlers públicos | Developers no entienden propósito |
| Variable naming descriptivo | 1 | 50% del subtotal<br>• IMPORTANTE: MVP error: "Confusing variable naming"<br>• CORRECTO: accountNumber, transactionCode<br>• INCORRECTO: x, tmp, var1 | Variables con nombres descriptivos<br>NO: single letters (excepto loops) | Código difícil de mantener |

**Código de Referencia:**

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

**Validación Automatizada:**

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

## 8. RESUMEN Y SCORING FINAL
📊 Distribución de Puntos (Total: 100)

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

## 📋 Validation Report Template
```csharp
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





