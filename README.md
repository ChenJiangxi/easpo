# 24machine 参数说明

本仓库只包含 `24machine` 算例的环境参数文件，不包含训练脚本、实验对比说明或结果数据。

所有文件位于 `configs/env/24machine/`。

## 文件组成

| 文件 | 作用 |
| --- | --- |
| `params_24machine.csv` | 24 台机器的逐机参数表，定义寿命分布、工序归属、工况系数、产能占比、维护时间和维护成本。 |
| `scenario_24machine_default_config.json` | 基础场景配置。 |
| `scenario_24machine_moderate_config.json` | 中等约束场景，主要收紧备件初始库存、补货量和部分 PM 动作的备件消耗设定。 |
| `scenario_24machine_tight_config.json` | 紧约束场景，进一步收紧备件供给，并让更多 PM 动作消耗备件。 |
| `workforce_config_24machine_default.json` | 基础人力与调度配置。 |
| `workforce_config_24machine_moderate.json` | 中等约束人力配置。 |
| `workforce_config_24machine_tight.json` | 紧约束人力配置。 |

## 配置配对方式

通常按同名后缀配对使用：

| 场景文件 | 人力文件 |
| --- | --- |
| `scenario_24machine_default_config.json` | `workforce_config_24machine_default.json` |
| `scenario_24machine_moderate_config.json` | `workforce_config_24machine_moderate.json` |
| `scenario_24machine_tight_config.json` | `workforce_config_24machine_tight.json` |

## 建模对象概览

该算例描述一个 24 台机器、8 道工序的生产维护系统。

| 项目 | 含义 |
| --- | --- |
| 机器数量 | 24 台。 |
| 工序数量 | 8 道工序。 |
| 产品类型 | 当前仓库中的环境实现按单产品 A 运行。 |
| 备件类型 | 3 类，记为 `S1`、`S2`、`S3`。 |
| 维护动作 | 当前环境实现的有效动作集合为 `0`、`0.4`、`0.6`、`0.8`、`0.98`。其中 `0` 表示不维护，`0.98` 表示 CM，其余为不同强度的 PM。 |

当前代码中，机器状态核心为：

```text
state = [age, failure_indicator, relative_lifetime]
```

其中：

- `age` 是虚拟年龄。
- `failure_indicator` 为设备状态，`1` 表示正常，`0` 表示故障。
- `relative_lifetime = age / lifetime`。

## `params_24machine.csv` 字段说明

该文件每一行对应一台机器。

| 列名 | 含义 | 备注 |
| --- | --- | --- |
| `machine_name` | 机器名称。 | 例如 `Machine1`。 |
| `lifetime` | 标称寿命。 | 用于计算 `relative_lifetime = age / lifetime`。 |
| `shape` | Weibull 分布形状参数。 | 决定故障率随年龄变化的陡峭程度。 |
| `scale` | Weibull 分布尺度参数。 | 与 `shape` 一起决定基础故障风险。 |
| `process` | 机器所属工序编号。 | 当前算例共有 8 道工序。 |
| `bias` | 工况偏置系数。 | 进入故障率中的指数项，和 `condition` 一起影响风险强度。 |
| `A_condition` | 产品 A 下的工况系数。 | 当前环境实际使用。 |
| `B_condition` | 产品 B 下的工况系数。 | 当前单产品 A 环境中未实际使用，保留为扩展字段。 |
| `C_condition` | 产品 C 下的工况系数。 | 当前单产品 A 环境中未实际使用，保留为扩展字段。 |
| `A_cap` | 产品 A 下该机器承担的产能占比。 | 当前环境实际使用，也影响工序损失计算。 |
| `B_cap` | 产品 B 下的产能占比。 | 当前单产品 A 环境中未实际使用。 |
| `C_cap` | 产品 C 下的产能占比。 | 当前单产品 A 环境中未实际使用。 |
| `cm_time` | 纠正性维护 CM 的维护时长。 | 当前代码中 PM 时长按 `cm_time * action` 计算。 |
| `cm_cost` | CM 成本。 | 当前代码中 CM 成本按固定值计。 |
| `pm_cost` | PM 单位强度成本。 | 当前代码中 PM 成本按 `pm_cost * action` 计算。 |
| `z_thr` | 阈值字段。 | 在当前仓库检查到的环境实现中未直接引用，可视为保留字段。 |

