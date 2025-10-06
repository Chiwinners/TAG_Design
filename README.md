# Mr Tagington Technical Documentation
![Main Board 3D Model](images/video_tag.gif)


## Table of Contents
- [System Architecture](#system-architecture)
- [Electronic Specifications](#electronic-specifications)
- [Power Subsystem](#power-subsystem)
- [Onboard Computer](#onboard-computer)
- [Data & Communication](#data--communication)
- [Mechanical Structure](#mechanical-design)
- [PCB design & modelling](#pcb-design--modeling)

## System Architecture
The Mr Tagington device employs a modular dual-PCB architecture that physically separates power management from data processing functions, enabling optimized performance in challenging underwater environments. This separation allows for specialized design considerations: the power board focuses on robust energy harvesting and regulation, while the main processing board handles high-frequency sensor data acquisition and communication tasks.
![Main Board Layout](images/tag_diagram.jpeg)


### Dual-PCB Design


```mermaid
graph TB
    subgraph PowerPCB [Power Management PCB]
        A[Pelton Turbine] --> B[Rectifier Bridge]
        B --> C[CJMCU-2557<br/>MPPT + BMS]
        C --> D[Li-Ion Battery 3.7V]
        C --> E[3.3V Regulator]
        C --> F[3.7V Regulator]
    end
    
    subgraph MainPCB [Main Processing PCB]
        G[STM32L4S5VIT6 MCU]
        
        subgraph Sensing [Sensor Suite]
            H[MS5837 Pressure/Temp]
            I[ICM-20948 IMU]
            J[HMC5883L Magnetometer]
            K[FDC1004 Capacitive]
            L[MAX-M10S GPS]
        end
        
        subgraph StorageComm [Storage & Communication]
            M[GD25Q16 Flash Memory]
            N[Iridium SLM9670]
        end
    end
    
    A --> B --> C --> D
    E --> G
    F --> Sensing
    G --> Sensing
    G --> StorageComm
```

**Key Design Features:**
- **Power PCB**: Hydraulic energy harvesting, battery management, dual-voltage regulation
- **Main PCB**: Sensor processing, behavioral algorithms, data storage, satellite communication
- **Dedicated Power Rails**: 3.3V for digital, 3.7V for analog components

## Electronic Specifications

### Microcontroller & Processing

| Component | Specification | Key Features |
|-----------|---------------|--------------|
| **STM32L4S5VIT6** | ARM Cortex-M4 @ 120MHz | Ultra-low-power, Multiple communication interfaces, Floating-point unit |

### Environmental Sensors

| Sensor | Measurement | Range | Interface |
|--------|-------------|-------|-----------|
| **MS5837-02BA** | Pressure: 0-30 bar<br>Temperature: -20°C to +85°C | ±0.01 mbar<br>±0.01°C | I²C1 |
| **FDC1004DSCT** | Capacitance: 0-15 pF | ±0.1 fF | I²C2 |

### Motion & Orientation Sensors

| Sensor | Measurement | Range | Interface |
|--------|-------------|-------|-----------|
| **ICM-20948** | Acceleration: ±16g<br>Gyroscope: ±2000°/s<br>Magnetometer: ±4900 μT | 16-bit resolution | SPI1 |
| **HMC5883L** | Magnetic Field: ±8 Gauss | ±0.1 μT | I²C3 |

### Communication & Storage

| Component | Function | Specification | Interface |
|-----------|----------|---------------|-----------|
| **MAX-M10S-00B** | GPS Positioning | 2.5m accuracy | UART2 + EXTINT |
| **GD25Q16ESIGR** | Data Storage | 16 Mbit (2MB) | SPI2 |
| **Iridium SLM9670** | Satellite Comm | Global coverage | SPI2 + GPIO |

### Design Criteria & Implementation

### Bus Architecture Rationale

**Dedicated Communication Buses:**
- **I²C1**: MS5837 Pressure/Temperature sensor
- **I²C2**: FDC1004 Capacitive sensor  
- **I²C3**: HMC5883L Magnetometer
- **SPI1**: ICM-20948 IMU (high-speed data)
- **SPI2**: GD25Q16 Flash + Iridium module
- **UART2**: MAX-M10S GPS module

**Benefits of Dedicated Buses:**
- Eliminates bus contention and arbitration delays
- Enables simultaneous sensor operation
- Simplifies debugging and fault isolation
- Optimizes power consumption per peripheral

### Key Design Innovations

#### 1. Intelligent Sensor Activation

**Pressure-Triggered GPS Wake-up:**
- EXTINT pin (PC5) monitors pressure changes
- GPS activates only during surfacing events
- Eliminates constant GPS power drain underwater

#### 2. Redundant Magnetic Sensing

**Dual Magnetometer Configuration:**
- Primary: ICM-20948 integrated magnetometer
- Secondary: HMC5883L dedicated sensor
- Provides orientation backup and calibration reference

#### 3. Robust Mechanical Integration

**Environmental Protection:**
- MS5837 thermally exposed for accurate water temperature
- Capacitive sensors isolated from mechanical stress
- Multiple ground planes for RF performance
- Pressure-compensated enclosure design

### Pin Assignment & Connectivity

#### Microcontroller Interface Mapping

| Peripheral | Interface | Pins | Purpose |
|------------|-----------|------|----------|
| **MS5837** | I²C1 | PB6 (SCL), PB7 (SDA) | Pressure & Temperature |
| **FDC1004** | I²C2 | PB10 (SCL), PB11 (SDA) | Capacitive Sensing |
| **HMC5883L** | I²C3 | PC0 (SCL), PC1 (SDA) | Magnetic Field |
| **ICM-20948** | SPI1 | PA5 (SCK), PA6 (MISO),<br>PA7 (MOSI), PA4 (CS) | Motion & Orientation |
| **GD25Q16** | SPI2 | PB13 (SCK), PB14 (MISO),<br>PB15 (MOSI), PB12 (CS) | Data Storage |
| **MAX-M10S** | UART2 | PA2 (TX), PA3 (RX),<br>PC5 (EXTINT) | GPS + Wake-up |
| **Iridium** | SPI2 + Control | Shared SPI2, PB0 (RESET) | Satellite Communication |

### Performance Characteristics

#### Data Acquisition Capabilities

**Continuous Monitoring:**
- IMU sampling: 100-200 Hz for biomechanical analysis
- Environmental sensors: 1-10 Hz based on state machine
- Magnetic field: 10-50 Hz for orientation tracking

**Storage Capacity:**
- 2MB flash storage ≈ 2-4 weeks of continuous data
- Adaptive compression based on behavioral patterns
- FIFO management for long-term deployments

## Power Subsystem

### Energy Harvesting & Management
The autonomous power system converts hydraulic energy to stable DC power:

```mermaid
graph TB
    subgraph Generation
        A[Pelton Turbine] --> B[Rectifier Bridge]
    end
    
    subgraph Management
        C[CJMCU-2557 BQ25570] --> D[Battery Management]
        C --> E[MPPT Control]
    end
    
    subgraph Storage
        F[Li-Ion Battery 3.7V] --> C
    end
    
    subgraph Distribution
        C --> G[3.3V Regulator] --> H[Digital Load]
        C --> I[3.7V Regulator] --> J[Analog Load]
    end
    
    B --> C
    E --> A
```

**Operational Modes:**
- **Active Generation**: Water flow → Turbine → Battery charging + load power
- **Battery-Only**: Battery → Regulators → Load power (no-flow conditions)
- **Combined**: Turbine + Battery → Optimized power distribution

**Key Components:**
- **Pelton Turbine**: Hydraulic-to-electrical conversion
- **CJMCU-2557**: Integrated MPPT + battery management
- **Dual Regulators**: 3.3V (digital) and 3.7V (analog) rails

## Onboard Computer
The onboard computer implements an intelligent state management system that dynamically optimizes the balance between data collection and power conservation through four distinct operational states. In Hibernation mode, the system maintains minimal power consumption by activating only the IMU for basic motion monitoring while tracking battery levels and time. This state serves as the default operational mode, consuming less than 500μA while waiting for activation triggers. When battery levels drop below 3.3V, the system transitions to Battery Saving mode, performing a complete shutdown of all non-essential systems to conserve energy until sufficient power is recovered. Activation events—either time-based (1800-second intervals) or behavior-triggered (hunting pattern detection)—prompt transition to Sensing state, where all environmental, motion, and position sensors are activated for comprehensive data collection, processing, and local storage. Finally, when conditions permit (adequate battery, surface proximity, and satellite availability), the system enters Transmission state, activating the satellite communication module and implementing a robust three-attempt retry protocol to ensure successful data delivery before returning to hibernation. This state machine architecture enables extended autonomous operation by ensuring that power-intensive components are activated only when necessary and for minimal durations.

### State Management System
The control system implements an intelligent state machine for optimal power and data management:

```mermaid
flowchart TD
    subgraph Hibernation
        H1[Monitor: IMU, Battery, Time] --> H2{Trigger event?}
    end

    H2 -->|Battery < 3.3V| BatterySaving
    H2 -->|Timer/Hunting| Sensing
    
    subgraph BatterySaving
        BS1[All Systems OFF] --> BS2{Battery > 3.5V?}
        BS2 -->|No| BS1
    end
    BS2 -->|Yes| Hibernation

    subgraph Sensing
        S1[Activate Sensors] --> S2[Read Data] --> S3[Process & Store] --> S4{Transmit Conditions?}
    end
    S4 -->|Yes| Transmission
    S4 -->|No| Hibernation

    subgraph Transmission
        T1[Activate Comm] --> T2[Transmit Data] --> T3{Success?}
        T3 -->|No| T4{Attempts < 3?} -->|Yes| T2
        T4 -->|No| Hibernation
        T3 -->|Yes| Hibernation
    end
```

**Operational States:**
- **Hibernation**: IMU-only monitoring, ultra-low power
- **Battery Saving**: Complete shutdown, emergency conservation
- **Sensing**: Full sensor activation and data collection
- **Transmission**: Satellite communication with retry logic

## Data & Communication
The data and communication system integrates a comprehensive sensor suite—including pressure, temperature, motion, magnetic field, and GPS sensors—that captures both environmental conditions and behavioral metrics. Raw sensor data is processed into calculated parameters such as depth, speed, and hunting probability, then encoded into efficient 64-character plain text strings for transmission. The system employs adaptive transmission modes, scaling from full datasets during optimal conditions to essential-only data during power conservation. 

### Sensor Suite & Processing

| Category | Component | Measurements | Interface |
|----------|-----------|-------------|-----------|
| **Environmental** | MS5837 | Pressure (0-30 bar), Temperature (-20°C to +85°C) | I²C1 |
| **Motion** | ICM-20948 | Acceleration (±16g), Gyroscope (±2000°/s) | SPI1 |
| **Magnetic** | HMC5883L | Magnetic field (±8 Gauss) | I²C3 |
| **Position** | MAX-M10S | GPS positioning | UART2 |
| **Storage** | GD25Q16 | 2MB data storage | SPI2 |
| **Communication** | Iridium SLM9670 | Global satellite transmission | SPI2 |

**Calculated Behavioral Metrics:**
- **Depth**: `(pressure - 1013.25) / 100` meters
- **Speed**: Position delta over time (m/s)
- **Hunting Score**: Multi-parameter movement analysis
- **Activity Index**: Acceleration magnitude and patterns

### Data Transmission Protocol
The following pipeline is designed for data transmission:

```mermaid
flowchart TD
    A[Raw Sensor Data] --> B[Parameter Calculation]
    B --> C[Data Encoding]
    C --> D[Checksum Generation]
    D --> E{Transmission Mode}
    
    E -->|Normal Operation| F[64-char String<br/>Full dataset]
    E -->|Low Power Mode| G[32-char String<br/>Essential data only]
    E -->|Emergency Mode| H[16-char String<br/>Critical data only]
    
    F --> I[Transmit]
    G --> I
    H --> I
```

**Fixed-Width Encoding (64-character):**
```
TTTTSSSSPPPPBBBBGGGGLLLLAAAAIIIIMMMMDDDDHHHHXXXXYYYYZZZZRRRRCCCC
```

**Data Structure:**
- `TTTT`: Time (HHMM), `SSSS`: Seconds, `PPPP`: Pressure (mbar×10)
- `BBBB`: Battery (mV), `GGGG/LLLL`: Latitude/Longitude (degrees×1000)
- `AAAA`: Acceleration (g×1000), `CCCC`: Checksum

**Adaptive Transmission Modes:**
- **Normal** (64 chars): Full dataset when surfacing + good power
- **Low Power** (32 chars): Essential data during conservation
- **Emergency** (16 chars): Critical status during faults

### Design Innovations

**Bus Architecture:**
- Dedicated I²C/SPI buses per sensor eliminate contention
- Simultaneous operation and simplified debugging
- Optimized power consumption per peripheral

**Intelligent Activation:**
- Pressure-triggered GPS wake-up (EXTINT pin)
- Dual magnetometer configuration for redundancy
- Event-driven sensor activation based on behavioral triggers

**Environmental Robustness:**
- IP68-rated pressure-compensated enclosure
- Thermally exposed sensors for accurate measurements
- Corrosion-resistant materials for marine deployment

This integrated system enables long-term autonomous monitoring of shark behavior through optimized power management, comprehensive sensing, and reliable satellite communication.


## Mechanical Design

### Structural Components

- **Hull**:  Base of the FDM-3D printed structural core. Manufactured in PLA with 100% infill for structural integrity and water-proofed with a coat of Vinyl Ester Resin. Holds the electrical components on the inside and the attatchment to the torsional spring on the outside.

![Hull Component](images/Hull.png)

- **Cover**: Cover of the FDM-3D printed structural core, complimentary to teh Hull with lip interlocking. Manufactured in PLA with 100% infill for structural integrity and water-proofed with a coat of Vinyl Ester Resin. Holds the snesors in contact to the outside and both the housing and rotational seal for the pelton turbine axle.

![Cover Component](images/Cover.png)

- **Pad Holds**: FDM-3D printed supports for the grip pads that non-invasively attatch to the sharks dorsal fin. Manufactured in PLA with 100% infill and water-proofed with a coat of Vinyl Ester Resin, these recieve the torsional spring and attatch to the silicone pads through an cyanoacrylate adhesive.

![Pad Holds Component](images/Pads.png)

- **Pelton Turbine**: FDM-3D printed Pelton turbina compatible with dc motor axle. Manufactured in PLA with 100% infill and and water-proofed with a coat of Vinyl Ester Resin, this component transforms the water flow along the surface of the tag into energy for the tag through a DC motor and the power system.

![Pelton Turbine](images/Pelton.png)

- **GFRP Skin**: 2mm thick skin of Glass Fiber Reinforced Polymer applied over the hole structured and binded to the nucleus through Vinyl Ester Resin. Manufactured from glass fiber symmetric plain weave and a Vinyl Ester Resin matrix, it handles the direct structural load due to external pressure and is supported angainst bukling by the PLA 3D printed nucleus.

### Design Criteria

#### Marine Environment Considerations
- **Corrosion Resistance**: Saltwater surface flow exposure is prone to corrosively degrade materials, this reduces ultimate product life and could cause loss of the electrical components due to structural failure as a result of the corrosion of the structure
- **Pressure Tolerance**: Water depth requires the exposure to equivalent external pressures for sealed bodies (such as our tag), approximations dictate 1 bar of pressure pero 10 meters of depth, however for depths of around 300 meters (current working limits of this prototype) the tag must be able to withstand approximately 30 bar or 3 MPa. Applying a safety factor of 1.25.
- **Biofouling Mitigation**: Crevice presence in bodies exposed to seawater often experience marine growth, smooth surfaces mitigates this.

#### Operational Requirements
- **Attachment Security**: Tag attatchment must adhere to non-invasive methods and avoid both securing through penetration (fin or skin) and excessive pressure on the animal's limbs.
- **Hydrodynamic Efficiency**: The tag must maintain a minimal drag coefficient and avoid impeding the animal's movement as much as possible.
- **Durability**: The tag must be able to withstand deployment and functionality for extended periods of time, managing dtructural durability and energy generation.
- **Manufacturing Scalability**: The tag must be easy to manufacture via 3D printing prototyping and scalable to mission requirements.
- **Structual Stability**: The tag must be able to withstand structurally the calculated pressure of 3 MPa with a safety factor of 1.25, resulting in a design pressure of 3.77 MPa. 

#### Design Constraints
- **Size Limitations**: The tag must allow space for all the electronic components while maintaining the smalles dimensions possible in order to lessen the impact on the animal's mobility. 
- **Hydrodynamic Interaction**: The hydrodynamic profile of the tag must minimize drag while allowing for power generation through a pelton turbine. 
- **Manufacturing Feasibility**: The use of 3D printing in the tag's manufacturing must allow foir rapid prototyping and adaptability to changing requirements.
- **Regulatory Compliance**: The tag must minimize of outright end any potential negative impact on animal welfare and enviromental safety, this can be done through non-invasive approaches to tag attatchment and the use of non polutting materials.

### Mechanical Design - Enclosure Structure

#### Structural Approach with Reinforcement
- **Structural Concept**: The enclosure is composed of an FDM 3D-printed **PLA nucleus** with a **3 mm wall thickness**, it is impregnated and sealed with a **vinyl ester resin** to eliminate smooth the surface and water-proof the structure.  
- **Reinforcement Layer**: A **2 mm Glass Fiber Reinforced Polymer (GFRP)** skin is applied over the nucleus using a vacuum layup. The composite layup uses a symmetric [±45/0/0/±45] stacking sequence of plain weave glass fiber for circumferential stiffness against external pressure loads.  
- **Material Rationale**: GFRP was selected instead of CFRP to ensure **magnetic transparency** for the onboard magnetometer and to prevent conductive interference.  
- **Interfacial Bonding**: The PLA–GFRP interface is bonded using the same vinyl ester resin matrix, this provides a continuous shear load transfer and reduces the risk of delamination.  
- **Sealing Method**: All three joints use static O-ring seals, two circular (18 mm diameter) and one square (3.9 × 4.9 mm), these are made from **EPDM/Viton (75–90A)** which allows for marine-grade sealing.  
- **Surface Treatment**: A smooth resin finish mitigates marine biofouling and improves laminar boundary flow across the hull surface.

#### Aerodynamic (Hydrodynamic) Profile Design
- **Streamlined Geometry**: The tag’s cross-section combines a **semicircular leading edge** external to de clamp on the animal's dorsal fin and a **rectangular base section**, yielding an approximate frontal area of **0.00256 m² (25.6 cm²)**. The external GFRP coating and rounded transitions reduce local flow separation.
  ![Front](images/Front_Profile-2.png)
- **Energy Generation Integration**: The body includes an **aperture for a Pelton turbine** on the lateral section. The opening's borders are filleted and aligned to minimize disturbance and separation bubbles in the flow of water through the turbine, allowing the turbine to harness part of the passing flow for onboard energy generation through a 9V DC motor.  
- **Flow Optimization**: The combined hull and cover form a smooth, continuous hydrodynamic contour which fades of to an end tail. The addition of the turbine increases the total drag coefficient to an estimated **Cd ≈ 0.6**, accounting for flow interference, while maintaining overall directional stability.
  ![Front](images/Side_Profile.png)

### Pressure Design Calculations

**Hydrostatic external pressure**

$$
p = \rho\, g\, h
$$

Where: $\rho = 1025\ \mathrm{kg\,m^{-3}}$, $g = 9.81\ \mathrm{m\,s^{-2}}$, $h = 300\ \mathrm{m}$.

**At 300 m (gauge pressure):**

$$
\begin{aligned}
p_\text{gauge}
  &= 1025 \times 9.81 \times 300 \\
  &= 3{,}016{,}575\ \mathrm{Pa} \\
  &= 3.02\ \mathrm{MPa}\ (30.17\ \mathrm{bar})
\end{aligned}
$$

**Design pressure with safety factor $FS=1.25$:**

$$
p_\text{design} = FS \cdot p_\text{gauge} = 1.25 \times 3.02\ \mathrm{MPa}
= 3.77\ \mathrm{MPa}\ (37.7\ \mathrm{bar})
$$

Thus, the enclosure must resist a **design pressure of 3.77 MPa (≈37.7 bar)** without yielding or delamination.  
The GFRP skin handles the bulk of the pressure load, while the PLA core prevents buckling in the composite skin through internal support.  
The resulting 5 mm composite wall (3 mm core + 2 mm GFRP) provides an good margin against both collapse and wrinkling failure.


#### 3D Printing Development and Manufacturing
- **Process Parameters**: The pieces are setup to be printed using **0.2 mm nozzle** for better printing resolution, **0.12–0.16 mm layer height** in order to minimize inter-layer crevices, and **100% infill** to maximize structural integrity.  
- **Tolerances**: ±0.2 mm from initial printing setup, with post-processing (fine sanding and resin impregnation) a finer margin can be achieved to prepare sealing-grade surfaces for O-rings.  
- **Impregnation Phase**: The printed hull is first coated in **vinyl ester resin** under vacuum seal to seal print lines before GFRP layup.  
- **Layup Phase**: The 2 mm GFRP skin is applied using vacuum assisted layup methods, ensuring high fiber–resin adhesion and minimizing porosity in the matrix.  
- **Final Assembly**: Mating surfaces (hull–cover joint and sensor port) are sanded to tight fit but sealed with resin nonetheless.


### Mechanical Design - Attachment System

#### Non-Invasive Coupling Approach
The tag attaches to the shark’s dorsal fin using a **non-invasive clamp**.  
This was a pretty strict requirement from the start — no penetration, no sutures, nothing that would hurt or stress the animal. The design uses **two pads** that gently press on both sides of the fin, each one lined with a **silicone layer** to distribute pressure evenly and prevent slipping.  

The clamp closes through a **torsional spring** mounted on a pivot axis, which allows it to apply a consistent force while still being flexible enough to adjust to the fin’s shape. The pads have a **textured surface**, to maximize grip


#### Torsional Spring Mechanism
The torsional spring handles the job of keeping the clamp closed without any active locking.  
Two arms connected at the hinge with a coiled spring that resists opening. When the tag is attached, the spring exerts its regular clamp force without being over stressed.

From tests and calculations, the clamp needs to withstand the drag forces acting on the tag at high swimming speeds. At around **10 m/s**, the tag can generate roughly **67 N of drag**, based on the drag coefficient and frontal area.

$$
F_D = \tfrac{1}{2} \rho C_d A v^2
$$

where  
$\rho = 1025\ \mathrm{kg/m^3}$ (seawater density),  
$C_d = 0.6$,  
$A = 0.00219\ \mathrm{m^2}$,  
$v = 10\ \mathrm{m/s}$  

so

$$
F_D = 0.5 \times 1025 \times 0.6 \times 0.00219 \times 10^2 \approx 67\ \mathrm{N}
$$

To keep the tag in place, the friction between the pads and the fin must at least balance that drag.  
With a friction coefficient $\mu = 0.5$ and **two clamp pads** sharing the load, the normal force per pad should be:

$$
N = \frac{F_D \times FS}{2 \mu}
$$

Using a safety factor $FS = 1.25$:

$$
N = \frac{67 \times 1.25}{2 \times 0.5} = 83.75\ \mathrm{N}
$$

Each pad needs roughly **84 N** of normal force.  
Enough to hold firmly without crushing tissue — the silicone helps spread it over about **60 cm²**, which keeps the contact pressure below **15 kPa**, well within safe limits for soft biological surfaces.


#### Spring Calculations and Grip Pad Sizing
From those numbers, the torsional spring was chosen to deliver about **120 N·mm per degree** at its working angle, with preload adjusted so it hits that 84 N per side when the clamp is closed.  
Each pad measures **10 cm × 6 cm**. This design can be scaled down for shorter fin cords while maintaining less than the critical **100 kPa** researched.  

The spring itself is stainless steel, marine-grade and easy to replace if fatigue ever shows up. The housing isolates it from direct seawater contact through a thin resin coating to avoid corrosion over long deployments.


### Mechanical Design - Pelton Turbine

#### Turbine Configuration
The tag carries a **small Pelton wheel** on the lateral section, placed so that the flow moving past the tag hits the buckets almost tangentially.

The turbine connects to a **9 V DC motor**, which doubles as a generator. It’s shaft sealed using a lip seal to prevent water leaking into the inside of the tag structure.

The hole that houses the turbine is shaped carefully — all edges are filleted, and the inner pocket blends smoothly into the main surface to reduce local turbulence. Still, adding the turbine naturallyincreases drag. Based on the projected area of the runner (about 25 mm diameter), the extra drag can be approximated by:

$$
\Delta F = \tfrac{1}{2} \rho C_{d,p} A_p v^2
$$

with $C_{d,p} \approx 1.1$, $A_p = \pi (0.0125)^2 = 4.9 \times 10^{-4}\ \mathrm{m^2}$, and $v = 10\ \mathrm{m/s}$:

$$
\Delta F \approx 0.5 \times 1025 \times 1.1 \times 4.9\times10^{-4} \times 10^2 \approx 27.7\ \mathrm{N}
$$

So roughly **28 N of additional drag** from the turbine at top speed — acceptable, considering it’s generating usable power at that point. The geometry keeps the flow attached and avoids cavitation, which could otherwise tear up the small buckets.


#### Flow and Structural Notes
The turbine cavity reinforces internally through the GFRP shell; no direct shaft penetrations are made. Heat from the motor dissipates through the composite wall into the surrounding seawater. The result is a compact, self-contained generator that stays sealed up to **300 m depth**, without risking leaks from rotating seals.



## PCB Design & Modeling

The electronic systems were implemented through comprehensive PCB design and 3D modeling, following the modular dual-PCB architecture described in the system specifications. Both boards were fully designed, routed, and validated virtually, with detailed 3D models created to verify mechanical integration and environmental compatibility.


### Main Processing PCB Design
The main board design integrates the complete sensor suite, processing core, and communication systems with emphasis on signal integrity and EMI mitigation in the virtual environment.


#### Schematic Design
![Main Board Schematic](images/schematic_diagram.png)
*Comprehensive schematic showing microcontroller implementation and sensor interfaces*



#### 3D Model
![Main Board 3D Model](images/view2_3D.png)
*3D model illustrating component clearance and mechanical mounting points*

### Power PCB
The power board design integrates the complete energy harvesting system, MPPT and BMS.
#### Schematic Design
<img src="images/schematic_power.jpeg" width="400" alt="Power Schematic">

*Comprehensive schematic showingpower subsystem*



#### 3D Model
<img src="images/3d_power.jpeg" width="400" alt="Power Schematic">

*3D model illustrating component clearance and mechanical mounting points*



Both PCB designs underwent extensive design rule checking, signal integrity simulation, and 3D mechanical validation to ensure the theoretical design meets all electrical and mechanical requirements for marine environment operation. The virtual prototyping approach allowed for comprehensive optimization of the system architecture before physical implementation.
