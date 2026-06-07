# World Model GeoSim: Stress-Testing the 2030 World

## Why I built a geopolitical simulation engine — and why spreadsheets can't model complexity anymore

---

### Part I: What We're Not Modeling

In 2022, Russia invaded Ukraine. The world was surprised.

In 2023, Hamas attacked Israel. The world was surprised.

In 2024, the Houthis blocked the Red Sea. The world was surprised.

In 2025, the US-China trade war escalated into semiconductor embargoes, rare earth export controls, and financial decoupling. The world was surprised.

In 2026, we're watching a dozen potential flashpoints — Taiwan, the South China Sea, the Korean Peninsula, the Taiwan Strait, the Arctic, the Sahel, the Eastern Mediterranean, the South Caucasus, the Mekong Delta — and pretending we can predict which one will break first.

We can't. Not because we lack data. Because we're using the wrong tools.

The world is a **complex adaptive system**. It doesn't follow linear paths. It doesn't respect spreadsheet models. It doesn't care about your five-year plan.

It cascades. It tips. It emerges. It surprises.

I built World Model GeoSim because I got tired of watching smart people get blindsided by dynamics they should have seen coming — if they had been modeling the system instead of the headlines.

---

### Part II: The Architecture of GeoSim

World Model GeoSim is a **multi-agent Complex Adaptive Systems simulation engine** focused on global dynamics toward 2030.

Here's how it works.

#### The Actors

GeoSim models 40+ regions as **adaptive agents** — each with their own:
- **Resource endowments** (energy, minerals, water, arable land, manufacturing capacity)
- **Strategic objectives** (territorial, economic, ideological, survival)
- **Constraints** (demographic, fiscal, geographic, political stability)
- **Relationships** (alliances, dependencies, rivalries, historical grievances)
- **Decision rules** (risk tolerance, time horizon, reciprocity, threshold effects)

These aren't static profiles. Agents **adapt** their strategies based on what other agents do. A trade restriction by Agent A changes the resource calculus of Agent B, which changes its strategic posture toward Agent C, which cascades through the network.

#### The Domains

GeoSim models **seven interacting domains**:

| Domain | Key Variables | Cascading Effects |
|--------|--------------|-------------------|
| **Geopolitical** | Military posture, alliance commitments, territorial disputes, diplomatic relations | Conflict → sanctions → trade disruption → economic pressure → regime stability |
| **Economic** | Trade flows, supply chains, currency regimes, debt levels, inflation | Tariffs → manufacturing shifts → employment changes → political pressure |
| **Energy** | Fossil fuel reserves, renewable capacity, grid infrastructure, energy corridors | Disruption → price spikes → industrial contraction → social unrest |
| **Food & Water** | Agricultural output, water stress, fertilizer access, climate resilience | Drought → crop failure → price spikes → food riots → migration |
| **Climate** | Temperature trajectories, extreme events, sea level rise, carbon budgets | Warming → agricultural shifts → migration → resource competition → conflict |
| **Technology** | AI capability, semiconductor supply chains, cyber warfare, space assets | AI acceleration → labor displacement → inequality → political instability |
| **Demographic** | Aging populations, youth bulges, migration patterns, urbanization | Demographic pressure → labor shortages → automation → social transformation |

Each domain is a CAS on its own. GeoSim models their **interaction** — because that's where the real dynamics live.

#### The Engine

Under the hood, GeoSim uses:

**Evolutionary Game Theory**: Agents don't play one-shot games. They play repeated games, learn from outcomes, and evolve their strategies. A nation that gets punished for aggression becomes less aggressive — unless the payoff matrix changes (defensive necessity, domestic pressure, perceived weakness of the opponent).

**Network Diffusion Models**: Shocks propagate through networks. A port closure in the Strait of Malacca doesn't just affect shipping — it diffuses through trade networks, insurance markets, commodity futures, manufacturing supply chains, and employment in 40+ countries. GeoSim models the diffusion path and speed.

**Tipping Point Detection**: The system identifies where small changes produce large effects. A 1°C temperature increase in the Sahel doesn't matter much — until it pushes pastoralists into farmland, which triggers ethnic violence, which triggers state collapse, which triggers a migration wave, which triggers political realignment in North Africa and Southern Europe. That's a tipping point.

