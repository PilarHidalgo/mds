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
Criticidad: 5/5 (BLOQUEANTE) Impacto: 5/5 (Afecta sistema completo) Frecuencia: 5/5 (Error común en MVP) Score = (5 × 5 × 5) / 12.5 = 10 puntos


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

Cálculo:

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

Cálculo:

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
