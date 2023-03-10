# Multi Objective Optimization on OpenFOAM cases

Relying on [ax-platform](https://ax.dev) to experiment around 0-code parameter variation and multi-objective optimization
of OpenFOAM cases.

## Objectives and features
- Parameter values are fetched through a YAML/JSON configuration file. Absolutely no code should be needed, add parameters
  to the YAML file and they should be picked up automatically
- The no-code thing is taken to the extreme, through a YAML config file, you can (need-to):
  - Specify the template case
  - Specify how the case is ran
  - Specify how/where parameters are substituted
  - Specify how your metrics are computed

## The scripts

- `paramVariation.py` runs a Parameter Variation Study with the provided config and outputs trial data as a CSV file.
  Trial parameters are generated with SOBOL.
- `multiObjOpt.py` runs a multi-objective optimization study (can use the same config file) and captures a JSON snapshot
  of the experiment. You can also run it with a single objective.
- `postProcessExp.py` post-processes the experience generated by `multiObjOpt.py` to produce Pareto Fronts and feature
  importance.

## How do I use this?

The only requirement of usage (aside from being able to install dependency software) is that your template case needs to
be **ready to be parameterized**.

```bash
# Clone the repository
git clone https://github.com/FoamScience/OpenFOAM-Multi-Objective-Optimization multiOptFoam
cd multiOptFoam
# Install dependencies
pip3 install -r requirements.txt
# Copy your case
cp -r /path/to/cavity/case template
# Grab a config file (or create one yourself, see below)
cp /path/to/config.yaml .
# Run parameter variation
./paramVariation.py
```

## Sample configuration
```yaml
problem:
  # Problem name to prefix output files with
  name: Example
  # The base OpenFOAM case; this needs to be fully functional case one paramters/files are substituted.
  template_case: 'template'
  # Experiment paramters
  parameters:
    Sv:
      type: range
      value_type: float
      bounds: [1e-5, 1e-3]
      log_scale: True
    maxY:
      type: choice
      value_type: float
      values: [0.02, 0.04, 0.06, 0.08]
      is_ordered: True
    numberOfProcesses:
      type: range
      value_type: int
      bounds: [4, 8]
      log_scale: False
  # Experiment objectives. Metric values are fetched through shell commands in the working directory of
  # the specific trial
  # You need at least one metric
  objectives:
    MaxError:
      command: ['awk', '/^Max/{print($6+0)}', 'log.pvpython']
      threshold: 1e-6
      minimize: True

meta:
  # When clonging template case, specify extra files/dirs to clone
  case_subdirs_to_clone: ["0.orig", "dynamicCode", "error.py"]
  # How should we run your case? also specify a timeout in case it gets stuck
  # This should command should be blocking for now, and delivering metric values
  # which are then extracted through problem.objectives.*.command
  case_run_command: ['./Allrun']
  case_command_timeout: 1000
  # Number of trials to generate:
  # - Using SOBOL for paramter variation
  # - The Model is automatically chosen for optimization studies
  n_trials: 7
  # Number of pareto front points to generate
  n_pareto_points: 20
  # Paramters can be substitued as whole case files
  # These are done first if present
  # The following will copy transportProperties.concreteModel to transportProperties
  # where concreteModel is a generated value of modelType parameter
  #file_copies:
  #modelType:
  #  template: "/constant/transportProperties"
  # Parameters can also be substituted per-file
  scopes:
    "/system/fvOptions":
      Sv: "codedSource.Sv"
      maxY: "codedSource.maxY"
    "/system/decomposeParDict":
      numberOfProcesses: "numberOfSubdomains"

# Evaluate how the optimization algorithm did on a test set of parameters
# (Needed only if running an optimization)
verify:
  var1:
    - Sv: 1e-4
    - maxY: 0.07
    - numberOfProcesses: 4
```

## Contribution is welcome!

By either filing issues or opening pull requests, you can contribute to the development
of this project, which I would appreciate.
