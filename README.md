# Project VERION: Smartphone–Intelligent Drone Hybrid

> **Version 0.31** — Reorders the document so the capability-based shortlist of five companies (Section 8) now appears before the Open Questions section; no analytical content was changed from v0.3. Version 0.3 added that shortlist alongside the societal benefits and impact analysis. This document is a living specification and will be revised as the design matures.

---

## Executive Summary

This document outlines the concept, technical requirements, and open risks for a hybrid device: a fully functional smartphone that can detach, unfold, or reconfigure itself into a small autonomous quadrotor drone. The device would serve as a personal communication tool and an on-demand aerial sensing/camera platform, controlled either directly by the user or semi-autonomously via onboard AI.

---

## 1. Technology and Innovations Required

Building a device that is simultaneously a daily-use smartphone and a flight-capable drone requires innovation across mechanical, electrical, and software domains.

### 1.1 Mechanical / Structural
- **Transforming chassis**: a folding or telescoping frame that converts a flat phone form factor into a quadrotor or foldable multi-rotor frame. Likely candidates: scissor-linkage arms, origami-inspired fold patterns, or telescoping carbon rods that deploy from the phone body.
- **Integrated propulsion bays**: recesses or cavities in the phone chassis that house folded propellers and motors, sealed by sliding or hinged covers when in "phone mode."
- **Self-locking joints**: mechanisms (spring-loaded latches, motorized locking pins) that hold the frame rigid in flight mode and flat in phone mode, able to survive thousands of transformation cycles.
- **Weight-distribution management**: battery and motor placement must keep the center of gravity stable in both configurations, since a phone's mass distribution (heavy at the camera/battery end) is very different from a drone's ideal symmetric layout.

### 1.2 Propulsion and Flight Control
- **Micro brushless motors** small and light enough to nest inside a phone-sized housing, with a folding propeller design (see Section 6).
- **Flight controller stack**: IMU (accelerometer, gyroscope, magnetometer), barometer, and a dedicated flight-control microcontroller running a real-time OS or RTOS-scheduled task alongside the main mobile SoC.
- **Redundant power delivery**: flight draws far more instantaneous current than normal phone use; the power management IC (PMIC) and battery chemistry must support high discharge rates without compromising standby phone battery life.

### 1.3 Sensing and Autonomy
- **Visual-inertial odometry (VIO)** and **SLAM** for GPS-denied indoor navigation, reusing the phone's existing cameras where possible plus dedicated downward/forward optical flow sensors.
- **Obstacle avoidance**: time-of-flight (ToF) sensors, stereo cameras, or millimeter-wave radar integrated into the drone's leading edges.
- **Onboard AI inference chip**: a neural processing unit (NPU) capable of running perception and control policies in real time (see Section 3 for RL and LLM tradeoffs).
- **Wireless mesh/telemetry**: robust low-latency link between phone-mode cellular/Wi-Fi radios and drone-mode telemetry, likely repurposing existing 5G/Wi-Fi 6E hardware with a dedicated low-latency control channel.

### 1.4 Software
- **Unified OS layer** that manages mode-switching, safely disabling cellular radio interference-sensitive components during flight and vice versa.
- **Flight firmware** (e.g., a customized PX4/ArduPilot-style stack) tightly coupled to the mobile OS rather than running as a fully separate system, to allow shared sensor and compute resources.
- **On-device AI assistant** for route planning, natural-language flight commands, and autonomous return-to-home / follow-me behavior.
- **Safety interlocks**: software-enforced no-fly zones, geofencing, and automatic grounding when structural sensors detect an incomplete or unsafe transformation.

---

## 2. Device Measurements

Target dimensions for a device that must remain pocketable as a phone yet aerodynamically viable as a drone.

| Configuration | Width | Height/Length | Thickness | Weight |
|---|---|---|---|---|
| **Phone mode (folded)** | 75 mm | 160 mm | 12–14 mm | 230–260 g |
| **Drone mode (arms deployed, diagonal motor-to-motor)** | 210–240 mm | 210–240 mm | 45–55 mm (with props and legs) | 230–260 g (same mass; propellers/arms add negligible mass) |
| **Individual folding propeller (deployed)** | — | 65–75 mm blade span | 4–6 mm | ~3–5 g each (×4) |
| **Arm/leg extension (each, telescoped)** | 8–10 mm diameter | 90–110 mm extended | — | ~12–18 g each (×4) |