### 机器故障与维护的使用方式

当前环境实现中，机器基础故障率按 Weibull 风险函数建模，核心形式可以概括为：

```text
h(t) = (shape / scale) * (t / scale)^(shape - 1) * exp(bias * condition)
```

在单产品 A 场景下，`condition` 和 `cap` 实际取自 `A_condition` 与 `A_cap`。

维护动作的含义如下：

| 动作值 | 含义 | 当前实现中的时间/成本解释 |
| --- | --- | --- |
| `0` | 不维护 | 时间和成本都为 0。 |
| `0.4` | 低强度 PM | 时长 `cm_time * 0.4`，成本 `pm_cost * 0.4`。 |
| `0.6` | 中等 PM | 时长 `cm_time * 0.6`，成本 `pm_cost * 0.6`。 |
| `0.8` | 高强度 PM | 时长 `cm_time * 0.8`，成本 `pm_cost * 0.8`。 |
| `0.98` | CM | 时长 `cm_time`，成本 `cm_cost`。 |

动作执行后，虚拟年龄更新规则为：

```text
PM: age <- (1 - action) * age
CM: age <- (1 - 0.98) * age
```

也就是说，CM 在当前实现里近似视为将虚拟年龄重置到 2%。

## `scenario_*.json` 字段说明

三份 `scenario` 文件描述的是环境级配置。

| 字段 | 含义 | 当前是否被环境运行逻辑直接使用 |
| --- | --- | --- |
| `scenario_name` | 场景名称。 | 否，主要用于标识。 |
| `description` | 场景文字说明。 | 否，主要用于标识。 |
| `machine_params_file` | 指向机器参数 CSV。 | 是。 |
| `environment_params` | 环境基本参数。 | 是。 |
| `process_to_devices` | 每道工序对应哪些机器。 | 是。 |
| `product_process_influence` | 上游工序对下游工序故障风险的影响结构。 | 是。 |
| `stage_setup_cost` | 按工序计的维护 setup 成本。 | 是。 |
| `maintenance_level` | 维护等级列表。 | 当前环境代码未直接读取，仅作配置性说明。 |
| `initial_spare_parts` | 每类备件的初始库存采样规则。 | 是。 |
| `maintenance_cost_matrix` | 机器在不同维护等级下的成本矩阵。 | 当前主环境实现未直接读取，该信息与 `cm_cost`、`pm_cost` 有部分重叠，更像附加说明或兼容字段。 |
| `spare_part_consumption` | 机器在不同维护动作下的备件消耗规则。 | 是。 |
| `spare_part_consumption_matrix` | 上述消耗规则的矩阵形式。 | 当前主环境实现未直接读取。 |
| `spare_parts_replenishment` | 周期性补货规则。 | 是。 |
| `reward_params` | 环境奖励与惩罚参数。 | 是。 |
| `model_params` | 模型结构超参数。 | 当前仓库中检查到的环境运行代码未直接读取。 |
| `graph_structures` | 图结构或超图结构元数据。 | 当前仓库中检查到的环境运行代码未直接读取。 |

### `environment_params`

| 字段 | 含义 |
| --- | --- |
| `num_batches` | 一个 episode 中的生产批次数。 |
| `tau` | 单个生产周期长度。 |
| `delta_tau` | 单个维护窗口长度。 |
| `product_price` | 单位产出价格，用于工序损失计算。 |

### `product_process_influence`

该字段定义工序间的风险传递关系。当前仓库的 `Machine` 实现会把上游设备的故障风险按权重叠加到下游设备的复合风险中。

结构可理解为：

```text
product -> downstream_process -> upstream_process -> influence_weight
```

当前 release 只针对产品 A 给出配置，且当前单产品环境也只使用产品 A。

### `initial_spare_parts`

每种备件支持按分布采样初始库存。当前文件里使用的是离散均匀分布：

```text
{"distribution": "uniform", "min": x, "max": y}
```

环境重置时，会在 `[min, max]` 区间内重新采样初始库存。

