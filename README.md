# Diagrama: Sistema de Verificação de Duplicidade

## 🏛️ Visão Geral da Arquitetura

```mermaid
graph TB
    subgraph Request["🌐 REQUEST Entry Point"]
        ProcessRequest["Process Manager Request<br/>Equipment Type + Product + Technology"]
    end

    ProcessRequest --> PipelineSelector["Pipeline Selector<br/>DuplicityCheckPipelines"]

    PipelineSelector --> Determination["Pipeline Determination<br/>Based on:<br/>- EquipmentType<br/>- ProductType<br/>- TechnologyEnum"]

    Determination --> SwtPipeline["SWT Pipeline"]
    Determination --> IPv6Pipeline["IPv6 Pipeline"]
    Determination --> PePipeline["PE Pipeline"]
    Determination --> VpnPipeline["VPN Pipeline"]
    Determination --> SipPipeline["SIP Pipeline"]

    subgraph SwtFlow["🔧 SWT Flow (GPON + Dedicated IP)"]
        SwtPipeline --> SWT1["1. DuplicityCheckSwtTask"]
        SWT1 --> SWT2["2. GetEquipmentDetailsTask"]
        SWT2 --> SWT3["3. VrfDiscoveryTask"]
        SWT3 --> SWT4["4. CollectHostnameTask"]
        SWT4 --> SWT5["5. DuplicityCheckResultTask"]
    end

    subgraph IPv6Flow["🌐 IPv6 Flow (Default Equipment)"]
        IPv6Pipeline --> IPV61["1. DuplicityCheckIPV6Task"]
    end

    subgraph PeFlow["⚙️ PE Flow (GPON + Default Product)"]
        PePipeline --> PE1["1. DuplicityCheckTask"]
        PE1 --> PE2["2. GetEquipmentDetailsDuplicityCheckTask"]
        PE2 --> PE3["3. GetCompletePeInterfaceDuplicityCheckTask"]
        PE3 --> PE4["4. GetCustomerNameDuplicityCheckTask"]
        PE4 --> PE5["5. DuplicityCheckResultTask"]
    end

    subgraph VpnFlow["🔐 VPN Flow (GPON/SWT/FSP)"]
        VpnPipeline --> VPN1["1. DuplicityCheckTask"]
        VPN1 --> VPN2["2. DuplicityCheckSipVrfTask"]
        VPN2 --> VPN3["3. DuplicityCheckSipVpnTask"]
        VPN3 --> VPN4["4. GetEquipmentDetailsTask"]
        VPN4 --> VPN5["5. GetCompletePeInterfaceDuplicityCheckTask"]
        VPN5 --> VPN6["6. GetCustomerNameDuplicityCheckTask"]
        VPN6 --> VPN7["7. DuplicityCheckResultTask"]
    end

    subgraph SipFlow["📞 SIP Flow (GPON/SWT/FSP)"]
        SipPipeline --> SIP1["1. DuplicityCheckSipTask"]
        SIP1 --> SIP2["2. DuplicityCheckSipVrfTask"]
        SIP2 --> SIP3["3. DuplicityCheckSipBgpTask"]
        SIP3 --> SIP4["4. DuplicityCheckSipVpnTask"]
        SIP4 --> SIP5["5. GetEquipmentDetailsTask"]
        SIP5 --> SIP6["6. GetCompletePeInterfaceDuplicityCheckTask"]
        SIP6 --> SIP7["7. GetCustomerNameDuplicityCheckTask"]
        SIP7 --> SIP8["8. DuplicityCheckResultTask"]
    end

    SWT5 --> Result["📊 Result Output"]
    IPV61 --> Result
    PE5 --> Result
    VPN7 --> Result
    SIP8 --> Result

    style Request fill:#2d5a7b,stroke:#4a90c4,color:#fff
    style PipelineSelector fill:#7b5d2d,stroke:#c49a4a,color:#fff
    style Determination fill:#5d2d7b,stroke:#8a4ac4,color:#fff
    style Result fill:#2d7b3d,stroke:#4ac45a,color:#fff
    style SwtFlow fill:#7b6a2d,stroke:#c4a84a,color:#fff
    style IPv6Flow fill:#2d7b6a,stroke:#4ac4a8,color:#fff
    style PeFlow fill:#6a2d7b,stroke:#a84ac4,color:#fff
    style VpnFlow fill:#7b2d3d,stroke:#c44a5a,color:#fff
    style SipFlow fill:#3d7b2d,stroke:#5ac44a,color:#fff
```

