# SkillOpt-EGT-RL: Evolutionary Architecture Blueprint

**Skill document = genome. Training loop = evolutionary pressure + reinforcement signal. Optimizer = mutation operator + policy network. Validation gate = selection mechanism.**

---

## 1. Core Thesis

Original SkillOpt treats skill as trainable state with SGD-like discipline. Evolved version treats skill as **evolutionary strategy** in population game, optimizer as **RL policy** that learns edit distributions, validation as **selection pressure** with Q-learned thresholds.

Three forces drive evolution:
- **EGT** (Evolutionary Game Theory): Population of skills compete, replicate, mutate, specialize into niches. ESS detection prevents overfitting.
- **GRPO** (Group Relative Policy Optimization): Optimizer trains via Group Relative Policy Optimization on edit proposals. No longer static prompt—learns what mutations work.
- **Q-Gate** (Q-Learning): Validation gate becomes soft adaptive threshold via Q-learning, eliminating hard-gate brittleness on small splits.

---

## 2. Architecture Layers

### Layer 0: Base SkillOpt (Frozen Substrate)
Keep original loop intact as execution substrate. All new layers wrap around, not replace. Backward compatibility: `--mode classic` runs vanilla SkillOpt.

### Layer 1: Population EGT (Pop-EGT)

**State**: Maintain population `P = {s_1, ..., s_N}` of skill genomes, not single `best_skill.md`.

**Fitness**: `f_i = accuracy(s_i, val_split) - λ·len(s_i) - μ·cost(s_i)`. Multi-objective: accuracy, compression, inference cost.

**Replicator Dynamics**: Each epoch, population evolves via:
```
x_i(t+1) = x_i(t) · f_i / φ(t)
```
where `φ(t) = Σ x_j·f_j` mean fitness, `x_i` = proportion of rollouts allocated to skill i.

**Operators**:
- **Mutation**: Standard SkillOpt edit (add/delete/replace) with rate `α(t)` = textual learning rate with cosine decay.
- **Crossover / Splice**: Merge two high-fitness skills. Optimizer takes skill A + skill B + task cluster, produces child skill C inheriting best sections. Lineage tracked in DAG.
- **Niche Partitioning**: Cluster tasks by embedding similarity. Skills specialize: `s_i` dominates niche `k` if `f_i^k > f_j^k ∀j`. Prevents single skill from overfitting average case.
- **ESS Detector**: Test if dominant skill resists invasion by small mutations. If yes, population converged—trigger early stopping or meta-update.

**Population Size**: `N = 8` default (paper-aligned compute budget). Scale to `N = 32` with K8s parallel rollout fleet.

### Layer 2: GRPO Optimizer (GRPO-Opt)

Replace static optimizer prompt with trainable policy `π_θ(edit | context)`.

**Group Relative Policy Optimization**:
1. **Sample**: For each reflection minibatch, generate `G = 4` candidate edit groups from current policy.
2. **Rollout**: Evaluate all G variants in parallel on selection split.
3. **Score**: Each group gets reward `R_g = Δaccuracy - λ·edit_size`.
4. **Relative Advantage**: `A_g = R_g - mean(R)`. No critic needed—relative within group.
5. **Update**: 
   ```
   ∇J = E[ (π_θ/π_old) · A_g · clip(π_θ/π_old, 1-ε, 1+ε) ]
   ```
6. **KL Penalty**: Keep optimizer close to base model to prevent mode collapse.

**Policy Architecture**: Same LLM backend, but weights updated via GRPO on skill-edit trajectories. Trajectory = (failure_context, proposed_edit, rollout_result). Freeze target model, train optimizer.

**Reward Shaping**:
- `+10` for validation gate pass
- `+5` for transfer improvement (cross-benchmark)
- `-1` per token of edit (compression pressure)
- `-20` for skill that breaks existing capabilities (catastrophic forgetting guard)

### Layer 3: Q-Gate (Q-Gate)

Hard gate rejects too many candidates on small splits. Replace with Q-learned adaptive gate.

**MDP Formulation**:
- **State** `s`: (current_skill_fingerprint, candidate_skill_fingerprint, task_distribution_entropy, epoch)
- **Action** `a`: {hard_accept, soft_accept, reject, partial_merge}
- **Reward** `r`: Accuracy trend over next 3 steps (long-term, not immediate)
- **Q-function**: `Q(s,a)` stored in table or small MLP. Updated via:
  ```
  Q(s,a) ← Q(s,a) + α[r + γ·max_a' Q(s',a') - Q(s,a)]
  ```

