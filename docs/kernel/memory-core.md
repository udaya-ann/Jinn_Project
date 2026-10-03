# Memory Core — Jinn OS

## 1. Visão Geral

O `Memory Core` é o subsistema responsável por gerenciar memória física e virtual do Jinn OS, fornecendo um modelo eficiente e seguro de alocação. Diferentemente de kernels monolíticos, Jinn centraliza decisões de alocação em um subsistema verificável, utilizando múltiplos alocadores otimizados (bitmap, buddy, slab) e pools especializados.

### Princípios de Design

- **Isolamento:** Memória isolada por domínio de segurança (capacidades)
- **Eficiência:** Múltiplos alocadores para diferentes tamanhos e padrões de uso
- **Previsibilidade:** Pools proativos via Predictive Engine reduzem fragmentação
- **Observabilidade:** Telemetria de uso para otimização
- **Escalabilidade:** Preparado para NUMA com localidade de nó

## 2. Objetivos

- ✅ Gerenciar memória física e virtual para kernel e serviços isolados
- ✅ Suportar alocadores eficientes (bitmap, buddy, slab) para diferentes casos de uso
- ✅ Integrar com Predictive Engine para pools proativos (cache warming)
- ✅ Garantir isolamento e segurança através de capabilities
- ✅ Suportar compartilhamento seguro entre domínios
- ✅ Minimizar fragmentação com compactação e page coalescing
- ✅ Oferecer latência de alocação previsível (<10μs para common paths)

## 3. Responsabilidades

- **Physical Memory Management:** Rastreamento de páginas físicas, buddy allocator
- **Virtual Memory:** Page table management, TLB invalidation, paging policies
- **Alocadores especializados:** Slab (objetos pequenos), buddy (páginas), bitmap (bootstrap)
- **Security:** Verificação de capabilities antes de acesso/compartilhamento
- **NUMA Awareness:** Localidade de dados e migração proativa
- **Pools Especializados:** DMA, Cache, Predictive
- **Telemetria:** Coleta de métricas para Predictive Engine

## 4. Arquitetura Híbrida

O Memory Core usa múltiplas camadas de alocadores, cada otimizado para um caso de uso específico:

```
┌───────────────────────────────────────────────────────────────┐
│                  Predictive Engine                            │
│            (Hints: cache warming, reservas)                  │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│           Pools Especializados                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│  │ DMA Pool │  │Cache Pool│  │Predictive│ (reservas)      │
│  │ (contig) │  │(warming) │  │  Pool    │                 │
│  └──────────┘  └──────────┘  └──────────┘                │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│        Slab Allocator (Small Objects)                       │
│  • Size classes: 8, 16, 32, 64, 128, 256, 512, 1K        │
│  • Per-CPU caches, partial/full slabs                      │
│  • O(1) alloc/free                                         │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│        Buddy Allocator (Pages)                              │
│  • Coalesces adjacent free blocks to power-of-2 sizes      │
│  • Free lists from 4KB to 1GB (order 0-18)                │
│  • Per-NUMA-node                                           │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│      Bitmap Allocator (Bootstrap Only)                      │
│  • Used during boot before buddy is ready                  │
│  • Fast but wasteful; immediately moved to buddy           │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│       Virtual Memory Manager (Page Tables)                  │
│  • 4-level page table hierarchy (x86_64)                   │
│  • COW (Copy-on-Write) support                             │
│  • Lazy paging via Predictive Engine                       │
└─────────────────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│          Physical Memory (RAM)                              │
│  4KB pages, NUMA-aware, organized by zones                │
└─────────────────────────────────────────────────────────────┘
```

### Camadas de Abstração

**1. Physical Memory Manager**
- Rastreia páginas físicas, sua propriedade e estado
- Interfaces: `alloc_pages()`, `free_pages()`
- Otimizações: buddy allocator com compactação

**2. Virtual Memory Manager**
- Mapeia espaços de endereço virtual para físico
- Gerencia page tables e TLB
- Interfaces: `map_pages()`, `unmap_pages()`, `remap_pages()`

**3. Capability System**
- Media acesso a páginas através de capabilities
- Cada tarefa tem set de capabilities vinculadas
- Interfaces: `grant_capability()`, `revoke_capability()`

**4. Pool Managers**
- Caches especializados para access patterns conhecidos
- Alimentados por Predictive Engine
- Interfaces: `alloc_from_pool()`, `reserve_pool()`

## 5. Estruturas de Dados Principais

### Physical Memory Structures