---

## 🔄 Fluxo Detalhado: Seleção de Pipeline

```mermaid
flowchart TD
    Start([🎯 START Request Received]) --> Parse["Parse Request Parameters"]

    Parse --> Extract["Extract:<br/>- Equipment Type<br/>- Product Type<br/>- Technology"]

    Extract --> BuildKey["Build Pipeline Key:<br/>DUPLICITY_CHECK_<br/>{equipmentId}_{productId}_{techCode}"]

    BuildKey --> CheckSwt{Equipment: SWT<br/>Product: DEDICATED_IP<br/>Tech: GPON?}

    CheckSwt -->|Yes| SwtPipeline["SWT Pipeline<br/>5 tasks"]
    CheckSwt -->|No| CheckIPv6{Equipment: DEFAULT<br/>Product: DEFAULT<br/>Tech: IPV6?}

    CheckIPv6 -->|Yes| IPv6Pipeline["IPv6 Pipeline<br/>1 task"]
    CheckIPv6 -->|No| CheckPe{Equipment: PE<br/>Product: DEFAULT<br/>Tech: GPON?}

    CheckPe -->|Yes| PePipeline["PE Pipeline<br/>5 tasks"]
    CheckPe -->|No| CheckVpn{Equipment: PE<br/>Product: VPN<br/>Tech: GPON/SWT/FSP?}

    CheckVpn -->|Yes| VpnPipeline["VPN Pipeline<br/>7 tasks"]
    CheckVpn -->|No| CheckSip{Equipment: PE<br/>Product: SIP<br/>Tech: GPON/SWT/FSP?}

    CheckSip -->|Yes| SipPipeline["SIP Pipeline<br/>8 tasks"]
    CheckSip -->|No| ErrorPipeline["❌ Pipeline Not Found<br/>Error Handler"]

    SwtPipeline --> Execute["Execute Pipeline Tasks<br/>Sequential Execution"]
    IPv6Pipeline --> Execute
    PePipeline --> Execute
    VpnPipeline --> Execute
    SipPipeline --> Execute

    Execute --> End([✅ END<br/>Duplicity Check Complete])
    ErrorPipeline --> ErrorEnd([❌ END<br/>Process Failed])

    style Start fill:#2d7b3d,stroke:#4ac45a,color:#fff
    style End fill:#2d7b3d,stroke:#4ac45a,color:#fff
    style ErrorEnd fill:#7b2d3d,stroke:#c44a5a,color:#fff
    style BuildKey fill:#7b6a2d,stroke:#c4a84a,color:#fff
    style Execute fill:#5d2d7b,stroke:#8a4ac4,color:#fff
```

---

## 🔄 Fluxo Detalhado: SWT Pipeline (GPON + Dedicated IP)

```mermaid
sequenceDiagram
    participant PM as 📋 Process Manager
    participant PL as DuplicityCheckPipelines
    participant T1 as DuplicityCheckSwtTask
    participant T2 as GetEquipmentDetailsTask
    participant T3 as VrfDiscoveryTask
    participant T4 as CollectHostnameTask
    participant T5 as DuplicityCheckResultTask
    participant NW as 🌐 Network Device

    Note over PM: User requests duplicity check<br/>for SWT equipment

    PM->>PL: Request DUPLICITY_CHECK<br/>SWT + DEDICATED_IP + GPON

    activate PL
    PL->>PL: Select SWT Pipeline<br/>5 tasks to execute

    PL->>T1: Execute Task 1
    activate T1
    T1->>NW: Check SWT configuration
    NW-->>T1: Configuration data
    T1-->>PL: Check complete
    deactivate T1

    PL->>T2: Execute Task 2
    activate T2
    T2->>NW: Get equipment details
    NW-->>T2: Equipment information
    T2-->>PL: Details retrieved
    deactivate T2

    PL->>T3: Execute Task 3
    activate T3
    T3->>NW: Discover VRF configuration
    NW-->>T3: VRF data
    T3-->>PL: Discovery complete
    deactivate T3

    PL->>T4: Execute Task 4
    activate T4
    T4->>NW: Collect hostname
    NW-->>T4: Hostname data
    T4-->>PL: Hostname collected
    deactivate T4

    PL->>T5: Execute Task 5
    activate T5
    T5->>T5: Consolidate results<br/>Generate report
    T5-->>PL: Final result
    deactivate T5

    PL-->>PM: ✅ Duplicity check result
    deactivate PL

    Note over PM: Process complete<br/>Result available
```