**Design notes:**
- Phone-mode thickness is aggressive; comparable to a slightly thick flagship phone (for reference, most 2025–2026 flagships sit at 7.5–9 mm). Housing folded motors/propellers will likely push this device toward the thicker end (12–14 mm) unless motors are made extremely thin (pancake-style axial-flux motors help here).
- Total flight weight (~250 g) is deliberately kept under many regional recreational-drone registration thresholds (e.g., 250 g is a common regulatory line), though actual classification depends on local aviation authority rules.
- A 210–240 mm diagonal motor spacing is small for a quadrotor and will limit payload, wind resistance, and flight time — a fundamental tension discussed in Section 4.

---

## 3. AI Systems: LLM and Reinforcement Learning Comparison

The device needs two distinct AI capabilities: a **conversational/planning layer** (LLM) for natural-language interaction and mission planning, and a **low-level control layer** (RL) for real-time flight stabilization and obstacle avoidance. These serve very different roles and should not be conflated.

### 3.1 LLM Options for On-Device / Hybrid Use

| Option | Strengths | Weaknesses | Fit for this device |
|---|---|---|---|
| **Small on-device LLM (2–4B parameters, quantized)** | Runs fully offline on an NPU; low latency; no connectivity dependency; better privacy for flight telemetry and location data | Weaker reasoning, limited world knowledge, may struggle with complex multi-step mission planning | Best fit for core commands ("hover here," "follow me," "return home") where latency and offline reliability matter most |
| **Mid-size on-device LLM (7–8B, quantized)** | Better reasoning and instruction-following than smaller models; still runnable on modern mobile NPUs with 8–12GB unified memory | Higher power draw, competes with flight-control compute budget, slower response under thermal throttling | Viable for richer conversational planning when the device is in phone mode and not actively balancing flight-control compute load |
| **Cloud-hosted large LLM (via API)** | State-of-the-art reasoning, large context, best for complex mission descriptions ("survey this field and flag irrigation issues") | Requires connectivity, adds latency unsuitable for real-time flight decisions, raises data-transmission privacy/security concerns for live video/location | Best used for pre-flight mission planning and post-flight analysis, never for in-flight control loops |
| **Hybrid routing (on-device small model + cloud fallback)** | Balances responsiveness and capability; simple commands stay local, complex ones route to cloud when connectivity allows | Added system complexity; requires a reliable arbitration layer and graceful degradation when offline | Most practical real-world approach — matches how most current AI-assistant-equipped hardware is architected |

**Key takeaway**: the LLM should never be in the flight-critical control loop. It handles intent, planning, and conversation; a separate, much faster and more deterministic system handles stabilization.

### 3.2 Reinforcement Learning Algorithms for Flight Control

| Algorithm | Type | Strengths | Weaknesses | Fit for this device |
|---|---|---|---|---|
| **PPO (Proximal Policy Optimization)** | On-policy, stochastic policy | Stable training, widely used in real drone RL (e.g., agile flight research), tolerant of reward-shaping mistakes | Sample-inefficient; needs large amounts of simulated flight data before deployment | Good for training the base attitude/stabilization policy in simulation before sim-to-real transfer |
| **SAC (Soft Actor-Critic)** | Off-policy, maximum-entropy, continuous action | Sample-efficient, good exploration, strong in continuous control tasks like thrust/attitude control | Can be less stable than PPO on some reward landscapes; more hyperparameter-sensitive | Strong candidate for fine-tuning control policies where real-flight data is expensive to collect |
| **TD3 (Twin Delayed DDPG)** | Off-policy, deterministic policy | Reduces overestimation bias vs. DDPG, sample-efficient, good for continuous control | Less exploratory than SAC (deterministic policy), can converge to suboptimal behaviors in sparse-reward settings | Useful for precision hovering/positioning tasks once basic flight is stable |
| **HER (Hindsight Experience Replay, combined with SAC/TD3)** | Off-policy augmentation for goal-conditioned tasks | Learns efficiently from failed attempts by relabeling goals; excellent for sparse-reward navigation ("reach this waypoint") | Adds complexity; benefits shrink for dense-reward tasks like basic stabilization | Best suited to goal-conditioned tasks such as autonomous docking/re-folding or precision landing on a charger |