```rust
// Page frame descriptor
pub struct Page {
    // 64 bytes (fits in L1 cache)
    physical_address: PhysicalAddress,
    
    // Ownership and state
    owner: Option<DomainId>,
    state: PageState,  // Free, Used, Reserved, Shared
    
    // Reference counting
    ref_count: AtomicU32,
    pinned: bool,  // Don't swap out
    
    // NUMA locality
    numa_node: u8,
    last_accessed_cpu: AtomicU8,
    
    // Mapping info
    vaddr: Option<VirtualAddress>,  // For quick reverse lookup
    
    // Flags
    dirty: bool,
    accessible: bool,
    writable: bool,
}

pub enum PageState {
    Free,           // Not allocated
    Used,           // Allocated
    Reserved,       // Reserved by Predictive Engine
    Shared,         // Shared between domains
    SwappedOut,     // On disk
}

// Physical memory region
pub struct PhysicalRegion {
    start: PhysicalAddress,
    size: usize,        // bytes
    numa_node: u8,
    pages: Vec<Page>,   // or bitmap
}

// Buddy allocator node
pub struct BuddyAllocator {
    order: u8,  // log2 of size (0=4K, 1=8K, ..., 18=1GB)
    free_list: LockFreeLinkedList<PageBlock>,
    metrics: AllocationMetrics,
}

pub struct PageBlock {
    start_page: usize,
    order: u8,
    state: BlockState,
}

pub enum BlockState {
    Free,       // Available for allocation
    Allocated,  // In use
    Merging,    // Being coalesced with buddy
}
```

### Virtual Memory Structures

```rust
// Page table entry (x86_64)
pub struct PTE {
    // Bits 0-11: Flags
    present: bool,
    writable: bool,
    user: bool,
    write_through: bool,
    cache_disabled: bool,
    accessed: bool,
    dirty: bool,
    huge_page: bool,
    global: bool,
    
    // Bits 12-51: Physical page number
    physical_page_number: u64,
    
    // Bits 52-62: Software-available
    software_flags: u16,
    
    // Bit 63: Execute-disable
    no_execute: bool,
}

// Virtual address space descriptor
pub struct VirtualAddressSpace {
    domain_id: DomainId,
    cr3: PhysicalAddress,  // Page table base
    
    // Page table cache
    l4_cache: Option<&'static PTE>,  // PML4T
    l3_cache: Option<&'static PTE>,  // PDPT
    l2_cache: Option<&'static PTE>,  // PDT
    
    // Permissions
    capabilities: CapabilitySet,
    
    // Metrics
    pages_mapped: AtomicUsize,
    cow_faults: AtomicU64,
    tlb_misses: AtomicU64,
}

// Capability para página/região
pub struct PageCapability {
    owner: DomainId,
    target_page: PhysicalAddress,
    size_order: u8,  // log2 of region size
    
    permissions: PagePermissions,
    created_at: Timestamp,
    
    // Tracking
    share_count: AtomicU32,
    last_used: AtomicU64,
}

pub struct PagePermissions {
    read: bool,
    write: bool,
    execute: bool,
    can_share: bool,
    can_delegate: bool,
}
```

### Slab Allocator Structures

```rust
pub struct SlabAllocator {
    // Size class information
    object_size: usize,
    objects_per_slab: usize,
    
    // Lists of slabs in different states
    full_slabs: Vec<Slab>,
    partial_slabs: Vec<Slab>,
    empty_slabs: Vec<Slab>,
    
    // Per-CPU cache for fast allocation
    cpu_caches: Vec<CpuSlabCache>,
    
    // Metrics
    allocations: AtomicU64,
    deallocations: AtomicU64,
    fragmentation: AtomicU32,
}

pub struct Slab {
    // Ownership and bookkeeping
    allocator: &'static SlabAllocator,
    start_page: PhysicalAddress,
    objects: Vec<SlabObject>,
    
    // Bitmap of free slots
    free_bitmap: BitMap,
    free_count: AtomicU16,
    
    // Color (cache-line offset for better cache behavior)
    color: u16,
}

pub struct SlabObject {
    offset_in_slab: u16,
    size: u16,
    free: bool,
}

// Per-CPU allocation cache (reduces lock contention)
pub struct CpuSlabCache {
    current_slab: Option<&'static Slab>,
    free_objects: Vec<SlabObject>,
    hot_reserve: u16,  // Reserva para allocs rápidas
}
```

## 6. Fluxos de Execução

### Fluxo de Alocação de Página

```
1. Serviço user-space requisita página via IPC
   │
   └─ mm_alloc_pages(count, flags)
      │
2. Memory Core verifica quota do serviço
   │
   ├─ Se excedida: return ENOMEM
   │
3. Seleciona NUMA node (policy: local, round-robin, spread)
   │
4. Buddy Allocator procura bloco livre
   │
   ├─ Se encontrado: retorna imediatamente (O(1))
   │
   ├─ Se não encontrado: tenta order maior
   │     └─ Divide (split) em blocos menores
   │
   ├─ Se ainda não há: tenta compactação
   │     ├─ Move páginas limpas para liberar espaço
   │     └─ Retry allocator
   │
5. Marca página como "Allocated", registra owner
   │
6. Cria capability de acesso
   │
7. Mapeia virtual para domínio do serviço
   │
8. Retorna (vaddr, paddr) via IPC
```

Diagrama:

```
User Service          Kernel Memory Core          Buddy Allocator
    │                      │                            │
    ├─ mm_alloc_pages() ──►│                            │
    │                      ├─ check_quota()            │
    │                      │                            │
    │                      ├─ select_numa_node()      │
    │                      │                            │
    │                      ├─ alloc_from_buddy() ─────►│
    │                      │                   ┌─ Find free block
    │                      │◄─ (pages) ────────┘
    │                      │
    │                      ├─ create_capability()
    │                      │
    │                      ├─ map_virtual()
    │                      │
    │◄─ (vaddr, paddr) ───┤
    │                      │
    ├─ [execution]         │
    │                      │
    └─ Write to page ─────►│
                           ├─ Page fault handler
                           │  (if COW or unmapped)
```

### Fluxo de Copy-on-Write (CoW)

```
1. Parent cria child via fork/spawn
   │
2. Memory Core marca todas páginas do parent como CoW
   │   (ambos parent e child apontam para mesmas páginas)
   │
3. Child executa, tenta escrever em página CoW
   │
4. MMU gera page fault
   │
5. Memory Core page fault handler:
   │
   ├─ Verifica se a página está marcada como CoW
   │
   ├─ Aloca nova página via buddy
   │
   ├─ Copia conteúdo da página original
   │
   ├─ Atualiza page table do child
   │
   ├─ Desmarca CoW flag
   │
   └─ Retorna para execução
   │
6. Child pode agora escrever livremente
```

### Fluxo de Pool Proativo (Predictive Engine)

```
1. Predictive Engine analisa workload
   │   ├─ "Serviço X vai precisar ~100 páginas em 50ms"
   │   └─ "Cache warming: prepare páginas para padrão de acesso Y"
   │
2. Emite PredictionHint para Memory Core
   │
3. Memory Core cria Predictive Pool
   │   ├─ Reserve 100 páginas do buddy
   │   ├─ Mark as "Reserved"
   │   ├─ Possível: warm com dados esperados
   │   └─ Marca com timestamp
   │
4. 50ms depois, quando serviço realmente requisita
   │
5. Memory Core aloca do pool (super rápido, já está pronto)
   │   └─ Benchmark: ~100ns vs ~1μs normal
   │
6. Se pool não foi usado em timeout, retorna ao buddy
```

### Fluxo de Compartilhamento de Memória

```
Domain A                Kernel Memory Core           Domain B
  │                            │                        │
  ├─ share_page() ────────────►│                        │
  │  (page, domainB, perms)    │                        │
  │                            ├─ Verifica capability de A
  │                            │                        │
  │                            ├─ Cria nova capability
  │                            │  com perms para B
  │                            │                        │
  │                            ├─ Maps vaddr em B ─────►│
  │                            │                    ┌─ Acesso
  │                            │◄───────────────────┘
  │                            │
  │  (escritas simultâneas     │
  │   podem causar conflitos   |
  |   de cache)                │
```

## 7. Interfaces Públicas

### Syscalls de Memória

```rust
/// Aloca pages contíguas do buddy allocator
pub extern "C" fn mm_alloc_pages(
    count: usize,
    flags: MemoryFlags,
) -> Result<(VirtualAddress, PhysicalAddress), MemoryError>;

/// Libera páginas alocadas
pub extern "C" fn mm_free_pages(
    vaddr: VirtualAddress,
    count: usize,
) -> Result<(), MemoryError>;

/// Mapeia páginas entre domínios
pub extern "C" fn mm_map_pages(
    target_domain: DomainId,
    vaddr: VirtualAddress,
    paddr: PhysicalAddress,
    count: usize,
    perms: PagePermissions,
) -> Result<(), MemoryError>;

/// Desmapeia páginas de um domínio
pub extern "C" fn mm_unmap_pages(
    vaddr: VirtualAddress,
    count: usize,
) -> Result<(), MemoryError>;

/// Compartilha página entre domínios
pub extern "C" fn mm_share_page(
    target_domain: DomainId,
    vaddr: VirtualAddress,
    perms: PagePermissions,
) -> Result<RemoteVirtualAddress, MemoryError>;

/// Revoga acesso de domínio a página compartilhada
pub extern "C" fn mm_revoke_share(
    target_domain: DomainId,
    remote_vaddr: RemoteVirtualAddress,
) -> Result<(), MemoryError>;

/// Obtém informações sobre página
pub extern "C" fn mm_page_info(
    vaddr: VirtualAddress,
) -> Result<PageInfo, MemoryError>;

/// Reserva pool proativo para uso previsto
pub extern "C" fn mm_reserve_pool(
    pool_type: PoolType,
    size: usize,
    lifetime_ms: u64,
) -> Result<PoolHandle, MemoryError>;
```

### Funções Internas do Kernel