---

## 🔄 Fluxo Detalhado: SIP Pipeline (GPON/SWT/FSP)

```mermaid
sequenceDiagram
    participant PM as 📋 Process Manager
    participant PL as DuplicityCheckPipelines
    participant T1 as DuplicityCheckSipTask
    participant T2 as DuplicityCheckSipVrfTask
    participant T3 as DuplicityCheckSipBgpTask
    participant T4 as DuplicityCheckSipVpnTask
    participant T5 as GetEquipmentDetailsTask
    participant T6 as GetCompletePeInterfaceTask
    participant T7 as GetCustomerNameTask
    participant T8 as DuplicityCheckResultTask

    Note over PM: SIP Product Request<br/>Most Complex Pipeline (8 tasks)

    PM->>PL: DUPLICITY_CHECK<br/>PE + SIP + GPON/SWT/FSP

    activate PL

    PL->>T1: 1. Check SIP Configuration
    activate T1
    T1-->>PL: SIP validation complete
    deactivate T1

    PL->>T2: 2. Check VRF for SIP
    activate T2
    T2-->>PL: VRF check complete
    deactivate T2

    PL->>T3: 3. Check BGP for SIP
    activate T3
    T3-->>PL: BGP check complete
    deactivate T3

    PL->>T4: 4. Check VPN for SIP
    activate T4
    T4-->>PL: VPN check complete
    deactivate T4

    PL->>T5: 5. Get Equipment Details
    activate T5
    T5-->>PL: Equipment data retrieved
    deactivate T5

    PL->>T6: 6. Get Complete PE Interface
    activate T6
    T6-->>PL: Interface data retrieved
    deactivate T6

    PL->>T7: 7. Get Customer Name
    activate T7
    T7-->>PL: Customer identified
    deactivate T7

    PL->>T8: 8. Generate Final Result
    activate T8
    T8->>T8: Consolidate all checks<br/>Generate comprehensive report
    T8-->>PL: Final result ready
    deactivate T8

    PL-->>PM: ✅ Complete SIP duplicity check
    deactivate PL

    Note over PM: Comprehensive validation complete
```

---

## 🔄 Fluxo Detalhado: VPN Pipeline (GPON/SWT/FSP)

```mermaid
flowchart TD
    Start([🔐 VPN Pipeline START]) --> Task1["1. DuplicityCheckTask<br/>Primary duplicity validation"]

    Task1 --> Check1{Duplicity<br/>found?}

    Check1 -->|No duplicity| Task2["2. DuplicityCheckSipVrfTask<br/>VRF validation for VPN"]
    Check1 -->|Duplicity exists| MarkDup["Mark as duplicate"]

    Task2 --> Task3["3. DuplicityCheckSipVpnTask<br/>VPN specific validation"]

    Task3 --> Task4["4. GetEquipmentDetailsTask<br/>Retrieve equipment information"]

    Task4 --> Task5["5. GetCompletePeInterfaceDuplicityCheckTask<br/>Get complete PE interface details"]

    Task5 --> Task6["6. GetCustomerNameDuplicityCheckTask<br/>Identify customer"]

    Task6 --> Task7["7. DuplicityCheckResultTask<br/>Consolidate and generate result"]

    MarkDup --> Task7

    Task7 --> Result["📊 VPN Duplicity Result<br/>- No duplicity found<br/>- Duplicity detected with details<br/>- Customer information<br/>- Equipment details"]

    Result --> End([✅ END VPN Check])

    style Start fill:#2d7b3d,stroke:#4ac45a,color:#fff
    style End fill:#2d7b3d,stroke:#4ac45a,color:#fff
    style MarkDup fill:#7b2d3d,stroke:#c44a5a,color:#fff
    style Result fill:#5d2d7b,stroke:#8a4ac4,color:#fff
```