**DST (Dempster-Shafer Theory) Uncertainty**: GeoSim doesn't produce false certainty. Every output comes with a [Belief, Plausibility] interval — the range of outcomes consistent with the evidence. When uncertainty is structural (we genuinely don't know), it says so. No point estimates pretending to be predictions.

---

### Part III: The Scenarios That Keep Me Up at Night

Here are three scenarios GeoSim has stress-tested. I'm not predicting these. I'm saying: the model says these are **plausible trajectories** that most institutions aren't modeling.

#### Scenario 1: The Taiwan Strait Blockade (2027-2028)

**Trigger**: China imposes a blockade on Taiwan in response to what it perceives as creeping independence.

**Modeled cascade**:
- Week 1: Global semiconductor supply drops 60%. Taiwan produces 90% of advanced chips.
- Week 2: US and allies impose crippling sanctions on China. China retaliates with rare earth export controls.
- Week 3: Global auto production halts (no chips). Consumer electronics prices spike 40%.
- Month 2: Supply chain restructuring begins — but it takes 3-5 years to build alternative fab capacity.
- Month 3: Recession in every major economy. 10-15% unemployment in manufacturing-dependent regions.
- Year 1: Permanent deglobalization. Regional blocs form. The world economy fragments.

**Who wins?**: Nobody. The model shows net-negative outcomes for every major actor. The question is who loses least — and the answer depends on how quickly alternative supply chains can be built.

**Signals to watch**: Semiconductor inventory levels, rare earth stockpiling, naval force posture in the South China Sea, TSMC expansion timelines in Arizona and Germany.

#### Scenario 2: The Sahel Collapse Cascade (2026-2030)

**Trigger**: A multi-year drought in the Sahel, combined with desertification and population growth, triggers a cascading state collapse from Mali through Sudan.

**Modeled cascade**:
- Year 1: Agricultural output drops 40%. Food prices double. Urban populations riot.
- Year 2: Governments lose territorial control. Armed groups fill the vacuum. Mineral resources (uranium, gold, oil) become contested.
- Year 3: Migration pressure on North Africa — 5-10 million people moving north.
- Year 4: Southern Europe faces unprecedented migration flows. EU internal cohesion fractures over border policy.
- Year 5: The Mediterranean becomes a militarized zone. North African regimes destabilized by economic and demographic pressure.

**Who wins?**: Russia and China gain access to Sahelian mineral resources through security agreements with surviving regimes. Europe loses. The US is distracted by the Pacific.

**Signals to watch**: Rainfall patterns in the Sahel (CHIRPS data), food price indices in West African capitals, French military posture, Wagner/ Africa Corps deployments, EU border enforcement budgets.

#### Scenario 3: The AI Acceleration Trap (2026-2028)

**Trigger**: AI capability growth accelerates faster than institutional adaptation capacity.

**Modeled cascade**:
- Year 1: AI automation displaces 15-20% of white-collar knowledge work. Not unemployment — underemployment. People keep their jobs but do less. Wages stagnate.
- Year 2: AI-powered disinformation makes democratic consensus impossible. Every election is contested. Trust in institutions collapses below 20%.
- Year 3: AI-enabled cyber attacks cripple critical infrastructure in multiple countries simultaneously. Not a single attack — a distributed campaign.
- Year 4: Governments respond with digital authoritarianism. The open internet fragments into national firewalls. The "splinternet" becomes permanent.
- Year 5: AI development races ahead of any governance framework. The alignment problem remains unsolved. We're running an AGI-capable system with 2021-level safety research.

**Who wins?**: The most authoritarian regimes — they can deploy AI control faster. Democratic societies are paralyzed by their own openness.

**Signals to watch**: Frontier model capability benchmarks, AI safety funding as % of total AI investment, government AI regulation timelines, cyber attack frequency and sophistication, public trust surveys.

---

### Part IV: Why Institutions Get It Wrong

The world's most powerful institutions — intelligence agencies, defense departments, central banks, multinational corporations — all have modeling capabilities. Why do they keep getting surprised?

**1. They model domains, not systems.**