**Practical architecture recommendation**: use PPO in high-fidelity simulation to bootstrap a robust base stabilization policy, transfer to hardware, then refine with SAC or TD3 (+HER for goal-conditioned docking/landing behaviors) using real flight data, following a similar sim-to-real pipeline used in current legged-robot and micro-aerial-vehicle research.

---

## 4. Risks and Challenges in Building the Hybrid Device

- **Weight/power/thermal budget conflict**: batteries good for a full day of phone use are typically not optimized for high-discharge flight bursts, and vice versa. A single battery chemistry serving both use cases will always be a compromise.
- **Structural fatigue**: transformation joints subject to thousands of fold/unfold cycles are a likely failure point; a single stuck or partially-deployed arm is a critical flight hazard.
- **Aerodynamic penalty of a small, dense airframe**: a 210–240 mm diagonal is very small for a quadrotor; expect short flight times (likely under 5–8 minutes), high sensitivity to wind, and limited payload margin for extra sensors.
- **Electromagnetic interference**: motors, ESCs, and high-current propulsion wiring sit close to cellular/Wi-Fi antennas and sensitive camera/IMU electronics, risking interference with both flight sensors and phone radios.
- **Thermal management**: the SoC, NPU, flight controller, and motor drivers all generate heat in an enclosure with little room for heat sinks or fans, especially problematic mid-flight when performance demands peak.
- **Certification complexity**: the device must simultaneously satisfy phone/radio certification (FCC/CE for cellular and Wi-Fi) and aviation authority requirements (e.g., FAA Part 107 or equivalent, remote ID broadcasting) — two very different regulatory regimes bundled into one SKU.
- **Repairability and cost**: precision folding mechanisms, micro motors, and dual-purpose electronics will likely make the device expensive to manufacture and difficult/costly to repair compared to either a standalone phone or standalone drone.
- **Sim-to-real transfer risk**: RL policies trained in simulation may not transfer cleanly to the specific, unusual aerodynamics of a phone-shaped drone body, requiring extensive real-world tuning.

---

## 5. Benefits to Society and How Society Will Be Affected

A device that merges a pocketable phone with an on-demand aerial platform could shift how ordinary people access aerial capability, currently limited mostly to hobbyists, professionals, and institutions.

- **Democratized aerial perspective**: everyday users gain instant access to overhead views for everyday tasks — checking a roof for storm damage, scouting a hiking trail ahead, finding a parking spot in a crowded lot, or capturing a family event from above — without owning or carrying separate drone hardware.
- **Emergency and personal safety uses**: a phone that can fly could locate a lost hiker's own position relative to a trail, search for a misplaced pet or child within sight of a campsite, or give a stranded driver a quick view of surrounding terrain, all using a device the person already has on them.
- **Search and rescue at civilian scale**: widespread ownership means that in a localized disaster (flood, earthquake, missing person), a far larger number of aerial "eyes" could be mobilized quickly by ordinary bystanders and coordinated by responders, supplementing dedicated rescue drones.
- **Accessibility gains**: people with mobility limitations could gain a new way to inspect hard-to-reach areas of their own home or property (gutters, roofs, high shelves) without climbing or requiring assistance.
- **Lowered cost of entry for creative and educational use**: students, hobbyist photographers, and small content creators would no longer need to buy a separate drone to experiment with aerial photography, mapping, or robotics/AI education, potentially broadening participation in STEM and creative fields.
- **Small business and gig-economy enablement**: real estate agents, small farmers, and local contractors could get basic aerial documentation (property overviews, crop checks, roof inspections) from a device they already carry, lowering costs currently tied to dedicated drone equipment or hired operators.
- **Shift in social norms around personal technology**: just as smartphones normalized always-on photography and connectivity, a flying phone would likely normalize casual aerial observation of daily life, changing public expectations around privacy in shared and semi-private spaces (yards, parks, streets) and likely accelerating "assume you might be filmed from above" norms.
- **Infrastructure and urban planning pressure**: cities and regulators would likely need to adapt low-altitude airspace management, insurance frameworks, and public-space rules well before mass adoption, similar to how ride-sharing and e-scooters forced rapid municipal policy adaptation.
- **Net effect is double-edged**: the same ubiquity that enables convenience, safety, and creative benefits (Section 5) is what drives the surveillance, airspace-congestion, and security risks (Section 6) — the two are not separable and any deployment strategy has to weigh them together rather than treating benefits and risks as independent tracks.