---

## 📦 Estrutura de Pipelines

### Pipeline Configuration

```mermaid
graph TB
    subgraph PipelineStructure["Pipeline Structure"]
        Name["Pipeline Name<br/>Format: ACTION_EQUIPMENT_PRODUCT_TECH"]
        Tasks["Task List<br/>Ordered sequential execution"]
        InputClass["Input Model Class<br/>Type-safe inputs"]
    end

    Name --> Example1["DUPLICITY_CHECK_SWT_DEDICATED_IP_GPON"]
    Name --> Example2["DUPLICITY_CHECK_PE_VPN_FSP"]
    Name --> Example3["DUPLICITY_CHECK_PE_SIP_GPON"]

    Tasks --> Sequential["Sequential Execution<br/>Each task processes previous output"]
    
    InputClass --> SwtInputs["DuplicityCheckSwtInputs"]
    InputClass --> PeInputs["DuplicityCheckInputs"]
    InputClass --> IPv6Inputs["DuplicityCheckIPV6Inputs"]

    style PipelineStructure fill:#7b6a2d,stroke:#c4a84a,color:#fff
    style Name fill:#2d7b3d,stroke:#4ac45a,color:#fff
    style Tasks fill:#5d2d7b,stroke:#8a4ac4,color:#fff
    style InputClass fill:#7b2d3d,stroke:#c44a5a,color:#fff
```

### Task Dependencies

```java
@Component
public class DuplicityCheckPipelines implements ModulePipelines {
    
    // Core Tasks (All Pipelines)
    private final DuplicityCheckTask duplicityCheckTask;
    private final DuplicityCheckResultTask duplicityCheckResultTask;
    
    // Equipment Tasks
    private final GetEquipmentDetailsTask getEquipmentDetailsTask;
    private final GetEquipmentDetailsDuplicityCheckTask getEquipmentDetailsDuplicityCheckTask;
    
    // Interface Tasks
    private final GetCompleteInterfaceTask getCompleteInterfaceTask;
    private final GetCompletePeInterfaceDuplicityCheckTask getCompletePeInterfaceDuplicityCheckTask;
    
    // SIP-specific Tasks
    private final DuplicityCheckSipTask duplicityCheckSipTask;
    private final DuplicityCheckSipBgpTask duplicityCheckSipBgpTask;
    private final DuplicityCheckSipVrfTask duplicityCheckSipVrfTask;
    private final DuplicityCheckSipVpnTask duplicityCheckSipVpnTask;
    
    // SWT-specific Tasks
    private final DuplicityCheckSwtTask duplicityCheckSwtTask;
    private final VrfDiscoveryTask vrfDiscoveryTask;
    private final CollectHostnameTask collectHostnameTask;
    
    // IPv6-specific Tasks
    private final DuplicityCheckIPV6Task duplicityCheckIPV6Task;
    
    // Customer Tasks
    private final GetCustomerNameDuplicityCheckTask getCustomerNameDuplicityCheckTask;
}
```

---

## 📊 Pipeline Matrix