```rust
/// Inicializa memory core (durante boot)
pub fn mm_initialize(phys_ram: &PhysicalRegion) -> Result<(), MemoryError>;

/// Aloca da Slab cache (objetos pequenos)
pub fn mm_slab_alloc(size: usize) -> Result<*mut u8, MemoryError>;

/// Libera de Slab cache
pub fn mm_slab_free(ptr: *mut u8);

/// Gerencia page fault handler
pub fn mm_handle_page_fault(
    vaddr: VirtualAddress,
    access_type: AccessType,
    task: &Task,
) -> Result<(), PageFaultError>;

/// Compactação de páginas (usado quando memória está fragmentada)
pub fn mm_compact() -> usize;  // Retorna páginas compactadas

/// Enfileira a migração de uma página para um nó NUMA específico
pub fn mm_migrate_to_node(vaddr: VirtualAddress, node: u8) -> Result<(), MemoryError>;

/// Hook chamado pelo Predictive Engine
pub fn mm_on_predictive_hint(hint: &MemoryHint);

/// Coleta telemetria de uso de memória
pub fn mm_get_telemetry() -> MemoryTelemetry;
```

### Trait para Alocadores

```rust
pub trait MemoryAllocator: Send + Sync {
    /// Aloca bloco de memória de um determinado tamanho
    fn allocate(&mut self, size: usize, align: usize) -> Result<*mut u8, AllocError>;
    
    /// Libera bloco previamente alocado
    fn deallocate(&mut self, ptr: *mut u8, size: usize);
    
    /// Retorna fragmentação atual (0.0 = perfeito, 1.0 = máximo waste)
    fn fragmentation(&self) -> f32;
    
    /// Realiza compactação/desfragmentação
    fn compact(&mut self) -> usize;  // Retorna bytes recuperados
    
    /// Nome do alocador (debug)
    fn name(&self) -> &'static str;
}
```

## 8. Integração com Outros Componentes

### Integração com Scheduler

```
Scheduler         Memory Core          Predictive
    │                 │                     │
    ├─ task_create()  │                    │
    │                 ├─ alloc stack    ◄──┤
    │                 │  (via slab)          │
    │                 │
    ├─ task_running() │
    │                 ├─ page_fault?    ◄──┤ (affinity hint)
    │                 │  (migrate to best  │
    │                 │   NUMA node)        │
    │                 │
    └─ task_exit()    │
                      ├─ free all pages
                      │  (revoke shares)
```

### Integração com Predictive Engine

```
Predictive Engine
    │
    ├─ Hint: "Service X will need 500 pages soon"
    │  └─ Memory Core: Reserve pages in Predictive Pool
    │
    ├─ Hint: "Pagina common pattern will be X->Y access"
    │  └─ Memory Core: Pre-fetch/warm pages (futuro)
    │
    ├─ Hint: "Task T should be on NUMA node 1"
    │  └─ Memory Core: Schedule migration
    │
    └─ Hint: "Cache L3 under pressure"
       └─ Memory Core: Trigger compaction
```

### Integração com IPC Core

```
User Service
    │
    ├─ send_message(channel, buffer)
    │  └─ Memory Core: Validate vaddr access
    │                  (check capabilities)
    │
    │  └─ If remote domain: zero-copy via shared pages
    │                       (or create transient mapping)
    │
    └─ receive_message()
       └─ Memory Core: Grant temporary read access
                       to message buffer
```

## 9. Segurança e Isolamento

### Modelo de Capabilities

Toda página é protegida por capabilities. Apenas domínios com capability apropriada podem:

- Ler a página
- Escrever na página
- Compartilhar com outro domínio
- Delegar capability a terceiro

```rust
pub struct PageCapabilitySet {
    owner: DomainId,
    pages: HashMap<PhysicalAddress, PageCapability>,
}

impl PageCapabilitySet {
    /// Verifica se domínio tem permissão
    pub fn can_access(
        &self,
        domain: DomainId,
        page: PhysicalAddress,
        access: AccessType,
    ) -> bool {
        if let Some(cap) = self.pages.get(&page) {
            match access {
                AccessType::Read => cap.permissions.read && 
                                   (cap.owner == domain || cap.shared),
                AccessType::Write => cap.permissions.write && 
                                    cap.owner == domain,  // Write é sempre exclusive
                AccessType::Execute => cap.permissions.execute &&
                                      cap.owner == domain,
            }
        } else {
            false
        }
    }
}
```

### Proteção contra Access Faults

```rust
pub fn mm_handle_page_fault(
    vaddr: VirtualAddress,
    access: AccessType,
    task: &Task,
) -> Result<(), PageFaultError> {
    // 1. Resolve virtual → physical
    let paddr = task.address_space.translate(vaddr)?;
    
    // 2. Check capability
    if !can_access(task.domain_id, paddr, access) {
        return Err(PageFaultError::AccessDenied);
    }
    
    // 3. Check page state
    let page = get_page(paddr)?;
    match page.state {
        PageState::Used | PageState::Shared => {
            // Valid, continue
        }
        PageState::SwappedOut => {
            // Restore from disk
            restore_page_from_swap(paddr)?;
        }
        PageState::Free | PageState::Reserved => {
            // Tried to access unallocated memory
            return Err(PageFaultError::UseAfterFree);
        }
    }
    
    // 4. Handle special cases
    if page.is_cow() && access == AccessType::Write {
        handle_cow_fault(paddr, task)?;
    }
    
    // 5. TLB invalidation
    invalidate_tlb_entry(vaddr);
    
    Ok(())
}
```