## 6. Risks and Challenges to Society

- **Privacy and surveillance concerns**: a drone that nearly everyone already carries in their pocket dramatically lowers the barrier to casual aerial surveillance, filming into windows, over fences, or of people without consent, at a scale far beyond today's dedicated consumer drones.
- **Airspace congestion and safety**: if adoption is high, low-altitude urban airspace could see far more small aircraft than today's drone population, increasing risk of collisions with people, property, other aircraft (including emergency response helicopters), and infrastructure.
- **Regulatory and enforcement gaps**: existing drone regulations (registration, remote ID, no-fly zones near airports/events) were not designed for a device that is also someone's primary phone; enforcement agencies would need new frameworks to distinguish "phone use" from "unauthorized flight."
- **Security exploitation**: a network-connected flying camera in mass-market hands is an attractive target for hacking, enabling unauthorized surveillance, stalking, or use as a physical intrusion/reconnaissance tool if compromised.
- **Weaponization potential**: any small, autonomous, camera-equipped flying platform carries dual-use risk; mass availability could lower the barrier for malicious actors to attach payloads or use swarms for harassment or attack, a concern already raised about consumer drones generally.
- **Noise and public nuisance**: widespread casual use in parks, neighborhoods, and events could create significant noise pollution and public annoyance beyond what dedicated hobbyist drones currently cause, since ownership friction would be far lower.
- **Accessibility of harm**: because it is disguised as an everyday object, bystanders and even law enforcement may have difficulty visually distinguishing "phone" from "armed/flying" states, complicating incident response.
- **Economic and labor disruption**: a ubiquitous personal aerial platform could disrupt existing commercial drone service industries (delivery, inspection, surveying) and raise new labor/safety questions for gig-economy operators.

---

## 7. Propeller and Propulsion Material Comparison