```mermaid
graph TB
    subgraph Matrix["Pipeline Configuration Matrix"]
        subgraph Row1["Equipment: SWT"]
            SWT_DED["Product: DEDICATED_IP<br/>Tech: GPON<br/>Tasks: 5"]
        end
        
        subgraph Row2["Equipment: DEFAULT"]
            DEF_IPV6["Product: DEFAULT<br/>Tech: IPV6<br/>Tasks: 1"]
        end
        
        subgraph Row3["Equipment: PE - Default"]
            PE_DEF["Product: DEFAULT<br/>Tech: GPON<br/>Tasks: 5"]
        end
        
        subgraph Row4["Equipment: PE - VPN"]
            PE_VPN_GPON["Product: VPN<br/>Tech: GPON<br/>Tasks: 7"]
            PE_VPN_SWT["Product: VPN<br/>Tech: SWT<br/>Tasks: 7"]
            PE_VPN_FSP["Product: VPN<br/>Tech: FSP<br/>Tasks: 7"]
        end
        
        subgraph Row5["Equipment: PE - SIP"]
            PE_SIP_GPON["Product: SIP<br/>Tech: GPON<br/>Tasks: 8"]
            PE_SIP_SWT["Product: SIP<br/>Tech: SWT<br/>Tasks: 8"]
            PE_SIP_FSP["Product: SIP<br/>Tech: FSP<br/>Tasks: 8"]
        end
    end

    SWT_DED -.->|Simplest SWT flow| Label1["Basic duplicity check<br/>+ Equipment details<br/>+ VRF discovery"]
    
    DEF_IPV6 -.->|Simplest overall| Label2["Single task<br/>IPv6 validation only"]
    
    PE_DEF -.->|Standard PE flow| Label3["Standard duplicity check<br/>+ PE interface details"]
    
    PE_VPN_GPON -.->|Multi-tech VPN| Label4["VPN validation<br/>+ VRF + VPN checks"]
    PE_VPN_SWT -.->|Multi-tech VPN| Label4
    PE_VPN_FSP -.->|Multi-tech VPN| Label4
    
    PE_SIP_GPON -.->|Most complex| Label5["SIP validation<br/>+ VRF + BGP + VPN checks"]
    PE_SIP_SWT -.->|Most complex| Label5
    PE_SIP_FSP -.->|Most complex| Label5

    style Row1 fill:#7b6a2d,stroke:#c4a84a,color:#fff
    style Row2 fill:#2d7b6a,stroke:#4ac4a8,color:#fff
    style Row3 fill:#6a2d7b,stroke:#a84ac4,color:#fff
    style Row4 fill:#7b2d3d,stroke:#c44a5a,color:#fff
    style Row5 fill:#3d7b2d,stroke:#5ac44a,color:#fff
```

---

## 📊 Task Execution Flow

```mermaid
graph LR
    subgraph Input["📥 Input Phase"]
        RequestIn["Process Request<br/>with parameters"]
        ValidateIn["Validate inputs<br/>against model"]
    end

    subgraph Selection["🎯 Selection Phase"]
        BuildKey["Build pipeline key"]
        FindPipe["Find matching pipeline"]
        LoadTasks["Load task list"]
    end

    subgraph Execution["⚙️ Execution Phase"]
        Task1["Execute Task 1"]
        Task2["Execute Task 2"]
        TaskN["Execute Task N"]
        
        Task1 -->|Output becomes input| Task2
        Task2 -->|Chained execution| TaskN
    end

    subgraph Output["📤 Output Phase"]
        Consolidate["Consolidate results"]
        Format["Format response"]
        Return["Return to Process Manager"]
    end

    RequestIn --> ValidateIn
    ValidateIn --> BuildKey
    BuildKey --> FindPipe
    FindPipe --> LoadTasks
    LoadTasks --> Task1
    TaskN --> Consolidate
    Consolidate --> Format
    Format --> Return

    style Input fill:#2d5a7b,stroke:#4a90c4,color:#fff
    style Selection fill:#7b5d2d,stroke:#c49a4a,color:#fff
    style Execution fill:#5d2d7b,stroke:#8a4ac4,color:#fff
    style Output fill:#2d7b3d,stroke:#4ac45a,color:#fff
```

---

## 🎯 Pontos Chave da Arquitetura

### ✅ Características Principais

- **Modularidade**: Cada pipeline é independente e focado em um tipo específico de equipamento/produto
- **Reutilização**: Tasks são compartilhadas entre pipelines quando aplicável
- **Flexibilidade**: Fácil adicionar novos pipelines sem afetar existentes
- **Type Safety**: Cada pipeline tem seu próprio modelo de input tipado
- **Manutenibilidade**: Separação clara de responsabilidades entre tasks
- **Escalabilidade**: Novos equipamentos/produtos podem ser adicionados facilmente

### 📋 Tipos de Equipamento

| Equipment Type | ID | Descrição |
|----------------|-------|-----------|
| **SWT** | 2 | Switch - Equipamento de comutação |
| **PE** | 3 | Provider Edge - Equipamento de borda |
| **DEFAULT** | 0 | Equipamento genérico |

### 📦 Tipos de Produto

| Product Type | ID | Descrição |
|--------------|-------|-----------|
| **DEFAULT** | 0 | Produto padrão |
| **DEDICATED_IP** | 1 | IP Dedicado |
| **VPN** | 2 | Virtual Private Network |
| **SIP** | 3 | Session Initiation Protocol |