### Proteção contra DMA Attacks

Páginas usadas para DMA devem ser:
- Pinned (não podem ser swapped)
- Contíguas na memória física
- Validadas contra IOMMU

```rust
pub fn mm_alloc_dma_pages(
    count: usize,
    max_physical_addr: u64,
) -> Result<(VirtualAddress, PhysicalAddress), MemoryError> {
    // 1. Allocate contiguous pages
    let pages = buddy.allocate_contiguous(count)?;
    
    // 2. Verify physical addresses are within limit
    for page in pages {
        if page.physical_address > max_physical_addr {
            buddy.deallocate(pages);
            return Err(MemoryError::DMAOutOfRange);
        }
    }
    
    // 3. Pin pages (prevent swapping)
    for page in pages {
        page.pinned = true;
    }
    
    // 4. Register with IOMMU
    iommu.register_region(pages);
    
    Ok((vaddr, paddr))
}
```

## 10. Escalabilidade e Limites

### Performance Targets

| Operação | Target | Implementação |
|----------|--------|----------------|
| Page allocation (hot path) | <500ns | Slab cache per-CPU |
| Page free | <500ns | Slab cache local |
| Page fault (no migration) | <10μs | Direct CoW copy |
| Page migration (NUMA) | <100μs | Async via balancer |
| Buddy coalescing | <1μs per block | Atomic flags |

### Overhead de Memória

```
Per Page:
  Page descriptor: 64 bytes
  Page table entry: 8 bytes
  
Per Slab (8-1024 bytes objects):
  Metadata: 256 bytes
  Bitmap: 128 bytes
  
NUMA awareness:
  Per node data: ~1 KB
  
Total per 4GB RAM: ~512 MB overhead (12.8%)
```

### Preparação para NUMA

```rust
pub struct NumaNode {
    id: u8,
    cpu_mask: u64,
    local_memory: PhysicalRegion,
    buddy_allocator: BuddyAllocator,
    
    // Cache local stats
    total_allocs: AtomicU64,
    remote_accesses: AtomicU64,
}

pub struct MemoryCoreMasked {
    nodes: Vec<NumaNode>,
    
    pub fn allocate_local(&mut self, cpuset: u64, size: usize) -> Result<PhysicalAddress, MemoryError> {
        let preferred_node = self.find_preferred_node(cpuset)?;
        preferred_node.buddy_allocator.allocate(size)
    }
    
    pub fn migrate_to_node(&mut self, page: PhysicalAddress, node: u8) -> Result<(), MemoryError> {
        // Async migration: update page table entries pointing
        // to this page to point to new location
        // Trigger on next access or via background thread
        Ok(())
    }
    
    fn find_preferred_node(&mut self, _cpuset: u64) -> Result<&mut NumaNode, MemoryError> {
        self.nodes.get_mut(0).ok_or(MemoryError::NoMemoryAvailable)
    }
}
```

## 11. Comparação com Sistemas Modernos

### Linux Kernel Memory Management vs Jinn

| Aspecto | Linux | Jinn |
|---------|-------|------|
| **Allocator** | Buddy + Slab/Slub | Buddy + Slab + Predictive Pool |
| **Complexity** | ~50k LOC (mm/) | ~5k LOC (alvo microkernel) |
| **NUMA Support** | Sim, altamente complexo | Sim, por-nó simplificado |
| **Capabilities** | Não (gerenciamento por processo/MMU tradicional) | Sim (modelo forte baseado em tokens/capabilities) |
| **Swapping** | Sim, complexo e integrado | Modular, isolado em serviço de user-space |
| **Transparency** | Políticas fixas no kernel | Políticas plugáveis orientadas por telemetria |
| **Fault Tolerance** | Moderate (panics em corrupção de heap) | High (isolamento e verificação formal) |

**Diferenças-chave:**
- O Linux integra memory management profundamente com VFS, swap e paging em anel 0.
- O Jinn isola o memory core em primitivas mínimas e verificáveis, delegando políticas a serviços e à Predictive Engine.
- O Jinn adiciona o Predictive Pool para aquecimento proativo de cache e pré-reserva de páginas.

### seL4 Memory Handling vs Jinn

| Aspecto | seL4 | Jinn |
|---------|------|------|
| **Formal Verification** | Sim (completo / Isabelle) | Em progresso (design verificável) |
| **Capability Model** | Untyped memory + CSpace | PageCapabilitySet unificado com IPC |
| **Allocators** | User-space gerencia toda alocação física | Híbrido: kernel gerencia Buddy/Slab, user-space gerencia objetos |
| **NUMA** | Não nativo | Sim (nós com alocadores locais) |
| **Performance** | Ótima em IPC e isolamento | Latência previsível com Predictive Engine |

### Zircon (Fuchsia) vs Jinn