### `spare_part_consumption`

该字段定义某台机器在某个维护动作下需要消耗哪些备件。

例如：

```json
"Machine1": {
  "0.8": {"S1": 1},
  "0.98": {"S1": 1, "S2": 1}
}
```

含义是：

- `Machine1` 执行 `0.8` 强度维护时，需要 1 个 `S1`。
- `Machine1` 执行 `0.98` CM 时，需要 1 个 `S1` 和 1 个 `S2`。

### `spare_parts_replenishment`

| 字段 | 含义 |
| --- | --- |
| `interval` | 补货周期。每经过这么多个 step 触发一次补货。 |
| `quantity` | 每次补货数量。 |
| `max_stock` | 库存上限。补货后不会超过该值。 |

### `reward_params`

| 字段 | 含义 |
| --- | --- |
| `spare_part_penalty` | 计划维护因备件不足而不可执行时的惩罚系数。 |
| `infeasible_penalty` | 不可行动作或不可执行维护的惩罚系数。 |
| `failure_penalty` | 设备故障数量相关惩罚系数。 |
| `maintenance_reward_coeff` | 维护收益相关系数。 |
| `age_penalty_coeff` | 年龄惩罚系数。 |
| `health_bonus_coeff` | 健康度奖励系数。 |

### 关于 `maintenance_level`

`scenario` 文件中写的是：

```text
[0.2, 0.4, 0.6, 0.8, 0.98]
```

但当前仓库里实际执行环境的动作集合是：

```text
[0, 0.4, 0.6, 0.8, 0.98]
```

也就是说：

- `0` 是当前环境真实支持的“不维护”动作。
- `0.2` 在当前环境代码中没有被直接当作有效动作使用。

## 三个 `scenario` 版本的差异

### `default`

- 初始备件库存相对宽松。
- 补货量最高。
- 只有部分机器在 `0.8` 和 `0.98` 下消耗备件。
- 额外包含 `model_params` 与 `graph_structures` 两组附加元数据。

### `moderate`

- 初始备件库存下调。
- 每次补货数量下调。
- 更多机器在 `0.6` 和 `0.8` PM 下开始消耗备件。
- 不再携带 `model_params` 与 `graph_structures` 字段。

### `tight`

- 初始备件库存进一步下调到更紧的区间。
- 每次补货数量进一步减少。
- 许多机器在 `0.4`、`0.6`、`0.8` PM 下都需要备件。
- 属于备件约束最紧的一版。

## `workforce_config_*.json` 字段说明

该类文件定义人力资源和调度求解配置。

### `workforce_configs`

`workforce_configs` 下可以包含多套可选配置，每一套通常描述一种人力结构。

| 字段 | 含义 |
| --- | --- |
| `description` | 人力配置说明。 |
| `num_workers` | 工人数 `H`。 |
| `num_skills` | 技能数 `Q`。 |
| `worker_skill_matrix` | 工人-技能矩阵，形状为 `H x Q`。元素为 1 表示工人具备该技能。 |
| `worker_efficiency` | 工人处理某类技能任务的时长倍率，键为 `(worker_id, skill_id)`。倍率越小表示越快。 |
| `action_skill_mapping` | 维护动作到主技能编号的映射。 |
| `machine_skill_mapping` | 工序到技能的映射。 | 
| `hybrid_skill_mapping` | 按“机器名 + 动作”细化后的技能映射。 |
| `skill_mapping_strategy` | 技能映射策略，可取 `action_only`、`machine_only`、`hybrid`。 |
| `scheduler_type` | 调度器类型，当前文件使用 `ortools`。 |
| `scheduling_method` | 调度方法，当前文件使用 `cp_sat`。 |
| `milp_time_limit` | 求解时间限制。 |
| `random_absence` | 是否模拟随机缺勤。 |
| `absence_probability` | 工人缺勤概率。 |

### 人力配置字段的实际解释