### 🔧 Tipos de Tecnologia

| Technology | Code | Descrição |
|------------|------|-----------|
| **GPON** | GPON | Gigabit Passive Optical Network |
| **SWT** | SWT | Switch Technology |
| **FSP** | FSP | Fiber Service Platform |
| **IPV6** | IPV6 | Internet Protocol version 6 |

### 🔄 Fluxo de Tarefas Comum

```mermaid
graph LR
    A[Duplicity Check] --> B[Get Equipment Details]
    B --> C[Get Interface Details]
    C --> D[Get Customer Info]
    D --> E[Generate Result]
    
    style A fill:#7b2d3d,stroke:#c44a5a,color:#fff
    style B fill:#7b6a2d,stroke:#c4a84a,color:#fff
    style C fill:#5d2d7b,stroke:#8a4ac4,color:#fff
    style D fill:#2d7b6a,stroke:#4ac4a8,color:#fff
    style E fill:#2d7b3d,stroke:#4ac45a,color:#fff
```

### 🎨 Padrões de Task

1. **Check Tasks**: Verificam duplicidade ou configuração
   - `DuplicityCheckTask`
   - `DuplicityCheckSipTask`
   - `DuplicityCheckSwtTask`
   - `DuplicityCheckIPV6Task`

2. **Specialized Check Tasks**: Verificações específicas de protocolo
   - `DuplicityCheckSipBgpTask`
   - `DuplicityCheckSipVrfTask`
   - `DuplicityCheckSipVpnTask`

3. **Get Tasks**: Obtêm informações de recursos
   - `GetEquipmentDetailsTask`
   - `GetEquipmentDetailsDuplicityCheckTask`
   - `GetCompleteInterfaceTask`
   - `GetCompletePeInterfaceDuplicityCheckTask`
   - `GetCustomerNameDuplicityCheckTask`

4. **Discovery Tasks**: Descobrem configurações
   - `VrfDiscoveryTask`
   - `CollectHostnameTask`

5. **Result Tasks**: Consolidam e geram resultado final
   - `DuplicityCheckResultTask`

---

## 🔍 Casos de Uso Detalhados

### Caso 1: SWT + DEDICATED_IP + GPON

**Cenário**: Cliente solicita IP dedicado em equipamento Switch GPON

**Pipeline**: `DUPLICITY_CHECK_2_1_GPON`

**Tasks Executadas**:
1. `DuplicityCheckSwtTask` - Verifica duplicidade na configuração do switch
2. `GetEquipmentDetailsTask` - Obtém detalhes do equipamento
3. `VrfDiscoveryTask` - Descobre VRF configurada
4. `CollectHostnameTask` - Coleta hostname do equipamento
5. `DuplicityCheckResultTask` - Gera resultado consolidado

**Resultado**: Validação se o IP pode ser provisionado sem conflitos

---

### Caso 2: PE + SIP + GPON

**Cenário**: Cliente solicita serviço SIP em equipamento PE GPON

**Pipeline**: `DUPLICITY_CHECK_3_3_GPON`

**Tasks Executadas**:
1. `DuplicityCheckSipTask` - Verifica duplicidade específica de SIP
2. `DuplicityCheckSipVrfTask` - Valida VRF para SIP
3. `DuplicityCheckSipBgpTask` - Valida configuração BGP para SIP
4. `DuplicityCheckSipVpnTask` - Valida VPN para SIP
5. `GetEquipmentDetailsTask` - Obtém detalhes do equipamento PE
6. `GetCompletePeInterfaceDuplicityCheckTask` - Obtém interface completa do PE
7. `GetCustomerNameDuplicityCheckTask` - Identifica cliente
8. `DuplicityCheckResultTask` - Gera resultado final completo

**Resultado**: Validação abrangente incluindo SIP, VRF, BGP, VPN e dados do cliente

---

### Caso 3: DEFAULT + DEFAULT + IPV6

**Cenário**: Cliente solicita validação de endereço IPv6

**Pipeline**: `DUPLICITY_CHECK_0_0_IPV6`

**Tasks Executadas**:
1. `DuplicityCheckIPV6Task` - Valida unicidade do endereço IPv6