| Aspecto | Zircon | Jinn |
|---------|--------|------|
| **Language** | C++ | Rust (memory-safety sem garbage collector) |
| **VMO (Virtual Memory Object)** | Sim (núcleo da abstração de memória) | Sim (VMOs gerenciados via capabilities) |
| **Predictive Engine** | Não | Sim (ajuste proativo de limites e warming) |

---

## 12. Pseudocódigo Avançado

### Algoritmo 1: Buddy Allocator (Alocação e Coalescência)

```pseudo
function buddy_alloc(order):
    // order = log2(páginas necessárias)
    // 1. Procura bloco livre na ordem solicitada ou superior
    for o = order to MAX_ORDER:
        if free_list[o] is not empty:
            block = free_list[o].pop()
            
            // Split progressivo até atingir a ordem desejada
            while o > order:
                o = o - 1
                buddy = split(block, o)
                free_list[o].push(buddy)
            
            mark_allocated(block)
            return block
    
    // Nenhum bloco disponível: tenta compactação
    if try_compact():
        return buddy_alloc(order)  // Retry pós-compactação
    
    return ERR_OUT_OF_MEMORY

function buddy_free(block, order):
    mark_free(block)
    
    // Tenta coalescer recursivamente com o buddy adjacente
    while order < MAX_ORDER:
        buddy_paddr = block.paddr XOR (PAGE_SIZE << order)
        buddy = find_block(buddy_paddr)
        
        if buddy != NULL and buddy.is_free and buddy.order == order:
            free_list[order].remove(buddy)
            block = coalesce(block, buddy)
            order = order + 1
        else:
            break
    
    free_list[order].push(block)
```

### Algoritmo 2: Slab Allocator (Objetos de Tamanho Fixo)

```pseudo
function slab_alloc(size):
    cache = find_cache_for_size(size)
    
    // 1. Tenta cache local por-CPU (hot path sem contenção de lock)
    cpu_cache = cache.per_cpu_caches[current_cpu()]
    if not cpu_cache.is_empty():
        return cpu_cache.pop()
    
    // 2. Tenta slabs parcialmente preenchidos
    acquire_lock(cache.lock)
    if not cache.partial_slabs.is_empty():
        slab = cache.partial_slabs.head()
        object = slab.allocate_slot()
        if slab.is_full():
            cache.partial_slabs.move_to(cache.full_slabs, slab)
        release_lock(cache.lock)
        return object
    
    // 3. Aloca novo slab a partir do Buddy Allocator
    new_slab_pages = buddy_alloc(cache.slab_order)
    if new_slab_pages == ERR_OUT_OF_MEMORY:
        release_lock(cache.lock)
        return ERR_OUT_OF_MEMORY
        
    slab = format_new_slab(new_slab_pages, cache.object_size)
    object = slab.allocate_slot()
    cache.partial_slabs.push(slab)
    release_lock(cache.lock)
    return object

function slab_free(ptr, size):
    cache = find_cache_for_size(size)
    slab = find_slab_for_pointer(ptr)
    
    acquire_lock(cache.lock)
    was_full = slab.is_full()
    slab.mark_slot_free(ptr)
    
    if was_full:
        cache.full_slabs.move_to(cache.partial_slabs, slab)
    else if slab.is_completely_empty():
        cache.partial_slabs.move_to(cache.empty_slabs, slab)
        // Se houver excesso de slabs vazios, devolve páginas ao Buddy
        if cache.empty_slabs.count() > cache.max_empty_threshold:
            buddy_free(slab.pages, cache.slab_order)
    release_lock(cache.lock)
```

### Algoritmo 3: Copy-on-Write (CoW) e Falta de Página

```pseudo
function fork_address_space(parent_task):
    child_space = create_address_space()
    
    for each mapping in parent_task.address_space.mappings:
        if mapping.writable:
            // Marca como Read-Only + CoW em ambos os espaços
            mapping.flags.writable = false
            mapping.flags.is_cow = true
            update_pte(parent_task, mapping.vaddr, mapping.flags)
            
            child_space.map(mapping.vaddr, mapping.paddr, mapping.flags)
            increment_page_refcount(mapping.paddr)
        else:
            // Página somente-leitura tradicional compartilhada diretamente
            child_space.map(mapping.vaddr, mapping.paddr, mapping.flags)
            increment_page_refcount(mapping.paddr)
            
    invalidate_tlb_all()
    return child_space

function handle_cow_fault(vaddr, task):
    pte = task.address_space.lookup_pte(vaddr)
    
    if not pte.is_cow:
        return ERR_SEGMENTATION_FAULT
    
    old_paddr = pte.physical_address
    
    // Se o contador de referência for 1, torna a página gravável diretamente
    if get_page_refcount(old_paddr) == 1:
        pte.writable = true
        pte.is_cow = false
        update_pte(task, vaddr, pte.flags)
        invalidate_tlb_entry(vaddr)
        return SUCCESS
    
    // Múltiplos donos: aloca nova página física e copia o conteúdo
    new_paddr = buddy_alloc(ORDER_4KB)
    if new_paddr == ERR_OUT_OF_MEMORY:
        return ERR_OUT_OF_MEMORY
        
    memcpy(direct_map(new_paddr), direct_map(old_paddr), PAGE_SIZE_4KB)
    decrement_page_refcount(old_paddr)
    
    pte.physical_address = new_paddr
    pte.writable = true
    pte.is_cow = false
    update_pte(task, vaddr, pte.flags)
    invalidate_tlb_entry(vaddr)
    return SUCCESS
```

