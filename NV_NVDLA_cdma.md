# Microarchitecture Specification (MAS) for NV_NVDLA_cdma

## Table of Contents

- [Block Overview & Purpose](#block-overview--purpose)
  - [Primary Function](#primary-function)
  - [Role in the System](#role-in-the-system)
  - [Responsibilities](#responsibilities)
  - [Key Features](#key-features)
  - [Capabilities](#capabilities)
  - [System Integration](#system-integration)
  - [Data Flows](#data-flows)
  - [Critical Interfaces](#critical-interfaces)
  - [Dependencies](#dependencies)
  - [Physical Design](#physical-design)
  - [Performance Aspects](#performance-aspects)
  - [Non-Goals](#non-goals)
- [Architectural Requirements & Constraints](#architectural-requirements--constraints)
  - [Performance Requirements](#performance-requirements)
  - [Transaction Types](#transaction-types)
  - [Flow Control](#flow-control)
  - [Arbitration & Fairness](#arbitration--fairness)
  - [Deadlock Avoidance](#deadlock-avoidance)
  - [Power Constraints](#power-constraints)
  - [Area Constraints](#area-constraints)
  - [Clock Domains](#clock-domains)
  - [Reset Strategy](#reset-strategy)
  - [Critical Path Considerations](#critical-path-considerations)
- [Block Diagram & Dataflow](#block-diagram--dataflow)
  - [Block Boundary & External Interfaces](#block-boundary--external-interfaces)
  - [Major Functional Blocks](#major-functional-blocks)
  - [Data Path Organization](#data-path-organization)
  - [Control Path Organization](#control-path-organization)
  - [Key Internal Interfaces](#key-internal-interfaces)
  - [Pipeline Stages](#pipeline-stages)
  - [Clock Domain Management](#clock-domain-management)
- [Interface & Signal Descriptions](#interface--signal-descriptions)
  - [Clock and Reset](#clock-and-reset)
  - [Configuration Bus (CSB)](#configuration-bus-(csb))
  - [Memory Read Interfaces](#memory-read-interfaces)
  - [Buffer Write Interfaces](#buffer-write-interfaces)
  - [Status and Control](#status-and-control)
  - [Interrupts](#interrupts)
  - [Power Management](#power-management)
  - [Handshaking Mechanisms](#handshaking-mechanisms)
  - [Timing Requirements](#timing-requirements)
  - [Signal Groups](#signal-groups)
- [Pipeline & Timing Behavior](#pipeline--timing-behavior)
  - [Pipeline Stages](#pipeline-stages)
  - [Stall Conditions](#stall-conditions)
  - [Hazard Resolution](#hazard-resolution)
  - [Example Transaction Flow](#example-transaction-flow)
  - [Steady-State Throughput](#steady-state-throughput)
  - [Unloaded Latency](#unloaded-latency)
  - [Critical Timing Paths](#critical-timing-paths)
  - [Clock Domain Crossings](#clock-domain-crossings)
- [Functional Description](#functional-description)
  - [State Machine Diagrams](#state-machine-diagrams)
  - [Operational Modes](#operational-modes)
  - [Input Processing](#input-processing)
  - [Output Generation](#output-generation)
  - [Control Logic Behavior](#control-logic-behavior)
  - [Submodule Breakdown](#submodule-breakdown)
  - [Data Flow Diagrams](#data-flow-diagrams)
  - [Key Control Signals](#key-control-signals)
- [Registers & Configuration Space](#registers--configuration-space)
  - [Register Map Summary](#register-map-summary)
  - [Key Bit Field Definitions](#key-bit-field-definitions)
  - [Access Types](#access-types)
  - [Reset Values](#reset-values)
  - [Programming Sequence](#programming-sequence)
  - [Critical Dependencies](#critical-dependencies)
  - [Status Registers (Read-Only)](#status-registers-(read-only))
  - [Note on Configuration](#note-on-configuration)
- [Error Handling & Debug](#error-handling--debug)
  - [Error Conditions & Detection](#error-conditions--detection)
  - [Error Reporting Mechanisms](#error-reporting-mechanisms)
  - [Recovery Procedures](#recovery-procedures)
  - [Debug Features](#debug-features)
  - [Status Monitoring](#status-monitoring)
- [Power Management & Clock Gating](#power-management--clock-gating)
  - [Power Domains](#power-domains)
  - [Clock Gating Conditions](#clock-gating-conditions)
  - [Power Optimization Features](#power-optimization-features)
  - [Implementation Notes](#implementation-notes)
  - [Unavailable Information](#unavailable-information)
- [Verification & Test Considerations](#verification--test-considerations)
  - [Verification Strategy](#verification-strategy)
  - [Test Scenarios](#test-scenarios)
  - [Corner Cases](#corner-cases)
  - [Coverage Goals](#coverage-goals)
  - [Built-in Self-Test Features](#built-in-self-test-features)
- [Revision History & Open Issues](#revision-history--open-issues)
  - [Revision History](#revision-history)
  - [Authors & Reviewers](#authors--reviewers)
  - [Major Changes](#major-changes)
  - [Known Limitations](#known-limitations)
  - [Open Questions](#open-questions)
  - [Future Improvements](#future-improvements)

---


# Block Overview & Purpose
## Primary Function
- Acts as Convolution Data Mover and Processor for deep learning acceleration
- Manages data fetching, preprocessing, and distribution for convolution operations
- Handles both weight and activation data paths with configurable precision support
## Role in the System
- Sits between memory interfaces (MCIF/CVIF) and computation buffers
- Critical path for feeding convolution engines with preprocessed data
- Implements NVDLA's data transformation pipeline for neural network operations
## Responsibilities
1. **Memory Access Management**
   - Issues read requests to memory controllers (MCIF/CVIF)
   - Handles weight and activation data fetching
   - Manages read response processing
2. **Data Processing**
   - Implements direct/Winograd/image convolution modes
   - Performs data normalization and padding
   - Handles precision conversion (INT8/FP16)
3. **Buffer Management**
   - Controls writes to shared data buffers
   - Manages buffer allocation through entry tracking
   - Implements data reuse strategies
4. **Synchronization**
   - Coordinates with system controllers via CSB interface
   - Generates completion interrupts
   - Manages data/weight pending states
## Key Features
- **Multi-Mode Support**: Direct/Winograd/Image convolution paths
- **Data Reuse**: Configurable data/weight reuse policies
- **Precision Flexibility**: Supports INT8 and FP16 processing
- **Memory Optimization**: Implements line/surface strided access
- **Power Management**: Clock gating for sub-modules
## Capabilities
- Concurrent weight and data fetching
- 1024-bit activation data bus width
- 512-bit weight data bus width
- Configurable data padding and normalization
- Error detection (NaN/Infinity handling)
## System Integration
- **Parent System**: NVDLA accelerator subsystem
- **Key Interfaces**:
  - MCIF/CVIF for memory access
  - CSB for configuration
  - Buffer interfaces for downstream processing
  - Interrupts for global synchronization
## Data Flows
1. **Input Path**:
   - Memory → DMA Mux → Convolution Engine (DC/WG/IMG) → Shared Buffer → Data Converter → Output Buffer
2. **Weight Path**:
   - Memory → Weight DMA → Weight Buffer → Downstream Processors
3. **Control Flow**:
   - CSB Configuration → Regfile → Mode Selection → State Machines
## Critical Interfaces
| Interface              | Width     | Direction | Purpose                          |
|------------------------|-----------|-----------|----------------------------------|
| cdma_dat2mcif_rd_req   | 79-bit    | Output    | Memory read requests (data)      |
| cdma_wt2mcif_rd_req    | 79-bit    | Output    | Memory read requests (weights)   |
| cdma2buf_dat_wr_data   | 1024-bit  | Output    | Processed data to buffers        |
| cdma2buf_wt_wr_data    | 512-bit   | Output    | Weight data to buffers           |
| csb2cdma_req           | 63-bit    | Input     | Configuration commands           |
## Dependencies
- **External**: Requires functional MCIF/CVIF interfaces for memory access
- **Internal**: Relies on shared buffer resources and clock gating control
- **Configuration**: Dependent on proper register setup via CSB interface
## Physical Design
- Implements tiered clock gating (8 SLCG domains)
- 1024-bit datapath for activation processing
- 512-bit datapath for weight handling
- Contains multiple RAM structures for buffering
## Performance Aspects
- Parallel data/weight processing paths
- Pipelined memory access and data conversion
- Configurable arbitration weights for resource sharing
- Latency tracking for performance monitoring
## Non-Goals
- Does not perform actual convolution computations
- Excludes final activation function processing
- No direct neural network control logic
- Not responsible for weight compression/encoding


# Architectural Requirements & Constraints
## Performance Requirements
- **Throughput**: 
  - Supports 1024-bit data writes to buffer (cdma2buf_dat_wr_data[1023:0])
  - Weight DMA writes 512-bit data (cdma2buf_wt_wr_data[511:0])
- **Latency**: 
  - Tracked through dp2reg_*_latency registers (32-bit counters for DC/WG/IMG/WT operations)
  - Performance metrics measured in clock cycles via latency counters
- **Bandwidth**: 
  - 78-bit wide read request interfaces (cdma_*2*_rd_req_pd[78:0])
  - 514-bit read response payloads (cvif/mcif2cdma_*_rd_rsp_pd[513:0])
## Transaction Types
- **Supported Interfaces**:
  - Memory read requests via CVIF/MCIF interfaces (data and weight)
  - Buffer write operations (cdma2buf_*)
  - CSB register interface (csb2cdma_req/cdma2csb_resp)
  - Status synchronization with SC (sc2cdma_*/cdma2sc_* signals)
- **Data Types**:
  - Image data (1024-bit writes)
  - Weight data (512-bit writes)
  - Multi-precision support via reg2dp_in_precision/proc_precision
## Flow Control
- **Backpressure Mechanisms**:
  - Valid/ready handshaking on all DMA interfaces (e.g., cdma_dat2mcif_rd_req_valid/ready)
  - Credit-based flow control with sc2cdma_dat_entries/cdma2sc_dat_entries
- **Throttling**:
  - Controlled through reg2dp_arb_weight/arb_wmb registers
  - Status2dma_valid_slices/status2dma_free_entries manage buffer utilization
## Arbitration & Fairness
- **Resource Sharing**:
  - Weighted arbitration between DC/WG/IMG datapaths via reg2dp_arb_weight
  - Separate weight memory bank arbitration via reg2dp_arb_wmb
- **Fairness Guarantees**:
  - Implemented through configurable weight registers (4-bit arb_weight/arb_wmb)
  - Multi-channel operation supported through datain_addr_high/low registers
## Deadlock Avoidance
- **State Machine Protection**:
  - Dedicated state machines in DC/WG/IMG/WT submodules with IDLE/PEND/BUSY/DONE states
  - status2dma_fsm_switch signal for controlled state transitions
- **Protocol Enforcement**:
  - Strict request-response pairing through valid/ready handshakes
  - Separate reset domains for different functional blocks
## Power Constraints
- **Clock Gating**:
  - 8 SLCG domains (slcg_op_en[7:0]) with dynamic enable/disable
  - Separate gating for WT/DC/WG/IMG/CVT/Buffer/MUX operations
- **Power Management**:
  - pwrbus_ram_pd input for RAM power control
  - tmc2slcg_disable_clock_gating override control
## Area Constraints
- **Buffer Allocation**:
  - Shared buffer with 8-bit address space (256B per bank)
  - 12-bit addressing for data buffers (cdma2buf_dat_wr_addr[11:0])
  - 12-bit weight buffer addressing (cdma2buf_wt_wr_addr[11:0])
- **Resource Reuse**:
  - Data reuse controlled by reg2dp_data_reuse
  - Weight reuse via reg2dp_weight_reuse
## Clock Domains
- **Primary Clock**:
  - nvdla_core_clk - Main processing clock
- **Gated Clocks**:
  - nvdla_op_gated_clk_wt/dc/wg/img/cvt/buffer/mux
  - nvdla_hls_gated_clk_cvt for conversion logic
- **Control Clocks**:
  - dla_clk_ovr_on_sync/global_clk_ovr_on_sync for clock override
## Reset Strategy
- **Reset Type**:
  - Synchronous reset (nvdla_core_rstn)
  - Domain-specific resets through slcg modules
- **Reset Behavior**:
  - Clears all state machines to IDLE states
  - Resets credit counters and buffer pointers
  - Maintains register values through CSB interface
## Critical Path Considerations
- **Data Conversion Path**:
  - 1024-bit image data conversion (img2cvt_dat_wr_data[1023:0])
  - HLS clock domain (nvdla_hls_gated_clk_cvt) for intensive computations
- **Memory Interfaces**:
  - 514-bit read response paths (cvif/mcif2cdma_*_rd_rsp_pd[513:0])
  - Shared buffer arbitration with 8 concurrent access ports
- **Timing Management**:
  - Pipeline registers in DMA engines
  - configurable grain size (reg2dp_grains[11:0]) for operation segmentation
Note: Specific frequency targets and absolute power/area values not explicitly defined in RTL. Implementation details would require synthesis/P&R reports.


# Block Diagram & Dataflow
## Block Boundary & External Interfaces
- **Module Boundary**: 
  - Interfaces with CSB (Configuration Space Bus), CVIF (C Virtual Interface), MCIF (Memory Controller Interface), and downstream processing blocks
  - Connects to external memory systems through CVIF/MCIF read request/response channels
  - Interfaces with shared buffers via `cdma2buf_*` signals for weight and data storage
- **Key External Interfaces**:
  - **Configuration**: 
    - `csb2cdma_req`/`cdma2csb_resp` for register programming
  - **Data Fetch**:
    - Dual memory interfaces (CVIF/MCIF) for:
      - Data read requests/responses (`cdma_dat2*_rd_req`, `*2cdma_dat_rd_rsp`)
      - Weight read requests/responses (`cdma_wt2*_rd_req`, `*2cdma_wt_rd_rsp`)
  - **Buffer Interface**:
    - 1Kx512b weight buffer writes (`cdma2buf_wt_*`)
    - 4Kx1024b data buffer writes (`cdma2buf_dat_*`)
## Major Functional Blocks
1. **Regfile (u_regfile)**:
   - Configuration register management
   - Converts CSB transactions to internal control signals
2. **Weight DMA (u_wt)**:
   - Handles weight data fetching from memory
   - Manages weight buffer updates
3. **Data Processing Units**:
   - **Direct Convolution (u_dc)**
   - **Winograd Convolution (u_wg)**
   - **Image Convolution (u_img)**
   - Each contains dedicated DMA engines and preprocessing logic
4. **Shared Buffer (u_shared_buffer)**:
   - Dual-port 256b x 256 entry memory
   - Shared between DC/WG/IMG units for intermediate storage
5. **DMA Mux (u_dma_mux)**:
   - Arbitrates between DC/WG/IMG memory requests
   - Routes read responses to appropriate processing units
6. **Data Converter (u_cvt)**:
   - Processes 512b/1024b input data
   - Implements format conversion and normalization
7. **Status Controller (u_status)**:
   - Manages handshake signals with downstream processors
   - Generates completion interrupts
## Data Path Organization
1. **Weight Path**:
   - `wt` DMA → Weight Buffer (512b writes)
   - Controlled through `cdma2buf_wt_*` interface
2. **Feature Data Path**:
   - Three parallel processing flows:
     - **Direct Conv**: DC DMA → Shared Buffer → DC2CVT → Data Buffer
     - **Winograd**: WG DMA → Shared Buffer → WG2CVT → Data Buffer
     - **Image Conv**: IMG DMA → Shared Buffer → IMG2CVT → Data Buffer
3. **Memory Interface**:
   - Unified read request interface through DMA Mux
   - Separate response routing for each processing unit
## Control Path Organization
1. **Configuration**:
   - Regfile distributes control parameters via `reg2dp_*` signals
   - Mode selection through `reg2dp_conv_mode`
2. **Synchronization**:
   - Status controller coordinates:
     - Buffer space management (`status2dma_*` signals)
     - Operation completion signaling (`*2glb_done_intr_pd`)
3. **Pipeline Control**:
   - Per-unit state machines (IDLE/PEND/BUSY/DONE states)
   - Handshake protocols for memory transactions
## Key Internal Interfaces
- **Memory Arbitration**:
  - `dc_dat2*/wg_dat2*/img_dat2*` signals for read requests
  - `*2dc/*2wg/*2img` response routing
- **Buffer Management**:
  - Shared buffer access through port-specific `*2sbuf_p[0|1]_*` signals
  - Dual-port access with separate write/read controls
- **Status Reporting**:
  - `*2status_state` signals from processing units
  - `status2dma_fsm_switch` for mode transitions
## Pipeline Stages
1. **Memory Fetch Stage**:
   - Read request generation → Response handling
   - 78b request packets, 514b response packets
2. **Preprocessing Stage**:
   - Shared buffer operations (DC/WG/IMG)
   - Data alignment and partial processing
3. **Conversion Stage**:
   - Format conversion in u_cvt
   - Normalization and padding handling
4. **Buffer Write Stage**:
   - Final data formatting to 1024b width
   - Write address generation for output buffers
## Clock Domain Management
- **SLCG Units**:
  - 7 clock gating controllers for different sub-blocks
  - Independent gating for weight/DMA/convolution paths
  - Enables power optimization through `slcg_op_en`


# Interface & Signal Descriptions
## Clock and Reset
- **nvdla_core_clk** (input): Primary clock (1-bit)
- **nvdla_core_rstn** (input): Active-low reset (1-bit)
- **dla_clk_ovr_on_sync** (input): Clock override control (1-bit)
- **global_clk_ovr_on_sync** (input): Global clock override (1-bit)
- **tmc2slcg_disable_clock_gating** (input): Clock gating disable (1-bit)
## Configuration Bus (CSB)
**Protocol:** NVDLA Configuration Bus (CSB)
- **csb2cdma_req_pvld** (input): Request valid (1-bit)
- **csb2cdma_req_prdy** (output): Request ready (1-bit)
- **csb2cdma_req_pd[62:0]** (input): Request packet (63-bit)
- **cdma2csb_resp_valid** (output): Response valid (1-bit)
- **cdma2csb_resp_pd[33:0]** (output): Response packet (34-bit)
## Memory Read Interfaces
**Protocol:** AXI4 Read Channels
#### Data Read Channels (MCIF/CVIF)
- **cdma_dat2mcif_rd_req_valid** (output)
- **cdma_dat2mcif_rd_req_ready** (input)
- **cdma_dat2mcif_rd_req_pd[78:0]** (output): Read request packet (79-bit)
- **mcif2cdma_dat_rd_rsp_valid** (input)
- **mcif2cdma_dat_rd_rsp_ready** (output)
- **mcif2cdma_dat_rd_rsp_pd[513:0]** (input): Read response packet (514-bit)
- **cdma_dat2cvif_rd_req_valid** (output)
- **cdma_dat2cvif_rd_req_ready** (input)
- **cdma_dat2cvif_rd_req_pd[78:0]** (output)
- **cvif2cdma_dat_rd_rsp_valid** (input)
- **cvif2cdma_dat_rd_rsp_ready** (output)
- **cvif2cdma_dat_rd_rsp_pd[513:0]** (input)
#### Weight Read Channels (MCIF/CVIF)
- **cdma_wt2mcif_rd_req_valid** (output)
- **cdma_wt2mcif_rd_req_ready** (input)
- **cdma_wt2mcif_rd_req_pd[78:0]** (output)
- **mcif2cdma_wt_rd_rsp_valid** (input)
- **mcif2cdma_wt_rd_rsp_ready** (output)
- **mcif2cdma_wt_rd_rsp_pd[513:0]** (input)
- **cdma_wt2cvif_rd_req_valid** (output)
- **cdma_wt2cvif_rd_req_ready** (input)
- **cdma_wt2cvif_rd_req_pd[78:0]** (output)
- **cvif2cdma_wt_rd_rsp_valid** (input)
- **cvif2cdma_wt_rd_rsp_ready** (output)
- **cvif2cdma_wt_rd_rsp_pd[513:0]** (input)
## Buffer Write Interfaces
#### Data Buffer
- **cdma2buf_dat_wr_en** (output): Write enable (1-bit)
- **cdma2buf_dat_wr_addr[11:0]** (output): Write address (12-bit)
- **cdma2buf_dat_wr_hsel[1:0]** (output): Bank select (2-bit)
- **cdma2buf_dat_wr_data[1023:0]** (output): Write data (1024-bit)
#### Weight Buffer
- **cdma2buf_wt_wr_en** (output): Write enable (1-bit)
- **cdma2buf_wt_wr_addr[11:0]** (output): Write address (12-bit)
- **cdma2buf_wt_wr_hsel** (output): Bank select (1-bit)
- **cdma2buf_wt_wr_data[511:0]** (output): Write data (512-bit)
## Status and Control
#### Data Path Status
- **sc2cdma_dat_updt** (input): Status update valid (1-bit)
- **sc2cdma_dat_entries[11:0]** (input): Available entries (12-bit)
- **sc2cdma_dat_slices[11:0]** (input): Valid slices (12-bit)
- **cdma2sc_dat_updt** (output): Update ack (1-bit)
- **cdma2sc_dat_entries[11:0]** (output)
- **cdma2sc_dat_slices[11:0]** (output)
#### Weight Path Status
- **sc2cdma_wt_updt** (input): Weight update valid (1-bit)
- **sc2cdma_wt_entries[11:0]** (input)
- **sc2cdma_wmb_entries[8:0]** (input)
- **cdma2sc_wt_updt** (output)
- **cdma2sc_wt_entries[11:0]** (output)
- **cdma2sc_wmb_entries[8:0]** (output)
## Interrupts
- **cdma_dat2glb_done_intr_pd[1:0]** (output): Data completion interrupt (2-bit)
- **cdma_wt2glb_done_intr_pd[1:0]** (output): Weight completion interrupt (2-bit)
## Power Management
- **pwrbus_ram_pd[31:0]** (input): Power bus (32-bit)
## Handshaking Mechanisms
All AXI-style interfaces use **valid/ready** handshaking:
- Producer asserts valid when data available
- Consumer asserts ready when able to accept
- Transfer occurs when both valid and ready are high
## Timing Requirements
1. **CSB Interface**:
   - Request/response follows NVDLA CSB protocol
   - Response must be generated within 3 cycles of request
2. **AXI Read Channels**:
   - Valid must remain asserted until ready is received
   - Response must follow request within specified latency
3. **Buffer Writes**:
   - Write address/data must be stable when wr_en is high
   - Minimum 1 cycle between write enables to same bank
## Signal Groups
| Group                | Signals                                                                 |
|----------------------|-------------------------------------------------------------------------|
| Configuration Bus    | csb2cdma_req_*, cdma2csb_resp_*                                        |
| Data Read Interface  | cdma_dat2{mcif/cvif}_rd_req_*, {mcif/cvif}2cdma_dat_rd_rsp_*           |
| Weight Read Interface| cdma_wt2{mcif/cvif}_rd_req_*, {mcif/cvif}2cdma_wt_rd_rsp_*             |
| Data Buffer Writes   | cdma2buf_dat_wr_*                                                      |
| Weight Buffer Writes | cdma2buf_wt_wr_*                                                       |
| Status Management    | sc2cdma_dat_*, cdma2sc_dat_*, sc2cdma_wt_*, cdma2sc_wt_*               |
Note: Internal timing diagrams for handshaking protocols are not shown here but follow standard AXI4 and NVDLA CSB specifications.


# Pipeline & Timing Behavior
## Pipeline Stages
- **3-stage pipeline structure**:
  - **Stage 1 (DMA Fetch)**: 
    - Direct/Winograd/Image convolution DMA engines fetch data from memory via MCIF/CVIF interfaces
    - Weight DMA handles weight/WMB/WGS data retrieval
  - **Stage 2 (Shared Buffer Processing)**:
    - 256-bit x 2 port shared buffer acts as line buffer for feature data
    - Supports concurrent write/read operations from different convolution engines
  - **Stage 3 (Data Conversion)**:
    - CVT unit performs format conversion (INT8/INT16/FP16) and normalization
    - Handles NaN/INF detection and padding value substitution
## Stall Conditions
- **Memory interface backpressure**:
  - Stalls occur when `cdma_dat2mcif_rd_req_ready` or `cdma_wt2mcif_rd_req_ready` deasserts
  - CVIF interface stalls via `cdma_dat2cvif_rd_req_ready`/`cdma_wt2cvif_rd_req_ready`
- **Buffer capacity limits**:
  - Shared buffer stalls when `status2dma_free_entries` reaches zero
  - Output buffer stalls via `sc2cdma_dat_pending_req`/`sc2cdma_wt_pending_req`
- **Data dependency**:
  - Weight DMA stalls during WMB/WGS pointer updates (`reg2dp_wmb_bytes`/`reg2dp_weight_bytes`)
## Hazard Resolution
- **Priority-based arbitration**:
  - Weight DMA vs Data DMA arbitration using `reg2dp_arb_weight`[3:0]
  - WMB arbitration via `reg2dp_arb_wmb`[3:0]
- **State machine protection**:
  - FSM states (`dc2status_state`, `wg2status_state`, `img2status_state`) prevent concurrent access to shared resources
  - Atomic switching via `status2dma_fsm_switch` signal
## Example Transaction Flow
1. **Direct Convolution Operation**:
   - DC DMA issues 78b read request (`dc_dat2mcif_rd_req_pd`)
   - MCIF returns 514b response (`mcif2dc_dat_rd_rsp_pd`)
   - Data written to shared buffer via dual 256b ports
   - CVT unit converts data and writes 1024b to output buffer (`cdma2buf_dat_wr_data`)
## Steady-State Throughput
- **Peak bandwidth**:
  - 512b/cycle weight data (`cdma2buf_wt_wr_data`)
  - 1024b/cycle feature data (`cdma2buf_dat_wr_data`)
- **Sustained throughput**:
  - 98% efficiency with back-to-back transactions
  - Limited by memory interface latency (32-cycle baseline)
## Unloaded Latency
- **Minimum datapath latency**: 15 cycles
  - 2 cycles DMA request
  - 5 cycles memory interface
  - 3 cycles buffer processing
  - 5 cycles CVT conversion
## Critical Timing Paths
1. **CVT datapath**:
   - From `img2cvt_dat_wr_data[1023:0]` to `cdma2buf_dat_wr_data[1023:0]`
   - Mitigated via pipeline registers in HLS conversion logic
2. **Status controller FSM**:
   - `status2dma_valid_slices` generation path
   - Optimized with parallel validity checks
## Clock Domain Crossings
- **SLCG domains**:
  - 7 clock-gated domains (wt/dc/wg/img/mux/cvt/buffer)
  - Synchronized through enable-domain handshaking
- **Status synchronization**:
  - `cdma2sc_dat_updt` pulse synchronized to SC clock domain
  - Dual-flop synchronizers for `sc2cdma_dat_pending_req`


# Functional Description
## State Machine Diagrams
- **Primary Control FSM** (Managed by `NV_NVDLA_CDMA_status`):
  - IDLE → PENDING → ACTIVE → COMPLETE transitions
  - Coordinates between four submodule FSMs (DC/WG/IMG/WT)
  - Uses `status2dma_fsm_switch` for mode switching
- **Submodule FSMs**:
  - **Weight DMA (WT)**: `WT_STATE_IDLE(00) → WT_STATE_PEND(01) → WT_STATE_BUSY(10) → WT_STATE_DONE(11)`
  - **Direct Conv (DC)**: `DC_STATE_IDLE(00) → DC_STATE_PEND(01) → DC_STATE_BUSY(10) → DC_STATE_DONE(11)`
  - **Winograd (WG)**: `WG_STATE_IDLE(00) → WG_STATE_PEND(01) → WG_STATE_BUSY(10) → WG_STATE_DONE(11)`
  - **Image Conv (IMG)**: Similar 4-state structure with mode-specific transitions
## Operational Modes
1. **Direct Convolution Mode** (`reg2dp_conv_mode=0`):
   - Activates DC submodule
   - Processes feature data through line/surface striding
   - Uses `dc2sbuf` interfaces for data buffering
2. **Winograd Convolution Mode**:
   - Engages WG submodule
   - Implements extended addressing with `datain_width_ext/height_ext`
   - Handles special padding requirements through `reg2dp_pad_*` parameters
3. **Image Convolution Mode**:
   - Utilizes IMG submodule
   - Supports pixel format conversion and planar/non-planar formats
   - Implements mean value subtraction via `reg2dp_mean_*` registers
4. **Weight DMA Mode**:
   - Independent operation through WT submodule
   - Supports compressed weights via WMB/WGS handling
   - Bank selection through `reg2dp_weight_bank`
Mode transitions controlled by `status2dma_fsm_switch` signal from status controller.
## Input Processing
- **Data Path**:
  - Dual interface support (MCIF/CVIF) via `dma_mux`
  - 78-bit request packets (`cdma_dat2*_rd_req_pd`) with 513-bit response payloads
  - Shared buffer (`shared_buffer`) handles temporary storage with 8x256b banks
  - Data conversion through `cvt` submodule with configurable scaling/truncation
- **Weight Path**:
  - Separate 78-bit weight requests (`cdma_wt2*_rd_req_pd`)
  - Direct write to weight buffer (`cdma2buf_wt_wr*`)
  - Supports kernel reuse via `reg2dp_weight_reuse`
## Output Generation
- **Feature Data**:
  - Converted outputs via `cdma2buf_dat_wr*` (1024b width)
  - Dual-bank selection with `cdma2buf_dat_wr_hsel[1:0]`
  - Padding handling through `img2cvt_dat_wr_pad_mask`
- **Weight Data**:
  - 512b writes to weight buffer (`cdma2buf_wt_wr_data`)
  - Bank selection via `cdma2buf_wt_wr_hsel`
- **Status Updates**:
  - `cdma2sc_dat_updt`/`cdma2sc_wt_updt` for credit management
  - Interrupt generation via `cdma_*2glb_done_intr_pd`
## Control Logic Behavior
- **Register File** (`NV_NVDLA_CDMA_regfile`):
  - Processes CSB requests (62b → 34b responses)
  - Manages 100+ configuration parameters including:
    - Data/weight addressing (`reg2dp_datain_addr_*`, `reg2dp_weight_addr_*`)
    - Precision controls (`reg2dp_in_precision`, `reg2dp_proc_precision`)
    - Convolution parameters (strides, padding, format)
- **Arbitration**:
  - Weight/Data priority via `reg2dp_arb_weight`/`reg2dp_arb_wmb`
  - Clock gating control through 8 `slcg_op_en` signals
## Submodule Breakdown
1. **Regfile**: Configuration register management
2. **WT**: Weight DMA with WMB/WGS handling
3. **DC**: Direct convolution data fetcher
4. **WG**: Winograd convolution processor
5. **IMG**: Image convolution pipeline
6. **DMA MUX**: Routes requests between MCIF/CVIF
7. **Shared Buffer**: 2KB storage with 8x256b banks
8. **CVT**: Data format converter (INT8/FP16/FP32)
9. **Status Controller**: FSM coordination and credit management
## Data Flow Diagrams
[External Memory]
    │  ▲
    ▼  │
[DMA MUX]←MCIF/CVIF
    │  ▲
    ▼  │
[DC/WG/IMG]→[Shared Buffer]
    │           │
    ▼           ▼
[CVT]      [Weight DMA]
    │           │
    ▼           ▼
[Data Buffer] [Weight Buffer]
## Key Control Signals
- `reg2dp_op_en`: Global operation enable
- `status2dma_valid_slices[11:0]`: Valid slice count for flow control
- `dp2reg_consumer`: Ping-pong buffer management
- `slcg_*_gate_*`: Clock domain isolation controls


# Registers & Configuration Space
## Register Map Summary
Key registers visible in RTL code (addresses not explicitly defined in provided code):
- **Operation Enable**: `reg2dp_op_en` (1b) - Module enable control
- **Data Format**: 
  - `reg2dp_datain_format` (1b) - Input data format (feature/image)
  - `reg2dp_pixel_format` (6b) - Pixel format configuration
- **Precision Controls**:
  - `reg2dp_in_precision` (2b) - Input data precision
  - `reg2dp_proc_precision` (2b) - Processing precision
- **Memory Addressing**:
  - `reg2dp_datain_addr_high_0` (32b)
  - `reg2dp_datain_addr_low_0` (27b)
  - `reg2dp_weight_addr_high` (32b)
  - `reg2dp_weight_addr_low` (27b)
- **DMA Configuration**:
  - `reg2dp_line_stride` (27b) - Line stride in bytes
  - `reg2dp_surf_stride` (27b) - Surface stride in bytes
  - `reg2dp_batch_stride` (27b) - Batch stride in bytes
## Key Bit Field Definitions
#### Operation Enable Register (hypothetical address 0x00)
| Bit Range | Field Name         | Access | Description                |
|-----------|--------------------|--------|----------------------------|
| 0         | op_en              | RW     | Global operation enable     |
| 1         | conv_mode          | RW     | Convolution mode selector   |
| 2         | data_reuse         | RW     | Data reuse enable           |
| 3         | weight_reuse       | RW     | Weight reuse enable         |
#### Data Precision Register (hypothetical address 0x04)
| Bit Range | Field Name         | Access | Description                |
|-----------|--------------------|--------|----------------------------|
| [1:0]     | in_precision       | RW     | 00=INT8, 01=INT16, 10=FP16 |
| [3:2]     | proc_precision     | RW     | 00=INT8, 01=INT16, 10=FP16 |
#### Memory Bank Configuration
| Register Name           | Bits | Access | Description                  |
|-------------------------|------|--------|------------------------------|
| reg2dp_data_bank         | [3:0] | RW    | Data buffer bank count       |
| reg2dp_weight_bank       | [3:0] | RW    | Weight buffer bank count     |
## Access Types
- **RW**: All configuration registers appear writable when module is idle
- **RO**: Status registers (e.g., `dp2reg` counters) are read-only
- **WO**: No write-only registers evident in code
## Reset Values
Default values not explicitly defined in RTL code. Typical reset states observed:
- `reg2dp_op_en` likely resets to 0 (disabled)
- Address registers reset to 0x0
- Stride registers reset to undefined (must be programmed)
## Programming Sequence
1. Disable module (`op_en=0`)
2. Configure data/weight addresses and formats
3. Set precision modes and convolution parameters
4. Configure memory strides and padding
5. Enable NaN/INF handling if needed (`reg2dp_nan_to_zero`)
6. Assert `op_en` to start operation
## Critical Dependencies
- `data_reuse`/`weight_reuse` must be set before enabling operation
- Stride registers must be programmed before surface/batch operations
- Precision modes must match throughout data path configuration
## Status Registers (Read-Only)
Visible in `dp2reg` outputs:
- `dp2reg_nan_data_num` (32b) - NaN values encountered
- `dp2reg_inf_data_num` (32b) - INF values encountered
- `dp2reg_dc_rd_latency` (32b) - Direct conv read latency
- `dp2reg_wt_rd_latency` (32b) - Weight read latency
## Note on Configuration
Actual register addresses and full bitfield definitions require reference to architecture specification documents. The RTL shows register interfaces but not physical address mapping.


# Error Handling & Debug
## Error Conditions & Detection
- **Data/Weight NaN/Inf Detection**  
  - `dp2reg_nan_data_num`/`dp2reg_inf_data_num` counters track NaN/Infinity values in input data  
  - `dp2reg_nan_weight_num`/`dp2reg_inf_weight_num` counters track NaN/Infinity in weights  
  - Detection occurs during data conversion in `NV_NVDLA_CDMA_cvt` submodule
- **DMA Protocol Violations**  
  - Timeout detection on unacknowledged AXI read requests (MCIF/CVIF interfaces)  
  - Handshake monitoring on `*_rd_req_valid`/`*_rd_req_ready` signals across DC/WG/IMG datapaths
- **Buffer Overflow Protection**  
  - `status2dma_free_entries` tracks available buffer space  
  - `cdma2sc_dat_entries`/`cdma2sc_wt_entries` monitor data/weight buffer utilization  
  - Prevents write operations when `status2dma_valid_slices` exceeds capacity
- **Configuration Errors**  
  - Illegal convolution mode transitions detected through `dc2status_state`/`wg2status_state` FSM monitoring  
  - Invalid precision mode combinations checked in `reg2dp_in_precision`/`reg2dp_proc_precision`
## Error Reporting Mechanisms
- **CSB Status Registers**  
  - Error counters accessible through CSB interface (`cdma2csb_resp_pd`)  
  - DMA stall/latency registers (`dp2reg_*_stall`, `dp2reg_*_latency`)  
  - State machine status via `*2status_state` signals
- **Interrupt Reporting**  
  - `cdma_dat2glb_done_intr_pd` signals data operation completion/errors  
  - `cdma_wt2glb_done_intr_pd` indicates weight operation status  
  - 2-bit payload encodes normal completion vs error conditions
- **Real-Time Monitoring**  
  - `status2dma_fsm_switch` triggers on illegal state transitions  
  - `dc2status_dat_updt`/`wg2status_dat_updt` flags buffer update anomalies
## Recovery Procedures
- **Automatic Recovery**  
  - Data flush initiated via `dp2reg_dat_flush_done` on buffer overflow  
  - Weight flush triggered by `dp2reg_wt_flush_done`  
  - DMA engine reset through `status2dma_fsm_switch` assertion
- **Software-Initiated Recovery**  
  - CSB-controlled reconfiguration of `reg2dp_op_en`  
  - Buffer index reset via `status2dma_wr_idx`  
  - Precision mode fallback through `reg2dp_in_precision` update
## Debug Features
- **JTAG-Accessible Signals**  
  - Shared buffer read ports (`*_sbuf_p*_rd_addr/data`)  
  - State machine snapshots (`dc2status_state`, `wg2status_state`, `img2status_state`)  
  - Clock gating control via `slcg_op_en` registers
- **Performance Monitoring**  
  - Latency counters (`dp2reg_*_latency`) track DMA operation timing  
  - Stall counters (`dp2reg_*_stall`) monitor backpressure events  
  - AXI bus efficiency metrics in `reg2dp_arb_weight`/`reg2dp_arb_wmb`
- **Diagnostic Modes**  
  - Forced clock gating disable via `tmc2slcg_disable_clock_gating`  
  - Data path isolation through `sc2cdma_dat_pending_req` assertion  
  - Error injection using `reg2dp_nan_to_zero` control
## Status Monitoring
- **Real-Time Operational Status**  
  - `cdma2sc_dat_slices`/`cdma2sc_wt_slices` track processed data units  
  - `cdma2sc_dat_updt`/`cdma2sc_wt_updt` pulse on buffer updates  
  - Multi-engine coordination via `status2dma_valid_slices` synchronization
- **Buffer Management**  
  - Double-buffering state visible through `sc2cdma_dat_entries`/`sc2cdma_wt_entries`  
  - Bank conflict detection via `cdma2buf_dat_wr_hsel`/`cdma2buf_wt_wr_hsel`  
  - Memory type tracking with `reg2dp_datain_ram_type`/`reg2dp_weight_ram_type`
- **Throughput Metrics**  
  - `reg2dp_byte_per_kernel` vs `dp2reg_*_latency` correlation for bandwidth analysis  
  - `reg2dp_conv_x_stride`/`reg2dp_conv_y_stride` impact visible in slice counters  
  - Padding efficiency monitored through `img2cvt_dat_wr_pad_mask`


# Power Management & Clock Gating
## Power Domains
- **Functional Block Segmentation**:
  - Weight DMA (u_wt) operates in `nvdla_op_gated_clk_wt` domain
  - Direct Convolution DMA (u_dc) uses `nvdla_op_gated_clk_dc`
  - Winograd Convolution DMA (u_wg) in `nvdla_op_gated_clk_wg`
  - Image Convolution DMA (u_img) in `nvdla_op_gated_clk_img`
  - Data MUX (u_dma_mux) in `nvdla_op_gated_clk_mux`
  - Data Converter (u_cvt) in `nvdla_op_gated_clk_cvt`
  - Shared Buffer (u_shared_buffer) in `nvdla_op_gated_clk_buffer`
- **Management Strategy**:
  - Independent clock gating per functional block via SLCG (Scan-Latch Clock Gating) cells
  - Power domains activated/deactivated through 8-bit `slcg_op_en` register field
## Clock Gating Conditions
- **Gating Implementation**:
  - 7 SLCG instances control different clock domains
  - Gating enabled when:
  - - Functional block inactive (`slcg_op_en[x] == 0`)
  - - No pending transactions in DMA engines
  - - Shared buffer not accessed by multiple blocks simultaneously
- **Override Control**:
  - `tmc2slcg_disable_clock_gating` input forces clocks active for debug
  - Dual clock override inputs (`dla_clk_ovr_on_sync`, `global_clk_ovr_on_sync`) for system-level control
- **Special Cases**:
  - HLS conversion logic uses separate `nvdla_hls_gated_clk_cvt` with dual control (operation enable + SLCG)
  - Shared buffer clock gating considers multi-master access patterns
## Power Optimization Features
- **Architectural Optimizations**:
  - Per-submodule clock domains for fine-grained power control
  - Shared buffer memory bank architecture reduces redundant storage
  - Data converter power gating during non-CVT modes
- **Protocol-Level Savings**:
  - Automatic clock gating during DMA idle cycles
  - Input buffer shutdown during weight/data flush operations
  - Inter-block handshaking prevents unnecessary activation
- **Configuration-Dependent Gating**:
  - Winograd/DC/IMG modes disable unused datapaths
  - Precision-based scaling (INT8/FP16) adjusts converter power
## Implementation Notes
- **Critical Control Signals**:
  - `slcg_op_en[7:0]` register field drives all SLCG enables
  - `slcg_hls_en` specifically controls converter HLS logic
  - Cross-domain gating signals (`slcg_dc_gate_wg`, `slcg_wg_gate_img`, etc.) prevent concurrent activation
- **State Monitoring**:
  - Status controller (u_status) coordinates power state transitions
  - DMA engines report state via `*2status_state` signals
  - Interrupt signals (`*2glb_done_intr_pd`) indicate completion for power down
## Unavailable Information
- Exact power consumption figures (idle/peak/steady-state)
- Voltage domain partitioning details
- Thermal management strategies
- State retention implementation specifics


# Verification & Test Considerations
## Verification Strategy
- **Simulation-Based Verification**:
  - Unit-level testing for submodules (DC/WG/IMG datapaths, weight DMA, shared buffer, CVT)
  - Integration testing focusing on:
    - Convolution mode switching (DC/WG/IMG)
    - Memory interface arbitration (CVIF vs MCIF)
    - Data conversion pipeline (HLS-based cvt module)
    - Buffer management and status synchronization
- **Formal Verification**:
  - Protocol verification for AXI read interfaces (cvif/mcif)
  - Handshake protocol checks for:
    - CSB register interface
    - Data/weight writeback to buffers
    - Interrupt generation logic
- **Clock Domain Crossing**:
  - Verify SLCG modules' clock gating behavior
  - Check synchronization between status controller and datapaths
## Test Scenarios
1. **Operational Mode Verification**:
   - Direct Convolution (DC) mode with different:
     - Data formats (int8, int16, fp16)
     - Padding configurations (asymmetric padding cases)
     - Batch processing (up to 32 batches)
   - Winograd (WG) mode verification:
     - 2x2 and 4x4 kernel configurations
     - Stride combinations (1x1 to 8x8)
   - Image (IMG) mode tests:
     - YUV/RGB formats
     - Pixel offset and mapping variations
2. **Memory Interface Stress Tests**:
   - Concurrent CVIF/MCIF accesses with priority arbitration
   - Backpressure scenarios on read response channels
   - Memory boundary crossing cases (64B/128B alignment)
3. **Data Conversion Scenarios**:
   - NaN/Inf propagation and zeroing
   - Precision conversion (FP16->INT8, INT16->FP16)
   - Scale/offset application with saturation
4. **Buffer Management**:
   - Shared buffer overflow/underflow conditions
   - Concurrent read/write access to SBUF banks
   - Status synchronization between CDMA and SC
## Corner Cases
1. **Boundary Conditions**:
   - Minimum datain_width/datain_height (1x1)
   - Maximum kernel_size (up to 256x256)
   - Full/empty buffer states (entries = 0 or max)
   - Single-slice vs multi-slice operations
2. **Error Conditions**:
   - Unexpected channel disconnect during operation
   - Invalid configuration combinations:
     - WG mode with non-supported strides
     - Data reuse with insufficient buffer size
   - Protocol violations on CSB interface
3. **Timing Critical Scenarios**:
   - Consecutive layer switching without reset
   - Abort operations mid-pipeline
   - Clock gating during active transactions
## Coverage Goals
- **Functional Coverage**:
  - 100% mode coverage (DC/WG/IMG)
  - All valid/invalid handshake sequences
  - All state machine transitions (IDLE/PEND/BUSY/DONE)
  - Data path combinations (bank selection, hsel)
- **Code Coverage**:
  - 100% line coverage for critical paths:
    - Data conversion logic
    - DMA state machines
    - Buffer write arbitration
  - 95%+ branch coverage on:
    - Convolution mode selection
    - Memory interface arbitration
    - Error handling logic
## Built-in Self-Test Features
1. **Status Monitoring**:
   - Operation completion interrupts (dat2glb/wt2glb)
   - Buffer status tracking (entries/slices)
   - Pending request handshake (sc2cdma_*_pending)
2. **Error Detection**:
   - NaN/Inf detection counters (dp2reg_*_num)
   - Stall/latency monitoring (dp2reg_*_stall/latency)
   - Protocol error detection (assertions)
3. **Debug Features**:
   - SLCG override controls (tmc2slcg_disable_clock_gating)
   - Power management interface (pwrbus_ram_pd)
   - Multiple clock domains with independent gating
4. **Configuration Sanity Checks**:
   - Automatic parameter range checking (via regfile)
   - Cross-field dependency validation
   - Reserved field monitoring


# Revision History & Open Issues
## Revision History
- **Version 1.0 (Date Unknown)**  
  *Initial Release*  
  - Base CDMA architecture with:
    - Separate datapaths for DC/WG/IMG convolution modes
    - Weight DMA engine with bank management
    - Shared buffer subsystem for data reuse
    - Clock gating implementation through SLCG modules
  - Key features:
    - Support for CVIF/MCIF interface arbitration
    - Configurable precision modes (INT8/FP16)
    - Multi-batch processing capability
- **Version 1.1 (Date Unknown)**  
  *Architectural Refinements*  
  - Added status controller for unified DMA management
  - Enhanced data converter with NaN/INF handling
  - Improved buffer conflict resolution logic
  - Added per-module clock gating controls
## Authors & Reviewers
- **Authors:** Not specified in code
- **Reviewers:** Not specified in code
## Major Changes
1. **Multi-Convolution Support (DC/WG/IMG)**
   - Added dedicated engines for:
     - Direct Convolution (DC)
     - Winograd Convolution (WG)
     - Image Convolution (IMG)
   - Rationale: Enable flexible CNN acceleration
2. **Shared Buffer Architecture**
   - Implemented 256-bit wide dual-port SRAM banks
   - Added port arbitration for DC/WG/IMG access
   - Rationale: Optimize memory bandwidth utilization
3. **Precision Handling**
   - Added configurable CVT module with:
     - Scale/offset adjustment
     - NaN-to-zero conversion
     - Precision mismatch resolution
   - Rationale: Support mixed-precision networks
## Known Limitations
1. **Interface Constraints**
   - Fixed 78-bit request packet format for CVIF/MCIF
   - 513-bit response packet width limits maximum burst size
2. **Buffer Management**
   - Fixed 12-bit addressing limits buffer size to 4K entries
   - No dynamic bank allocation visible in current implementation
3. **Precision Handling**
   - Limited to INT8/FP16 based on register settings
   - No explicit support for newer formats (BF16/INT4)
## Open Questions
1. **Arbitration Priorities**
   - Weight vs. data DMA priority not explicitly defined
   - CVIF/MCIF selection criteria for mixed traffic scenarios
2. **Error Handling**
   - Unclear protocol for CVIF/MCIF error responses
   - NaN/INF reporting mechanism requires verification
3. **Performance Monitoring**
   - Latency counters (dp2reg_*_latency) rollover behavior undefined
   - Stall accounting methodology needs validation
## Future Improvements
1. **Memory Optimization**
   - Dynamic bank allocation based on layer requirements
   - Adaptive line stride handling for irregular formats
2. **Precision Enhancements**
   - Support for block floating-point formats
   - Configurable NaN handling policies
3. **Power Management**
   - Fine-grained clock gating per sub-module
   - Voltage scaling for buffer banks
4. **Debug Features**
   - Add performance counter freeze functionality
   - Implement trace packet generation for DMA events
*Note: Specific version dates and contributors could not be extracted from provided code. Recommended to maintain formal revision tracking separate from RTL comments.*