**Resultado**: Validação rápida e direta de IPv6 (pipeline mais simples)

---

### Caso 4: PE + VPN + FSP

**Cenário**: Cliente solicita VPN em equipamento PE FSP

**Pipeline**: `DUPLICITY_CHECK_3_2_FSP`

**Tasks Executadas**:
1. `DuplicityCheckTask` - Verificação primária de duplicidade
2. `DuplicityCheckSipVrfTask` - Valida VRF
3. `DuplicityCheckSipVpnTask` - Valida configuração VPN
4. `GetEquipmentDetailsTask` - Obtém detalhes do equipamento
5. `GetCompletePeInterfaceDuplicityCheckTask` - Obtém interface PE completa
6. `GetCustomerNameDuplicityCheckTask` - Identifica cliente
7. `DuplicityCheckResultTask` - Consolida resultado

**Resultado**: Validação completa de VPN com informações de cliente e equipamento

---

## 📈 Comparativo de Complexidade

```mermaid
graph TB
    subgraph Complexity["Complexidade por Pipeline"]
        Simple["🟢 Simples (1 task)<br/>IPv6"]
        Medium["🟡 Média (5 tasks)<br/>SWT, PE Default"]
        Complex["🟠 Complexa (7 tasks)<br/>VPN (todas tech)"]
        VeryComplex["🔴 Muito Complexa (8 tasks)<br/>SIP (todas tech)"]
    end

    Simple -.->|Uso| UseSimple["Validações rápidas<br/>Endereços IPv6"]
    Medium -.->|Uso| UseMedium["Validações padrão<br/>Equipamentos básicos"]
    Complex -.->|Uso| UseComplex["Validações avançadas<br/>VPN com múltiplos protocolos"]
    VeryComplex -.->|Uso| UseVeryComplex["Validações críticas<br/>SIP com BGP/VRF/VPN"]

    style Simple fill:#2d7b3d,stroke:#4ac45a,color:#fff
    style Medium fill:#7b6a2d,stroke:#c4a84a,color:#fff
    style Complex fill:#7b5d2d,stroke:#c49a4a,color:#fff
    style VeryComplex fill:#7b2d3d,stroke:#c44a5a,color:#fff
```

---

## 🔐 Validações de Segurança

```mermaid
flowchart TD
    Start([🔒 Security Validation]) --> InputValidation["Input Validation<br/>Verify model integrity"]

    InputValidation --> PipelineAuth["Pipeline Authorization<br/>Check user permissions"]

    PipelineAuth --> ResourceCheck["Resource Availability<br/>Verify equipment access"]

    ResourceCheck --> DuplicityCheck["Duplicity Check<br/>Prevent conflicts"]

    DuplicityCheck --> ProtocolValidation["Protocol Validation<br/>VRF, BGP, VPN checks"]

    ProtocolValidation --> CustomerValidation["Customer Validation<br/>Verify ownership"]

    CustomerValidation --> FinalValidation["Final Validation<br/>All checks passed"]

    FinalValidation --> Approved{All validations<br/>passed?}

    Approved -->|Yes| Success([✅ Approved])
    Approved -->|No| Reject([❌ Rejected])

    style Start fill:#5d2d7b,stroke:#8a4ac4,color:#fff
    style Success fill:#2d7b3d,stroke:#4ac45a,color:#fff
    style Reject fill:#7b2d3d,stroke:#c44a5a,color:#fff
    style DuplicityCheck fill:#7b6a2d,stroke:#c4a84a,color:#fff
```

---

## 📊 Estatísticas de Pipelines

### Total de Pipelines Configurados

- **Total**: 9 pipelines
- **Equipamentos**: 3 tipos (SWT, PE, DEFAULT)
- **Produtos**: 4 tipos (DEFAULT, DEDICATED_IP, VPN, SIP)
- **Tecnologias**: 4 tipos (GPON, SWT, FSP, IPV6)

### Distribuição de Tasks

| Pipeline Type | Task Count | Complexity |
|---------------|------------|------------|
| IPv6 | 1 | Baixa |
| SWT + Dedicated IP | 5 | Média |
| PE + Default | 5 | Média |
| PE + VPN (all tech) | 7 | Alta |
| PE + SIP (all tech) | 8 | Muito Alta |