Small folding propellers and propulsion components must balance thrust efficiency, durability, noise, weight, and safety (since they will operate close to human hands and faces far more often than a dedicated drone's).

| Material | Strengths | Weaknesses | Best fit |
|---|---|---|---|
| **Injection-molded polycarbonate (PC) / PC-ABS blends** | Cheap, tough, good impact resistance, easy to mass-produce and color-match to phone housing | Lower stiffness than composites, can flex under load reducing efficiency, more prone to wear over many cycles | Cost-driven consumer version; acceptable for short-flight, low-payload use |
| **Nylon (often glass- or carbon-fiber reinforced)** | Good fatigue resistance, flexible enough to survive minor prop strikes without shattering, widely used in commercial mini-drones | Reinforced versions cost more, can absorb moisture over time affecting stiffness | Good middle-ground choice; likely the most practical default for folding propellers |
| **Carbon-fiber composite** | Excellent stiffness-to-weight ratio, very efficient blades, minimal flex/vibration | Expensive, brittle on hard impact (shatters rather than flexing), sharp edges when broken increase injury risk | Premium/performance variant; less ideal for a mass-market device that will be handled by non-expert users |
| **Thermoplastic elastomer (TPE) blade tips or full-soft-blade designs** | Much safer on contact with skin, quiet, forgiving in collisions | Lower efficiency, more flex/vibration reduces flight performance and battery efficiency | Strong candidate specifically because this device will be handled far more casually than a normal drone; a safety-first choice |
| **Aluminum or magnesium alloy (for motor housings/arms, not blades)** | High strength-to-weight for structural arms and motor mounts, good heat dissipation for motor housings | Heavier than composites/plastics, magnesium alloys can be less corrosion-resistant | Good for structural arms and motor housings where heat dissipation and rigidity matter more than minimal weight |
| **Titanium alloy (for hinge pins/locking mechanisms)** | Extremely high fatigue resistance, ideal for parts cycled thousands of times (fold/unfold joints) | Expensive, harder to machine at micro scale | Best reserved for the highest-fatigue components (hinge pins, locking latches) rather than the whole structure |

**Recommendation**: a hybrid material strategy is most realistic — reinforced nylon or TPE-tipped blades for user safety and durability, aluminum/magnesium for structural arms and motor housings, and titanium limited to high-fatigue hinge components. Full carbon-fiber, while highest-performing, is likely too brittle and hazardous for a device that ordinary consumers will handle casually like a phone.

---

## 8. Five Companies with Relevant Capabilities to Build Project VERION

This shortlist identifies companies whose existing capabilities map to major parts of the smartphone–intelligent drone hybrid: premium smartphone design and software, compact mobile compute, and autonomous flight systems. It is a **capability-based shortlist, not a claim that any company has committed to build VERION or that a partnership is available**. A phone-drone product would likely require substantial cross-company engineering.

### 8.1 Apple — Integrated Smartphone Product and User Experience

**Why it is relevant**
- Apple develops the iPhone, its operating system, custom silicon, camera systems, and tightly integrated hardware/software experiences.
- Its September 2026 introduction of a foldable iPhone demonstrates ongoing work on compact, transforming mobile-device form factors.

**Potential VERION contribution**
- Product definition, industrial design, enclosure integration, mobile operating system, privacy controls, camera pipeline, and on-device AI experience.
- A unified experience for switching between phone and flight modes, with clear safety prompts and user controls.

**Main gaps / challenges**
- Apple is not established here as a manufacturer of consumer autonomous quadrotors; flight hardware, flight-control safety, and aviation certification would require major new capability or a specialist partner.
- Integrating motors, propellers, battery discharge, and thermal loads into a premium phone would challenge thinness, durability, repairability, and safety.

**Evidence:** Apple’s September 9, 2026 announcement describes its foldable iPhone, custom A20 Pro silicon, thermal management, and Apple Intelligence. <Cite refs={["turn0search10"]} />

### 8.2 Samsung Electronics — Smartphone Manufacturing and Foldable Hardware

**Why it is relevant**
- Samsung has broad smartphone hardware and manufacturing capabilities, including foldable devices and a connected Galaxy ecosystem.
- Samsung collaborates with Qualcomm on mobile platforms and AI experiences across device categories.

**Potential VERION contribution**
- Smartphone mechanical integration, displays, battery packaging, mass manufacturing, and Android/Galaxy ecosystem integration.
- Prototyping a transforming chassis and coordinating mobile compute with specialized flight electronics.

**Main gaps / challenges**
- A folding phone is not equivalent to a flight-capable airframe; precision rotor mechanics, vibration isolation, and flight safety would need dedicated development.
- A hybrid would add unusual mechanical wear, exposed-rotor hazards, and certification burdens.

**Evidence:** Qualcomm’s July 22, 2026 release describes its collaboration with Samsung across Galaxy smartphones and other connected devices. <Cite refs={["turn0search0"]} />

### 8.3 Qualcomm — Mobile Compute, Connectivity, and On-Device AI

**Why it is relevant**
- Qualcomm supplies Snapdragon mobile platforms that combine processing, AI acceleration, and wireless connectivity for smartphone manufacturers.
- These capabilities align with VERION’s need for efficient on-device perception, mission planning, and communications.

**Potential VERION contribution**
- Mobile SoC/NPU platform, connectivity, power-efficient AI inference, and engineering support for an OEM integrating compute into a compact device.
- A potential platform partner rather than necessarily the consumer-facing product manufacturer.

**Main gaps / challenges**
- A mobile application processor is not a substitute for a deterministic, safety-critical flight controller, motor drivers, or validated flight-control software.
- Flight workloads could compete with phone tasks for battery power, memory bandwidth, and thermal headroom.

**Evidence:** Qualcomm’s May 7, 2026 release describes Snapdragon mobile platforms with AI-powered camera features, performance, power efficiency, and 5G/Wi-Fi connectivity. <Cite refs={["turn0search4"]} />

### 8.4 DJI — Consumer Drone Hardware and Aerial Imaging

**Why it is relevant**
- DJI is an established drone company with consumer and enterprise aircraft, camera systems, and supporting software.
- Its experience is directly relevant to compact propulsion, aerial imaging, flight-control integration, and consumer drone workflows.

**Potential VERION contribution**
- Drone subsystem expertise: rotor/motor design, flight-control integration, stabilization, camera gimbaling or stabilization, and compact aircraft packaging.
- Potential technical partner or reference point for a dedicated flight module.

**Main gaps / challenges**
- DJI’s established drone products are separate aircraft, not evidence of a smartphone-sized integrated flying phone.
- Regulatory and supply-chain constraints may affect feasibility for particular markets, especially the United States; legal status should be checked for the intended launch date and product category.

**Evidence:** Industry comparisons describe DJI’s broad consumer-to-enterprise drone ecosystem, while 2026 reporting discusses U.S. restrictions and market uncertainty. <Cite refs={["turn0search11","turn0news12"]} />

### 8.5 Skydio — Autonomous Flight and AI Perception

**Why it is relevant**
- Skydio develops AI-enabled autonomous drones and integrated software for applications such as inspection and public safety.
- Its autonomy, perception, and real-time navigation experience map to VERION’s obstacle avoidance and autonomous mission ambitions.

**Potential VERION contribution**
- Autonomy software, perception/navigation architecture, flight-safety lessons, and integration expertise.
- A potential autonomy technology partner or source of engineering know-how, rather than a demonstrated smartphone manufacturer.

**Main gaps / challenges**
- Skydio’s products are purpose-built drones; miniaturizing their relevant capabilities into a phone-sized, consumer-priced device would be a separate engineering program.
- Its current commercial emphasis includes enterprise, public safety, and defense, so a mass-market smartphone hybrid may not match its present product focus.

**Evidence:** Skydio describes its integrated drones, docks, autonomy platform, and software; its April 2026 announcement also outlined U.S. manufacturing and R&D expansion. <Cite refs={["turn0search3","turn0search1"]} />

### 8.6 Comparative Capability Matrix

| Company | Smartphone / mobile integration | Drone / flight expertise | Autonomy / AI | Plausible VERION role |
|---|---|---|---|---|
| Apple | Core strength | Not established as a core product area | On-device AI and ecosystem | Product owner, phone platform, UX |
| Samsung | Core strength | Not established as a core product area | Connected-device AI ecosystem | Phone hardware and manufacturing |
| Qualcomm | Mobile platform supplier | Not a drone manufacturer | Mobile AI acceleration | Compute and connectivity supplier |
| DJI | Companion apps, not phone OEM | Core strength | Drone automation and imaging | Drone subsystem specialist |
| Skydio | Not a phone OEM | Core strength | Core strength in autonomous flight | Autonomy and perception specialist |

### 8.7 Practical Partnership Hypothesis

A plausible *hypothesis* is a cross-disciplinary team rather than one company doing everything:

1. **Smartphone/product lead:** Apple or Samsung.
2. **Mobile compute and connectivity:** Qualcomm or the phone maker’s own silicon platform.
3. **Flight hardware and imaging:** a drone specialist such as DJI, subject to market-specific regulatory and supply-chain review.
4. **Autonomy and perception:** a specialist such as Skydio, or an independently developed flight stack.
5. **Independent safety engineering:** flight-control, battery, rotor-guard, cybersecurity, privacy, and certification specialists.

This is a conceptual division of responsibilities, not a proposed or confirmed partnership. The most important early validation would be a working flight-capable prototype that proves the combined mass, thrust, battery discharge, rotor safety, thermal limits, and controlled transformation can coexist in the intended phone envelope.

### 8.8 Research Caveat

Company capabilities and regulatory conditions change. The references above support the specific capabilities stated, but do not establish that any company has announced a VERION-like product, agreed to collaborate, or confirmed that the concept is commercially feasible. Recheck corporate announcements, supplier availability, and applicable aviation and radio rules before using this shortlist for outreach or investment decisions.

---

## Open Questions

**Carried over from v0.1:**
- Final choice of on-device NPU and its shared compute budget between flight control and LLM inference.
- Regulatory strategy for dual phone/aircraft certification across major markets.
- User-safety design for exposed propellers when handled in transition between modes.
- Battery chemistry and thermal strategy under combined flight + cellular load.

**New in v0.2:**
- What insurance and liability model would apply to casual/bystander-operated search-and-rescue use, given the device is not owned or trained as dedicated rescue equipment?
- How should municipal/urban airspace policy be updated in advance of mass adoption, rather than reactively after incidents occur?
- What technical or policy mechanisms (e.g., visible flight-mode indicators, mandatory remote ID broadcast) could reduce the "disguised as an everyday object" ambiguity raised in the societal risks section?
- How should the societal benefits (democratized access, accessibility gains, education) be weighed against surveillance-normalization risk when deciding default privacy settings and geofencing defaults out of the box?

---

*End of Version 0.31 draft.*