---

## 13. Diagramas ASCII Avançados

### Estrutura do Buddy Allocator e Coalescência

```
                     Hierarquia de Ordens Buddy (Potências de 2)

MAX_ORDER (Order 18 = 1GB)  ┌────────────────────────────────────────────────────────┐
                            │                    [ Bloco de 1 GB ]                   │
                            └───────────────────────────┬────────────────────────────┘
                                                        │ Split / Coalesce
                                           ┌────────────┴────────────┐
                                           ▼                         ▼
Order 17 (512MB)                ┌─────────────────────┐   ┌─────────────────────┐
                                │      [ 512 MB ]     │   │      [ 512 MB ]     │
                                └──────────┬──────────┘   └─────────────────────┘
                                           │
                                         . . .
                                           │
Order 1 (8KB)                   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
                                │ B_1  │ │ B_2  │ │ B_3  │ │ B_4  │
                                └──────┘ └──────┘ └──────┘ └──────┘
                                   │        │
                                   └────┬───┘  Buddy Property:
                                        ▼      Buddy_Address = Address ⊕ (PAGE_SIZE << order)
Order 0 (4KB Páginas)           ┌────┐ ┌────┐
                                │ P0 │ │ P1 │
                                └────┘ └────┘
```

### Hierarquia de Paginação Virtual x86_64 de 4 Níveis (48-bit Canonical)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Endereço Virtual x86_64 (48 bits)                        │
│ ┌──────────────┬──────────────┬──────────────┬──────────────┬─────────────┐ │
│ │ PML4 (9b)    │ PDPT (9b)    │ PDT (9b)     │ PT (9b)      │ Offset(12b) │ │
│ │ Bits [47:39] │ Bits [38:30] │ Bits [29:21] │ Bits [20:12] │ Bits [11:0] │ │
│ └──────┬───────┴──────┬───────┴──────┬───────┴──────┬───────┴──────┬──────┘ │
└────────┼──────────────┼──────────────┼──────────────┼──────────────┼────────┘
         │              │              │              │              │
         ▼              ▼              ▼              ▼              ▼
    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │  PML4T  │───>│  PDPT   │───>│   PDT   │───>│   PT    │───>│ Página  │
    │ (512 e) │    │ (512 e) │    │ (512 e) │    │ (512 e) │    │ Física  │
    │  [CR3]  │    │         │    │         │    │         │    │ (4 KB)  │
    └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
```

### Topologia de Memória NUMA com Alocadores Locais

```
            NUMA Node 0 (Socket 0)                    NUMA Node 1 (Socket 1)
       ┌──────────────────────────────┐          ┌──────────────────────────────┐
       │   CPU0    CPU1    CPU2  CPU3 │          │   CPU4    CPU5    CPU6  CPU7 │
       └──────────────┬───────────────┘          └──────────────┬───────────────┘
                      │ (Acesso Rápido ~50ns)                   │ (Acesso Rápido ~50ns)
       ┌──────────────▼───────────────┐          ┌──────────────▼───────────────┐
       │     Buddy Allocator Node 0   │          │     Buddy Allocator Node 1   │
       │     Per-CPU Slab Caches      │          │     Per-CPU Slab Caches      │
       └──────────────┬───────────────┘          └──────────────┬───────────────┘
                      │                                         │
       ┌──────────────▼───────────────┐ Interconnect (UPI/QPI) ┌▼──────────────┐
       │     DRAM Local (Node 0)      │◄──────────────────────►│ DRAM Local (Node 1) │
       │     (Ex: 16 GB DDR4/5)       │  (Penalidade: ~150ns)  │ (Ex: 16 GB DDR4/5)  │
       └──────────────────────────────┘                        └─────────────────────┘
```

### Isolamento por Domínios e Mediação de Capabilities

```
  Domínio A (Driver de Rede)       Domínio B (Serviço de Arquivos)
┌────────────────────────────┐    ┌────────────────────────────┐
│ • Task 1 (vaddr: 0x1000)   │    │ • Task 2 (vaddr: 0x8000)   │
│ • Cap: {Read, Write}       │    │ • Cap: {Read}              │
└─────────────┬──────────────┘    └─────────────┬──────────────┘
              │                                 │
              └────────────────┬────────────────┘
                               ▼
            ┌──────────────────────────────────────┐
            │       Memory Core Capability Set     │
            │  Validação de permissão e tokens     │
            └──────────────────┬───────────────────┘
                               ▼
            ┌──────────────────────────────────────┐
            │   Página Física Compartilhada        │
            │   (Zero-Copy Shared DMA Buffer)      │
            └──────────────────────────────────────┘