### Tasks Mais Utilizadas

1. `DuplicityCheckResultTask` - Presente em 8 pipelines
2. `GetEquipmentDetailsTask` - Presente em 7 pipelines
3. `GetCustomerNameDuplicityCheckTask` - Presente em 6 pipelines
4. `DuplicityCheckTask` - Presente em 4 pipelines
5. `GetCompletePeInterfaceDuplicityCheckTask` - Presente em 6 pipelines

---

## 🎯 Decisões de Design

### Por que múltiplos pipelines?

1. **Separação de Concerns**: Cada tipo de equipamento/produto tem necessidades específicas
2. **Performance**: Executa apenas as tasks necessárias para cada cenário
3. **Manutenibilidade**: Fácil localizar e modificar lógica específica
4. **Testabilidade**: Cada pipeline pode ser testado independentemente
5. **Escalabilidade**: Novos pipelines não afetam os existentes

### Por que task-based architecture?

1. **Reutilização**: Tasks podem ser compartilhadas entre pipelines
2. **Composição**: Fácil criar novos pipelines combinando tasks existentes
3. **Isolamento**: Cada task tem responsabilidade única e bem definida
4. **Debugging**: Fácil identificar qual task falhou no pipeline
5. **Evolução**: Novas tasks podem ser adicionadas sem modificar existentes

### Por que Spring Component?

1. **Dependency Injection**: Todas as tasks são injetadas automaticamente
2. **Lifecycle Management**: Spring gerencia criação e destruição
3. **Testing**: Fácil mockar dependências em testes
4. **Configuration**: Integração com configurações do Spring
5. **Monitoring**: Integração com métricas e observabilidade

---

## 🔄 Fluxo de Integração

```mermaid
sequenceDiagram
    participant User as 👤 User
    participant UI as 🖥️ UI Application
    participant PM as 📋 Process Manager
    participant PL as DuplicityCheckPipelines
    participant TK as 🔧 Task Executors
    participant NW as 🌐 Network Devices
    participant DB as 💾 Database

    User->>UI: Request service provisioning
    UI->>PM: Submit provisioning request
    PM->>PM: Validate request data

    PM->>PL: Request DUPLICITY_CHECK<br/>with equipment/product/tech

    activate PL
    PL->>PL: Determine pipeline<br/>based on request type

    PL->>TK: Execute pipeline tasks sequentially
    activate TK

    loop For each task
        TK->>NW: Query network device
        NW-->>TK: Device response
        TK->>DB: Store intermediate results
        TK->>TK: Process data
    end

    TK-->>PL: All tasks completed
    deactivate TK

    PL->>PL: Consolidate results
    PL-->>PM: Return duplicity check result

    deactivate PL

    PM->>PM: Evaluate result

    alt No duplicity found
        PM->>UI: ✅ Proceed with provisioning
        UI->>User: Service approved
    else Duplicity detected
        PM->>UI: ❌ Block provisioning
        UI->>User: Service rejected - conflict detected
    end
```

---

## 📝 Exemplo de Código de Uso

### Registering a New Pipeline

```java
new Pipeline(
    PipelineActions.DUPLICITY_CHECK + "_" +
        EquipmentTypeEnum.PE.getIdEquipmentType() + "_" +
        ProductTypeEnum.VPN.getIdProductType() + "_" +
        TecnologyEnum.GPON.getCode(),
    List.of(
        duplicityCheckTask,
        duplicityCheckSipVrfTask,
        duplicityCheckSipVpnTask,
        getEquipmentDetailsTask,
        getCompletePeInterfaceDuplicityCheckTask,
        getCustomerNameDuplicityCheckTask,
        duplicityCheckResultTask
    ),
    DuplicityCheckInputs.class
)
```

### Pipeline Key Format

```
DUPLICITY_CHECK_{equipmentId}_{productId}_{technologyCode}

Examples:
- DUPLICITY_CHECK_2_1_GPON    (SWT + Dedicated IP + GPON)
- DUPLICITY_CHECK_3_3_FSP     (PE + SIP + FSP)
- DUPLICITY_CHECK_0_0_IPV6    (Default + Default + IPv6)
```

---

**Última atualização:** 30 de Outubro de 2025