**Gate Decision**: Accept if `Q(s, accept) > Q(s, reject) + threshold`. Threshold anneals from 0.5 to 0.1 over epochs.

**Exploration**: ε-greedy gate early, greedy late. Ensures population diversity.

### Layer 4: Meta-Game EGT (Meta-EGT)

Multiple optimizer instances compete on same benchmark pool.

**Payoff Matrix**: `M[i,j] = accuracy(optimizer_i, benchmark_j)`. Optimizers replicate based on aggregate payoff.

**Purpose**: 
- Discover robust optimization strategies vs. benchmark-specific hacks
- Prevent optimizer overfitting to single task distribution
- Meta-optimizer selects which optimizer policy to deploy

**Replicator Equation at Meta-Level**:
```
dp_i/dt = p_i( Σ_j M_{ij} q_j - Σ_{k,l} p_k M_{kl} q_l )
```
where `p` = optimizer frequencies, `q` = benchmark frequencies.

### Layer 5: Skill Ecology (Eco-Skill)

Skills interact beyond competition.

**Symbiosis Detection**: Skill A boosts Skill B's fitness on transfer tasks. If `f_{A+B} > f_A + f_B`, form symbiotic pair. Deploy as ensemble.

**Parasitism Guard**: Detect skill that exploits benchmark artifacts (e.g., answer leakage in context). Fitness drops on held-out adversarial split → purge from population.

**Phylodendrogram**: Track lineage DAG. Identify "founder effects"—all good skills descend from one lucky early mutation. If yes, inject fresh random seed skills to maintain diversity.

---

## 3. Data Structures & APIs

### SkillGenome
```rust
struct SkillGenome {
    id: Uuid,
    document: String,           // skill.md content
    fitness: f64,               // current validation score
    niche: ClusterId,           // task specialization
    lineage: Vec<<EditId>,       // ancestry
    age: u32,                   // epochs survived
    mutation_rate: f64,         // individual learning rate
}
```

### PopulationState
```rust
struct PopulationState {
    genomes: Vec<<SkillGenome>,
    generation: u32,
    diversity_score: f64,       // average pairwise edit distance
    ess_detected: Option<<Uuid>,
}
```

### EditAction (RL Action Space)
```rust
enum EditAction {
    Add { section: String, content: String },
    Delete { section: String },
    Replace { section: String, content: String },
    Splice { parent_a: Uuid, parent_b: Uuid, crossover_points: Vec<<usize> },
    Meta { instruction: String }, // slow/meta update
}
```

### QGate
```rust
struct QGate {
    q_table: HashMap<(StateFingerprint, GateAction), f64>,
    epsilon: f64,
    alpha: f64,  // learning rate
    gamma: f64,  // discount
}
```

---

## 4. Training Loop: Evolved Protocol

```python
# Pseudocode for evolved training loop
for epoch in range(num_epochs):
    # 1. POPULATION ROLLOUT
    for skill in population:
        rollouts = parallel_rollout(skill, batch_size // pop_size)
        skill.fitness = evaluate(rollouts, gate_metric='multi_objective')
    
    # 2. REPLICATOR DYNAMICS
    population = replicator_update(population)  # fitness-proportionate
    
    # 3. GRPO OPTIMIZER UPDATE
    for minibatch in reflection_batches:
        groups = [sample_optimizer_policy(minibatch) for _ in range(G)]
        rewards = [evaluate_group(g) for g in groups]
        advantages = [r - mean(rewards) for r in rewards]
        update_optimizer_policy(groups, advantages)  # GRPO step
    
    # 4. MUTATION & CROSSOVER
    offspring = []
    for _ in range(population.target_size):
        if random() < crossover_rate:
            parents = tournament_select(population, k=3)
            child = splice_skills(parents[0], parents[1], optimizer)
        else:
            parent = roulette_select(population)
            child = mutate_skill(parent, optimizer, lr=alpha(epoch))
        offspring.append(child)
    
    # 5. Q-GATE VALIDATION
    for child in offspring:
        state = fingerprint(child, parent)
        action = q_gate.select_action(state)  # ε-greedy
        if action == ACCEPT:
            population.insert(child)
        elif action == PARTIAL_MERGE:
            merged = merge_skills(parent, child, optimizer)
            population.insert(merged)
        # else: reject, store in replay buffer
    
    # 6. NICHE DETECTION
    clusters = task_clustering(current_batch)
    assign_niches(population, clusters)
    
    # 7. ESS & DIVERSITY CHECK
    if detect_ess(population):
        trigger_early_stop_or_meta_update()
    if diversity_score < 0.1:
        inject_random_skills(n=2)
    
    # 8. SLOW / META UPDATE (EGT-enhanced)
    meta_skill = generate_meta_skill(population)  # consensus across niches
    meta_skill = q_gate.evaluate(meta_skill)      # adaptive gate
    if accepted:
        inject_meta_into_all(population)
```