```

---

## 14. Considerações para Implementação em Rust

### Abstração Segura sobre Memória Física e Pools

```rust
pub struct MemoryPool {
    allocator: Box<dyn MemoryAllocator>,
    metrics: Arc<Mutex<PoolMetrics>>,
}

impl MemoryPool {
    pub fn allocate(&mut self, size: usize, align: usize) -> Result<Box<[u8]>, AllocError> {
        let ptr = self.allocator.allocate(size, align)?;
        
        // SAFETY: O alocador garante que o ponteiro é válido, alinhado e exclusivo
        // para a quantidade de bytes solicitada durante a vida do Box.
        let slice = unsafe { core::slice::from_raw_parts_mut(ptr, size) };
        Ok(Box::from_raw(slice))
    }
}
```

### Operações Atômicas de Páginas e Contadores de Referência

```rust
use core::sync::atomic::{AtomicU8, AtomicU32, Ordering};

pub struct AtomicPageDescriptor {
    pub state: AtomicU8,
    pub ref_count: AtomicU32,
    pub physical_address: PhysicalAddress,
}

impl AtomicPageDescriptor {
    pub fn try_acquire_exclusive(&self) -> Result<(), AllocError> {
        self.state
            .compare_exchange(
                PageState::Free as u8,
                PageState::Used as u8,
                Ordering::Acquire,
                Ordering::Relaxed,
            )
            .map(|_| ())
            .map_err(|_| AllocError::AlreadyAllocated)
    }

    pub fn inc_ref(&self) -> u32 {
        self.ref_count.fetch_add(1, Ordering::AcqRel) + 1
    }

    pub fn dec_ref(&self) -> u32 {
        self.ref_count.fetch_sub(1, Ordering::AcqRel) - 1
    }
}
```

### Justificativa de `unsafe` e Mitigações

1. **Manipulação de Tabelas de Páginas (`PML4`, `PDPT`, `PDT`, `PT`):**
   - **Risco:** Corrupção de memória ou escape de privilégio via mapeamentos inválidos.
   - **Mitigação:** Tipos fortes `PhysicalAddress` e `VirtualAddress` com validação canônica de bits [47:0] e wrappers seguros para atualização de PTEs com invalidação de TLB obrigatória.
2. **Acesso Direto à Memória Física via HHDM (Higher-Half Direct Map):**
   - **Risco:** Aliasing e dataraces em ponteiros brutos.
   - **Mitigação:** Utilização da base HHDM fornecida pelo Limine encapsulada em referências Rust rastreadas por ownership ou spins sincronizados.

---

## 15. Roadmap de Implementação

### Fase 0 (v0.0.1 — Atual)
- [x] Bitmap / Bump Allocator inicial de bootstrap
- [x] Estruturas de memória básicas no kernel (`memory.rs`)
- [x] Integração básica de HHDM com Limine
- [x] Documentação arquitetural completa

### Fase 1 (v0.0.2 — Próximo Milestone)
- [ ] Buddy Allocator físico completo para gestão de páginas 4KB
- [ ] Slab Allocator para objetos pequenos (8B a 2048B)
- [ ] Leitura da tabela de mapa de memória (Memory Map) via protocolo Limine
- [ ] Mapeador virtual e criação de tabelas de página do kernel

### Fase 2 (v0.1.0)
- [ ] Suporte a Copy-on-Write (CoW) em clone de tarefas
- [ ] Sistema de Capabilities integrado a descritores de página
- [ ] Shared Memory Pages para zero-copy IPC
- [ ] Suporte inicial a topologia NUMA

### Fase 3 (v0.2.0)
- [ ] Integração com a Predictive Engine (aquecimento proativo e pools)
- [ ] Compactação e desfragmentação periódica de páginas
- [ ] Suporte a swapping modular via serviço de armazenamento

### Fase 4 (v1.0.0)
- [ ] Verificação formal de invariantes de alocação
- [ ] Otimização para centenas de núcleos e múltiplos nós NUMA
- [ ] Telemetria avançada de cache e memória em tempo real

---

## 16. Checklist de Revisão

- [x] Separação clara entre alocação física, mapeamento virtual e capacidades
- [x] Definição de metas de latência (<500ns no hot path do slab)
- [x] Modelagem de estruturas de dados essenciais (`Page`, `PhysicalRegion`, `NumaNode`, `PageCapabilitySet`)
- [x] Integração documentada com Scheduler, Predictive Engine e IPC
- [x] Pseudocódigos completos para Buddy, Slab e CoW
- [x] Diagramas de arquitetura, paginação e isolamento
- [x] Padrões seguros de Rust e justificativas explícitas para blocos `unsafe`
- [x] Roadmap e fases de entrega alinhados ao projeto Jinn OS

---

Arquivo: [Memory Core](./memory-core.md)

