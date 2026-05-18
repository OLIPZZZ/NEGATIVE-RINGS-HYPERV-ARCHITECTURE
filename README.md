# 🔬 Anéis Negativos e Hyper-V: Arquitetura de Privilégios Computacionais

> *"A segurança e o isolamento de processos em sistemas computacionais modernos são fundamentados em hierarquias de privilégios suportadas e impostas diretamente pelo hardware."*

Repositório de análise arquitetural sobre o paradigma dos **Anéis Negativos (IA Negative Rings)**, os níveis de privilégio de hardware em arquiteturas x86/x64 e a arquitetura do **Microsoft Hyper-V**. Cobre a correspondência entre o Ring -1 e a citação de Pavel Yosifovich sobre o Hyper-V, revelando o abismo entre a nomenclatura de mercado e o rigor técnico da arquitetura de microprocessadores.

> 📚 Conteúdo para fins educacionais e de pesquisa em sistemas operacionais, virtualização e segurança de baixo nível.

---

## 📁 Índice

- [1. Fundamentos da Arquitetura de Privilégios](#1-fundamentos-da-arquitetura-de-privilégios)
- [2. O Modelo Clássico de Anéis de Proteção x86](#2-o-modelo-clássico-de-anéis-de-proteção-x86)
- [3. O Gargalo da Virtualização e os Anéis Negativos](#3-o-gargalo-da-virtualização-e-os-anéis-negativos)
  - [3.1 Ring -1 — Hypervisor](#31-ring--1--hypervisor)
  - [3.2 Ring -2 — System Management Mode (SMM)](#32-ring--2--system-management-mode-smm)
  - [3.3 Ring -3 — Management Engine (ME)](#33-ring--3--management-engine-me)
- [4. A Arquitetura do Microsoft Hyper-V](#4-a-arquitetura-do-microsoft-hyper-v)
  - [4.1 Partições e o Papel do Hipervisor](#41-partições-e-o-papel-do-hipervisor)
  - [4.2 Hypercalls e a Interface de Virtualização](#42-hypercalls-e-a-interface-de-virtualização)
- [5. Implicações de Segurança](#5-implicações-de-segurança)
- [Glossário Técnico](#glossário-técnico)

---

## 1. Fundamentos da Arquitetura de Privilégios

A segurança e o isolamento de processos em sistemas computacionais modernos são fundamentados em hierarquias de privilégios suportadas e impostas diretamente pelo hardware.

A **premissa central** deste modelo arquitetural é garantir a tolerância a falhas e a segurança intrínseca do sistema operacional, particionando de forma rígida:

- O **código de aplicativo de usuário** (considerado não confiável ou propenso a erros)
- Das **funções críticas de controle de hardware**, escalonamento de processos e gerenciamento de memória física e virtual

A lógica de hardware subjacente impõe que os processos em execução em um determinado nível de privilégio — determinado fisicamente pelos bits de **Descriptor Privilege Level (DPL)** nos seletores de segmento da CPU — só possam acessar a memória e os recursos correspondentes àquele nível ou a níveis numericamente superiores (domínios menos privilegiados).

---

## 2. O Modelo Clássico de Anéis de Proteção x86

A concepção original da arquitetura x86 previu a implementação de **quatro níveis de privilégio** estritamente positivos, numerados hierarquicamente de 0 a 3, onde o grau de privilégio e controle sobre o hardware é **inversamente proporcional ao número do anel**.

| Anel de Proteção | Nomenclatura Comum | Nível de Privilégio (DPL) | Escopo Operacional |
|-----------------|-------------------|--------------------------|-------------------|
| **Ring 0** | Modo Kernel / Supervisor | Máximo (DPL 0) | Nível de maior privilégio do modelo x86 clássico. Detém controle irrestrito, acesso total às instruções sensíveis da CPU, aos registradores de controle e a todo o mapeamento de memória. Domínio exclusivo do núcleo do sistema operacional e drivers de dispositivo base. |
| **Ring 1** | Modo Privilegiado Nível 1 | Intermediário Alto (DPL 1) | Historicamente subutilizado. Empregado pontualmente por interfaces como o DPMI do Windows 3.0. Na virtualização via paravirtualização (Xen), hospeda o Kernel do sistema operacional convidado para evitar o ring aliasing. |
| **Ring 2** | Modo Privilegiado Nível 2 | Intermediário Baixo (DPL 2) | Historicamente empregado pelo sistema operacional OS/2 para código privilegiado com permissões estritas de Entrada/Saída (I/O). Praticamente obsoleto em sistemas modernos. |
| **Ring 3** | Modo de Usuário (User Mode) | Mínimo (DPL 3) | Aplicativos operam de forma isolada (sandbox), sem capacidade de acessar hardware diretamente. Dependem exclusivamente de chamadas de sistema (syscalls) para delegar operações ao Ring 0. |

> A rigidez deste modelo provou-se altamente eficaz por décadas, sustentando a arquitetura de sistemas operacionais modernos como o **Windows NT** e o **Linux**.

### Por que só Ring 0 e Ring 3 são usados na prática?

Tradicionalmente, o desenvolvimento de sistemas operacionais adotou uma **abordagem essencialmente binária**, descartando os níveis intermediários para:
- Simplificar o design do kernel
- Garantir portabilidade de código entre arquiteturas que poderiam não suportar quatro anéis distintos

---

## 3. O Gargalo da Virtualização e os Anéis Negativos

A introdução e a necessidade premente de **consolidação de servidores** através da virtualização de hardware criaram um gargalo arquitetural crítico no modelo clássico de quatro anéis.

Os sistemas operacionais convidados (Guest OS) — sejam eles instâncias virtualizadas de Windows ou Linux — foram originalmente projetados e compilados sob a premissa inegociável de que possuíam controle exclusivo do Ring 0. Quando executados dentro de um ambiente virtualizado, esse pressuposto gera conflito direto com o controle do hardware físico real.

A solução foi a introdução dos **Anéis Negativos** — níveis de privilégio abaixo do Ring 0 — habilitados por extensões de hardware como **Intel VT-x** e **AMD-V (SVM)**.

```
┌─────────────────────────────────────────────────┐
│                    Ring -3                       │
│         Intel Management Engine (ME)            │
│         AMD Platform Security Processor (PSP)   │
├─────────────────────────────────────────────────┤
│                    Ring -2                       │
│         System Management Mode (SMM)            │
│         (Firmware / BIOS / UEFI)                │
├─────────────────────────────────────────────────┤
│                    Ring -1                       │
│         Hypervisor (VMX root mode)              │
│         Microsoft Hyper-V / VMware / KVM        │
├─────────────────────────────────────────────────┤
│                    Ring 0                        │
│         Kernel do Sistema Operacional           │
│         Drivers de Dispositivo                  │
├─────────────────────────────────────────────────┤
│                    Ring 1                        │
│         (Praticamente não utilizado)            │
├─────────────────────────────────────────────────┤
│                    Ring 2                        │
│         (Praticamente não utilizado)            │
├─────────────────────────────────────────────────┤
│                    Ring 3                        │
│         Aplicativos de Usuário                  │
│         Processos não privilegiados             │
└─────────────────────────────────────────────────┘
```

### 3.1 Ring -1 — Hypervisor

O **Ring -1** é o nível de privilégio onde o **Hypervisor** opera quando a virtualização assistida por hardware está habilitada.

**Características fundamentais:**

- Opera em **VMX root mode** (Intel) ou **SVM mode** (AMD)
- Intercepta e controla o acesso das VMs ao hardware físico
- Gerencia a criação, execução e encerramento de máquinas virtuais
- Mantém o controle de registradores de controle críticos (CR0, CR3, CR4) e tabelas de descriptores (GDT, IDT)

**Por que Ring -1 e não Ring 0?**

Quando a virtualização assistida por hardware foi introduzida com as extensões **Intel VT-x (Virtualization Technology for IA-32/64)**, tornou-se necessário criar um novo nível de privilégio *abaixo* do Ring 0 convencional. Isso é necessário porque:

1. O Guest OS (sistema operacional convidado) acredita estar operando no Ring 0
2. O Hypervisor precisa de um nível ainda mais privilegiado para interceptar e controlar esse Guest OS
3. A CPU, ao detectar a tentativa do Guest OS de executar instruções sensíveis (como `VMXON`, `VMCALL`, etc.), realiza um **VM Exit** e transfere o controle para o Hypervisor no Ring -1

> **Citação de Pavel Yosifovich (autor de "Windows Kernel Programming"):** O Hyper-V opera em um nível de privilégio que não existia no modelo clássico x86 — o que a indústria convencionou chamar de Ring -1 ou VMX root mode.

### 3.2 Ring -2 — System Management Mode (SMM)

O **System Management Mode (SMM)** representa um nível de privilégio ainda mais profundo, operando em território de firmware.

**Características:**

- Introduzido originalmente pela Intel no processador 386SL (1990)
- É ativado através de uma interrupção especial chamada **SMI (System Management Interrupt)**
- Quando ativado, o processador para completamente o que está fazendo e executa o código de gerenciamento de sistema no **SMRAM (System Management RAM)** — uma região de memória isolada e protegida
- O sistema operacional, o Hypervisor e qualquer outro software no Ring -1 ou superior são **completamente invisíveis e inacessíveis** ao SMM

**Usos legítimos do SMM:**

- Gerenciamento de energia e controle térmico (APM/ACPI)
- Emulação de hardware legado (USB Legacy Support, virtualização de teclado PS/2)
- Controle de fan speed e voltagem
- Implementação de recursos de segurança como TPM e Secure Boot

**Implicações de segurança:**

O SMM é um vetor de ataque extremamente poderoso — e perigoso. Um código malicioso executando no SMM (comumente chamado de **SMM Rootkit** ou **BIOS Rootkit**):

- É **invisível** a qualquer antivírus, EDR ou Hypervisor
- Persiste através de reinstalações de sistema operacional
- Pode sobreviver a formatações de disco
- Tem acesso irrestrito a toda a memória física do sistema

Ataques conhecidos explorando SMM incluem o **ThinkPwn** (2016) e vulnerabilidades documentadas em implementações de UEFI.

### 3.3 Ring -3 — Management Engine (ME)

O **Intel Management Engine (ME)** — ou **AMD Platform Security Processor (PSP)** na arquitetura AMD — representa o nível mais profundo e controverso da hierarquia de privilégios.

**O que é o Management Engine:**

- Um microprocessador autônomo separado embutido dentro do chipset Intel (e dos próprios processadores modernos desde a arquitetura Sandy Bridge)
- Executa seu próprio sistema operacional baseado em **MINIX 3** (no caso Intel ME)
- Opera **independentemente da CPU principal** e do sistema operacional instalado
- Possui acesso direto à memória física, interface de rede e outros periféricos **mesmo com o computador aparentemente desligado** (desde que haja energia de standby)

**Funcionalidades do Intel ME:**

- **Intel AMT (Active Management Technology):** Gerenciamento remoto out-of-band — permite controle total do sistema mesmo com o SO desligado ou corrompido
- **Intel Boot Guard:** Garante a integridade da cadeia de inicialização (Secure Boot)
- **Intel PTT (Platform Trust Technology):** Implementação de TPM via firmware (TPM 2.0 virtual)
- **Intel TXT (Trusted Execution Technology):** Criação de ambientes de execução confiáveis

**Por que Ring -3?**

O ME opera em um nível de privilégio tão fundamental que:
- Não pode ser desligado pelo usuário (em versões modernas)
- Não pode ser inspecionado pelo sistema operacional
- Possui acesso a regiões de memória inacessíveis ao próprio Hypervisor
- Pode executar código mesmo quando a CPU principal está em estado de halt

> **Controvérsia:** Pesquisadores de segurança como Damien Zammit (2016), Igor Skochinsky e Mark Ermolov descobriram múltiplas vulnerabilidades no ME ao longo dos anos, incluindo a crítica **SA-00086** (2017), que permitia execução arbitrária de código no Ring -3 remotamente.

---

## 4. A Arquitetura do Microsoft Hyper-V

O **Microsoft Hyper-V** é um Hypervisor do Tipo 1 (bare-metal) que implementa a virtualização assistida por hardware através das extensões Intel VT-x e AMD-V.

### 4.1 Partições e o Papel do Hipervisor

O Hyper-V organiza a execução de sistemas operacionais no conceito de **Partições**:

```
┌──────────────────────────────────────────────────────────────┐
│                    Hyper-V Hypervisor                        │
│                    (VMX Root / Ring -1)                      │
├─────────────────────────┬────────────────────────────────────┤
│    Partição Raiz         │     Partições Filho (VMs)         │
│    (Root Partition)      │     (Child Partitions)            │
│                         │                                    │
│  ┌─────────────────┐   │   ┌──────────────────────────┐    │
│  │  Windows Server │   │   │  Guest OS (Windows/Linux) │    │
│  │  (Ring 0)       │   │   │  (Ring 0 virtualizado)    │    │
│  ├─────────────────┤   │   ├──────────────────────────┤    │
│  │  VMBus          │   │   │  VMBus / Enlightenments   │    │
│  │  Drivers VSP    │   │   │  Drivers VSC              │    │
│  └─────────────────┘   │   └──────────────────────────┘    │
└─────────────────────────┴────────────────────────────────────┘
```

**Partição Raiz (Root Partition):**
- A primeira e mais privilegiada partição Hyper-V
- Executa o Windows Server como sistema operacional de gerenciamento
- Possui acesso direto ao hardware físico através dos **VSPs (Virtualization Service Providers)**
- Hospeda o **Hyper-V Manager** e a **WMI Provider** para gerenciamento das VMs

**Partições Filho (Child Partitions / VMs):**
- Executam os sistemas operacionais convidados (Guest OS)
- Não têm acesso direto ao hardware físico
- Comunicam-se com a Partição Raiz através do **VMBus** para acesso virtualizado a I/O (disco, rede, etc.)
- Com os **Hyper-V Integration Services** instalados, utilizam os **VSCs (Virtualization Service Clients)** para comunicação eficiente com os VSPs

### 4.2 Hypercalls e a Interface de Virtualização

Quando o Guest OS precisa realizar operações privilegiadas que normalmente acessariam o hardware diretamente (e que o Hypervisor precisa interceptar e mediar), o mecanismo utilizado são as **Hypercalls** — o equivalente, no contexto de virtualização, das Syscalls no contexto kernel-to-user.

**Mecanismo de VM Exit:**

```
Guest OS executa instrução sensível
           │
           ▼
    CPU detecta conflito
           │
           ▼
    VM Exit: CPU salva estado do Guest
           │
           ▼
    Controle transferido para Hypervisor (Ring -1)
           │
           ▼
    Hypervisor processa a operação
    (emula, delega ou bloqueia)
           │
           ▼
    VM Entry: CPU restaura estado do Guest
           │
           ▼
    Guest OS continua execução
```

**Hyper-V Enlightenments:**

Os **Enlightenments** são otimizações que permitem ao Guest OS "saber" que está rodando dentro do Hyper-V e utilizar interfaces mais eficientes ao invés das instruções x86 padrão que causariam VM Exits custosos:

- `HV_X64_MSR_GUEST_OS_ID` — Identificação do Guest OS
- `HvCallNotifyLongSpinWait` — Otimização de spinlocks
- `HvCallSendSyntheticClusterIpi` — IPIs sintéticos eficientes
- `HvCallFlushVirtualAddressList` — TLB flush otimizado

---

## 5. Implicações de Segurança

### Virtualização Baseada em Segurança (VBS) — Windows 10/11 e Server 2016+

A Microsoft implementou a **Virtualization-Based Security (VBS)** para utilizar o Hyper-V como um mecanismo de segurança, não apenas de virtualização:

```
┌─────────────────────────────────────────────────────┐
│              Hyper-V Hypervisor (Ring -1)           │
├──────────────────┬──────────────────────────────────┤
│  Normal World     │      Secure World               │
│  (VTL 0)         │      (VTL 1 - IUM)              │
│                  │                                  │
│  Windows Kernel  │  Secure Kernel (skci.dll)       │
│  (Ring 0)        │  LSA (lsaiso.exe)               │
│                  │  Credential Guard               │
│  Aplicativos     │  Code Integrity (HVCI)          │
│  (Ring 3)        │                                  │
└──────────────────┴──────────────────────────────────┘
```

**Virtual Trust Levels (VTL):**

O Hyper-V implementa dois Virtual Trust Levels:
- **VTL 0 (Normal World):** Onde o Windows normal e os aplicativos executam
- **VTL 1 (Secure World / IUM - Isolated User Mode):** Onde componentes de segurança críticos como o **Credential Guard** (proteção do LSASS) e o **Device Guard (HVCI)** executam

Um compromisso completo do Ring 0 no VTL 0 (ex: rootkit de kernel) **não consegue acessar** os segredos armazenados no VTL 1.

### Ataques Visando Anéis Negativos

| Ring | Tipo de Ataque | Exemplo Real | Impacto |
|------|---------------|-------------|---------|
| Ring -1 | Hypervisor Escape | CVE-2021-28476 (Hyper-V RCE) | VM escapa para o host |
| Ring -2 | SMM Rootkit | ThinkPwn (2016), SA-00086 (2017) | Persistência invisível |
| Ring -3 | ME Exploit | Intel ME SA-00086 | Controle total remoto |

### Mitigações Modernas

- **Secure Boot + Boot Guard:** Garante integridade da cadeia de inicialização desde o firmware
- **HVCI (Hypervisor-Protected Code Integrity):** Impede injeção de código não assinado no kernel
- **IOMMU/VT-d:** Protege contra ataques DMA (Direct Memory Access) de dispositivos periféricos maliciosos
- **TPM 2.0:** Armazena chaves criptográficas em hardware dedicado
- **Kernel DMA Protection:** Bloqueia acesso DMA não autorizado de dispositivos Thunderbolt/PCI-e

---

## 📖 Glossário Técnico

| Termo | Definição |
|-------|-----------|
| **DPL (Descriptor Privilege Level)** | Bits nos seletores de segmento da CPU que determinam o nível de privilégio de execução |
| **VMX (Virtual Machine Extensions)** | Extensões Intel VT-x que habilitam a virtualização assistida por hardware |
| **VMX Root Mode** | Modo de operação do processador onde o Hypervisor executa (Ring -1) |
| **VM Exit** | Transferência de controle do Guest OS para o Hypervisor, disparada por instruções sensíveis |
| **VM Entry** | Transferência de controle do Hypervisor de volta para o Guest OS |
| **SMI (System Management Interrupt)** | Interrupção de hardware não mascarável que ativa o SMM |
| **SMRAM** | Região de memória reservada e protegida onde o código SMM executa |
| **Hypercall** | Interface de chamada de sistema entre Guest OS e Hypervisor |
| **VMBus** | Barramento de comunicação virtual entre Partição Raiz e Partições Filho no Hyper-V |
| **VSP (Virtualization Service Provider)** | Componente da Partição Raiz que fornece serviços virtualizados às VMs |
| **VSC (Virtualization Service Client)** | Componente do Guest OS que consome serviços do VSP via VMBus |
| **VTL (Virtual Trust Level)** | Nível de confiança virtual implementado pelo Hyper-V para VBS |
| **IOMMU (Input-Output Memory Management Unit)** | Unidade de gerenciamento de memória para dispositivos I/O, protege contra ataques DMA |
| **TLB (Translation Lookaside Buffer)** | Cache de hardware para traduções de endereços virtuais para físicos |
| **Ring Aliasing** | Problema em virtualização onde o Guest OS operando no Ring 1 (ao invés do Ring 0) pode ser detectado |

---

<div align="center">

⚙️ **Systems Architecture | Low-Level Security | Virtualization Research**

[![Debian](https://img.shields.io/badge/Debian-D70A53?style=for-the-badge&logo=debian&logoColor=white)](https://debian.org)
[![Intel](https://img.shields.io/badge/Intel-0071C5?style=for-the-badge&logo=intel&logoColor=white)](https://intel.com)

</div>