---

## 5. Integration with Existing SkillOpt

### New CLI Arguments
| Flag | Default | Description |
|---|---|---|
| `--mode` | `classic` | `classic` / `pop` / `grpo` / `meta` / `full` |
| `--pop_size` | `8` | Population size |
| `--grpo_groups` | `4` | Group size for GRPO |
| `--crossover_rate` | `0.3` | Probability of splice vs mutation |
| `--q_gate` | `false` | Enable Q-learning gate |
| `--ess_threshold` | `0.95` | Fitness ratio for ESS detection |
| `--niche_clusters` | `3` | Number of task niches |

### Backward Compatibility
- `configs/_base_/default.yaml` unchanged
- New file: `configs/_base_/evolved.yaml` inherits + overrides
- All original backends, benchmarks, harnesses work unchanged
- `scripts/train.py` dispatches to `EvolvedTrainer` if `--mode != classic`

---

## 6. Production Infrastructure

### Parallel Rollout Fleet
Population evaluation = embarrassingly parallel. K8s Job per skill genome:
```yaml
# k8s/evolved-rollout.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: skillopt-rollout-{genome_id}
spec:
  parallelism: pop_size
  template:
    spec:
      containers:
      - name: rollout
        image: skillopt:latest
        command: ["python", "-m", "skillopt.evolved.rollout_worker"]
```

### Lineage Storage
DAG stored in SQLite or Redis:
- Node = skill genome
- Edge = edit operation + GRPO advantage
- Query: "Show all ancestors of best skill" for auditability

### Cost-Aware Scheduling
Fitness-weighted compute: high-fitness skills get larger rollout batches. Low-fitness skills get minimal evaluation (early stopping). Prevents waste.

---

## 7. Evaluation Protocol

### Baselines
1. Original SkillOpt (single skill, static optimizer, hard gate)
2. Pop-EGT only
3. GRPO-Opt only
4. Q-Gate only
5. Full evolved stack

### Metrics
- **Convergence**: Epochs to 90% of final accuracy
- **Absolute**: Final accuracy on 6 benchmarks × 7 models
- **Robustness**: Transfer accuracy to nearby benchmarks without re-optimization
- **Diversity**: Population pairwise edit distance (prevents collapse)
- **Efficiency**: Total tokens consumed / accuracy point gained
- **ESS Stability**: Variance across 5 runs with different seeds

### Expected Gains
- +5-8 points over original SkillOpt from population exploration
- +3-5 points from GRPO optimizer learning edit priors
- -30% token waste from Q-gate reducing false rejects
- Transfer improvement: skills evolved with EGT show higher cross-benchmark generalization (niche specialization → modular components)

---

## 8. Citation (Evolved)

```bibtex
@misc{skillopt_egt_rl_2026,
  title={SkillOpt-EGT-RL: Evolutionary Game Theory and Reinforcement Learning for Self-Evolving Agent Skills},
  author={[dnzengou] and SkillOpt Contributors},
  year={2026},
  note={Evolutionary extension of SkillOpt (Yang et al., 2026) integrating population dynamics, GRPO, and Q-learning},
  url={https://github.com/dnzengou/SkillOpt-EGT-RL}
}
```

---

## 9. Implementation Priority

**Phase 1 (Week 1-2)**: Pop-EGT layer. Population + replicator + crossover. Lowest risk, highest immediate gain.

**Phase 2 (Week 3-4)**: Q-Gate. Replace hard gate with Q-table. Fixes small-split brittleness.

**Phase 3 (Week 5-6)**: GRPO-Opt. Requires optimizer policy training infrastructure. Highest complexity.

**Phase 4 (Week 7-8)**: Meta-EGT + Eco-Skill. Multi-optimizer competition + symbiosis detection.

**Phase 5 (Week 9+)**: Integration hardening, K8s fleet, lineage UI, benchmark sweep.
