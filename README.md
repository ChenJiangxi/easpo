# 24machine Case Parameters

This repository provides the configuration files for a 24-machine production and maintenance case.

All files are located in `configs/env/24machine/`.

## Files

- `params_24machine.csv`
  Per-machine parameters, including lifetime distribution parameters, process assignment, condition-related coefficients, maintenance time, and maintenance cost values.

- `scenario_24machine_default_config.json`
  Baseline scenario configuration for the 24-machine case.

- `scenario_24machine_moderate_config.json`
  Scenario configuration with moderately tighter resource settings.

- `scenario_24machine_tight_config.json`
  Scenario configuration with tighter operating constraints.

- `workforce_config_24machine_default.json`
  Workforce and scheduler configuration corresponding to the default setting.

- `workforce_config_24machine_moderate.json`
  Workforce and scheduler configuration corresponding to the moderate setting.

- `workforce_config_24machine_tight.json`
  Workforce and scheduler configuration corresponding to the tight setting.

## What Is Defined In The Scenario Files

The scenario configuration files define the environment-level structure of the case, including:

- basic environment parameters such as planning horizon and batch settings
- mapping from production processes to machines
- production influence relationships
- spare-parts initialization and replenishment settings
- maintenance cost matrices and reward-related parameters
- graph structures used to describe resource, economic, and degradation relationships

Each scenario file refers to `params_24machine.csv` through the `machine_params_file` field.

## What Is Defined In The Workforce Files

The workforce configuration files define:

- available workers and skill coverage
- worker efficiency settings
- mappings between maintenance actions, machines, and required skills
- scheduler-related options
- workforce-related reward parameters

## Usage

Use one scenario file together with the workforce file of the same suffix:

- `default` with `default`
- `moderate` with `moderate`
- `tight` with `tight`