| 字段 | 解释 |
| --- | --- |
| `worker_skill_matrix` | 第 `h` 行对应工人 `h`，第 `q` 列对应技能 `q`。 |
| `worker_efficiency` | 实际处理时长按 `base_duration * efficiency` 计算，因此 `0.8` 表示比基准更快，`1.0` 表示基准时长。 |
| `action_skill_mapping` | 当前代码默认把 `0.98` 映射为技能 `0`，`0.8/0.6` 映射为技能 `1`，`0.4` 映射为技能 `2`。 |
| `machine_skill_mapping` | 只有在 `skill_mapping_strategy = machine_only` 或 hybrid 回退逻辑中才会用到。 |
| `hybrid_skill_mapping` | 只有在 `skill_mapping_strategy = hybrid` 时启用，用于覆盖“同一动作在不同机器上所需技能不同”的情况。 |

### `reward_params`

`workforce_config` 里的 `reward_params` 与 `scenario` 里的 `reward_params` 不是同一层参数。

这里的人力奖励参数主要服务于带人力约束的环境：

| 字段 | 含义 |
| --- | --- |
| `lambda_parts` | 备件不足惩罚系数。 |
| `mu_workforce` | 人力不可行惩罚系数。 |
| `nu_spillover` | CM 任务超出维护窗口的惩罚系数。 |
| `emergency_cost` | CM 因缺件而发生紧急采购时的成本系数。 |

## 三个 `workforce` 版本的差异

### `workforce_config_24machine_default.json`

- 主配置为 `ortools_default`。
- `num_workers = 4`。
- 使用 `skill_mapping_strategy = hybrid`。
- 提供 `machine_skill_mapping` 与 `hybrid_skill_mapping`，适合更细的技能建模。
- 同时保留多套 `reward_params` 方案，便于切换。

### `workforce_config_24machine_moderate.json`

- 新增主配置 `moderate_constraint`。
- `num_workers = 3`，比默认少 1 人。
- 仍保留较细的 `hybrid_skill_mapping`。
- 同时保留一个 `ortools_default` 配置作为兼容备用。
- `reward_params` 只保留一套 `default`。

### `workforce_config_24machine_tight.json`

- 新增主配置 `tight_constraint`。
- `num_workers = 2`，人力约束最强。
- 两名工人都覆盖全部技能，但效率不同。
- 未设置 `skill_mapping_strategy`，因此当前代码会走默认的 `action_only` 技能映射。
- 同时保留一个 `ortools_default` 配置作为兼容备用。

## 当前 release 中哪些字段是“实际生效”的

如果直接对应当前仓库中检查到的环境实现，可以这样理解：

| 字段类别 | 当前状态 |
| --- | --- |
| `params_24machine.csv` 中的 `shape`、`scale`、`bias`、`A_condition`、`A_cap`、`cm_time`、`cm_cost`、`pm_cost`、`process`、`lifetime` | 直接参与环境运行。 |
| `B_condition`、`C_condition`、`B_cap`、`C_cap` | 当前单产品 A 环境中保留但未启用。 |
| `z_thr` | 当前仓库中未发现直接运行引用。 |
| `scenario` 中的 `environment_params`、`process_to_devices`、`product_process_influence`、`stage_setup_cost`、`initial_spare_parts`、`spare_part_consumption`、`spare_parts_replenishment`、`reward_params` | 直接参与环境运行。 |
| `maintenance_cost_matrix`、`spare_part_consumption_matrix` | 当前主环境实现未直接读取。 |
| `model_params`、`graph_structures` | 当前主环境实现未直接读取，更适合作为附加元数据。 |
| `workforce_config` 中的 `num_workers`、`num_skills`、`worker_skill_matrix`、`worker_efficiency`、`action_skill_mapping`、`scheduler_type`、`scheduling_method`、`milp_time_limit` | 直接参与人力调度。 |
| `machine_skill_mapping`、`hybrid_skill_mapping`、`skill_mapping_strategy` | 仅在对应策略启用时生效。 |

## 使用时的建议

- 如果需要最完整的说明性元数据，可参考 `default` 场景文件。
- 如果需要体现备件约束逐步收紧，优先比较 `default -> moderate -> tight`。
- 如果需要体现人力瓶颈逐步收紧，优先比较 `workforce default -> moderate_constraint -> tight_constraint`。
- 如果只是复现实验环境，请确保场景文件与人力文件使用同一后缀版本。