The CIA models geopolitical risk. The Fed models economic risk. The EPA models climate risk. None of them model the **interaction** between these domains. The 2008 financial crisis wasn't a banking crisis — it was a housing crisis that cascaded through banking, insurance, employment, consumption, and government finances. No single-domain model catches that.

**2. They optimize for precision, not robustness.**

Point estimates feel scientific. "There's a 30% chance of conflict in the Taiwan Strait by 2028." But that precision is false. The real answer is a range — and the range is wide. Institutions that plan for a range survive surprises. Institutions that plan for a point estimate don't.

**3. They ignore second-order effects.**

Every intervention produces second-order effects. Sanctions on Russia produced energy price spikes that hurt the sanctioning countries more than Russia. Interest rate hikes to control inflation triggered banking crises. Drone warfare reduced casualty costs for attackers, which lowered the threshold for initiating conflict. Institutions model the first-order effect. The second-order effect is where the surprise lives.

**4. They're captured by their own narratives.**

Intelligence agencies have institutional narratives that resist contradictory evidence. The "Russia is a gas station with nukes" narrative made Europe underestimate Russian industrial adaptation. The "China is an economic partner" narrative made the US underestimate Chinese strategic competition. Narratives are necessary for sensemaking — but they're also the primary source of blind spots.

GeoSim doesn't have narratives. It has agents, payoffs, networks, and feedback loops. It doesn't want to believe anything. It just models what happens.

---

### Part V: What GeoSim Actually Does

Here's the concrete output.

**Regional risk assessments:**
- Current stability score for 40+ regions (0-100)
- Trend direction (deteriorating, stable, improving)
- Primary risk drivers (which domains are pushing the region toward instability)
- Cascade vulnerability (if this region tips, which other regions get dragged in)

**Scenario stress tests:**
- "What happens if the Strait of Malacca is blocked for 90 days?"
- "What happens if the Greenland ice sheet passes a tipping point?"
- "What happens if the US and China fully decouple?"
- "What happens if AI capabilities double in 18 months?"

Each scenario produces: cascade map, affected regions, timeline estimates, confidence intervals, and actionable signals to monitor.

**Network analysis:**
- Which regions are systemically important (not just powerful — connected)
- Which relationships are stabilizing vs. destabilizing
- Where are the leverage points for intervention
- What are the second-order effects of a given policy

**Early warning indicators:**
- Specific, measurable signals to watch for each scenario
- Threshold values (when a signal crosses a threshold, the cascade probability jumps)
- Update frequency (daily, weekly, monthly depending on the signal)
- False positive history (how reliable each signal has been)

---

### Part VI: Why This Matters

We are entering a period of **structural uncertainty** that has no modern precedent.

The Cold War was dangerous, but it was simple: two superpowers, nuclear deterrence, clear alliances. The post-Cold War era was complex, but it was stable: US hegemony, expanding trade, global institutions. The current era is neither simple nor stable.

We have:
- **Multiple poles of power** (US, China, Russia, EU, India, regional powers)
- **Multiple domains of competition** (military, economic, technological, informational, environmental)
- **Accelerating technology** (AI, biotech, quantum, space — all advancing faster than governance)
- **Degrading institutions** (trust in governments, media, science, and international organizations is at historic lows)
- **Planetary boundaries** (climate, biodiversity, water, land use — all under unprecedented pressure)

This is not a world that can be managed with spreadsheet models and quarterly planning cycles.

This is a world that requires **scenario stress-testing**, **network modeling**, and **adaptive strategy** — the tools of complex systems science applied to global governance.

World Model GeoSim is a step in that direction.

It's built for decision-makers who can't afford to be wrong. National security professionals who need to see around corners. Corporate strategists whose supply chains span contested regions. Investors whose portfolios depend on stable geopolitical assumptions. Anyone who's realized that the old tools don't work for the new world.

The 20th century was linear. The 21st century is adaptive.

Model accordingly.

---

*World Model GeoSim is live: [worldmodel-geosim.vercel.app](https://worldmodel-geosim.vercel.app/)*

*Built by Désiré Yavro Ednzengou — founder, systems thinker, former nuclear physicist turned AI builder.*

*Follow the build: @desireyavro*
