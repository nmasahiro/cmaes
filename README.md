# CMA-ES

Lightweight Covariance Matrix Adaptation Evolution Strategy (CMA-ES) [1] implementation.

![visualize-six-hump-camel](https://user-images.githubusercontent.com/5564044/73486622-db5cff00-43e8-11ea-98fb-8246dbacab6d.gif)

<details>
<summary>Himmelblau function.</summary>

![visualize-himmelblau](https://user-images.githubusercontent.com/5564044/73486618-dac46880-43e8-11ea-8a2e-69d745f008b5.gif)

</details>

<details>
<summary>Rosenbrock function.</summary>

![visualize-rosenbrock](https://user-images.githubusercontent.com/5564044/73486620-dac46880-43e8-11ea-9295-ec0bfa774655.gif)

</details>

<details>
<summary>Quadratic function.</summary>

![visualize-quadratic](https://user-images.githubusercontent.com/5564044/73486619-dac46880-43e8-11ea-859d-5f8358ac8be9.gif)

</details>

These GIF animations are generated by [visualizer.py](./visualizer/visualizer.py).


## Installation

Supported Python versions are 3.5 or later.

```
$ pip install cmaes
```

Or you can install via [conda-forge](https://anaconda.org/conda-forge/cmaes).

```
$ conda install -c conda-forge cmaes
```

## Usage

This library provides two interfaces that an Optuna's sampler interface and a low-level interface.
I recommend you to use this library via Optuna.

### Optuna's sampler interface

[Optuna](https://github.com/optuna/optuna) [2] is an automatic hyperparameter optimization framework.
A sampler based on this library is available from [Optuna v1.3.0](https://github.com/optuna/optuna/releases/tag/v1.3.0).
Usage is like this:

```python
import optuna

def objective(trial: optuna.Trial):
    x1 = trial.suggest_uniform("x1", -4, 4)
    x2 = trial.suggest_uniform("x2", -4, 4)
    return (x1 - 3) ** 2 + (10 * (x2 + 2)) ** 2

if __name__ == "__main__":
    sampler = optuna.samplers.CmaEsSampler()
    study = optuna.create_study(sampler=sampler)
    study.optimize(objective, n_trials=250)
```

See [the documentation](https://optuna.readthedocs.io/en/stable/reference/samplers.html#optuna.samplers.CmaEsSampler) for more details.

<details>

<summary>Monkeypatch for faster CMA-ES sampler of Optuna v1.3.x.</summary>

If you are using Optuna v1.3.x, you can make `optuna.samplers.CmaEsSampler` faster.

```python
import optuna
from cmaes.monkeypatch import patch_fast_intersection_search_space

patch_fast_intersection_search_space()

def objective(trial: optuna.Trial):
    x1 = trial.suggest_float("x1", -4, 4)
    x2 = trial.suggest_float("x2", -4, 4)
    return (x1 - 3) ** 2 + (10 * (x2 + 2)) ** 2

if __name__ == "__main__":
    sampler = optuna.samplers.CmaEsSampler()
    study = optuna.create_study(sampler=sampler)
    study.optimize(objective, n_trials=250)
```

</details>

<details>

<summary>For older versions (Optuna v1.2.0 or older)</summary>

If you are using older versions, please use `cmaes.samlper.CMASampler`.

```python
import optuna
from cmaes.sampler import CMASampler

def objective(trial: optuna.Trial):
    x1 = trial.suggest_uniform("x1", -4, 4)
    x2 = trial.suggest_uniform("x2", -4, 4)
    return (x1 - 3) ** 2 + (10 * (x2 + 2)) ** 2

if __name__ == "__main__":
    sampler = CMASampler()
    study = optuna.create_study(sampler=sampler)
    study.optimize(objective, n_trials=250)
```

</details>

Note that CmaEsSampler doesn't support categorical distributions.
If your search space contains a categorical distribution, please use [TPESampler](https://optuna.readthedocs.io/en/latest/reference/samplers.html#optuna.samplers.TPESampler).

### Low-level interface

This library also provides an "ask-and-tell" style interface.

```python
import numpy as np
from cmaes import CMA

def quadratic(x1, x2):
    return (x1 - 3) ** 2 + (10 * (x2 + 2)) ** 2

if __name__ == "__main__":
    cma_es = CMA(mean=np.zeros(2), sigma=1.3)

    for generation in range(50):
        solutions = []
        for _ in range(cma_es.population_size):
            x = cma_es.ask()
            value = quadratic(x[0], x[1])
            solutions.append((x, value))
            print(f"#{generation} {value} (x1={x[0]}, x2 = {x[1]})")
        cma_es.tell(solutions)
```

## Benchmark results

Optuna officially implements [a sampler based on pycma](https://optuna.readthedocs.io/en/latest/reference/integration.html#optuna.integration.CmaEsSampler).
It achieves almost the same performance. But this library is faster and simple.

### Algorithm's efficiency

| [Rosenbrock function](https://www.sfu.ca/~ssurjano/rosen.html) | [Six-Hump Camel function](https://www.sfu.ca/~ssurjano/camel6.html) |
| ------------------- | ----------------------- |
| ![rosenbrock](https://user-images.githubusercontent.com/5564044/73486735-0cd5ca80-43e9-11ea-9e6e-35028edf4ee8.png) | ![six-hump-camel](https://user-images.githubusercontent.com/5564044/73486738-0e9f8e00-43e9-11ea-8e65-d60fd5853b8d.png) |

This implementation (green) stands comparison with [pycma](https://github.com/CMA-ES/pycma) (blue).
See [benchmark](./benchmark) for details.

### Execution Speed

| trials/params | storage | pycma integration sampler |       this library      |
| ------------- | ------- | ------------------------- | ----------------------- |
|     100 /   5 |  memory |     4.976 sec (+/- 0.596) |   0.197 sec (+/- 0.078) |
|     500 /   5 |  memory |    71.651 sec (+/- 3.847) |   0.656 sec (+/- 0.044) |
|     500 /  50 |  memory |   291.002 sec (+/- 5.010) |   1.981 sec (+/- 0.041) |
|     100 /   5 |  sqlite |    16.143 sec (+/- 3.487) |  11.843 sec (+/- 1.390) |
|     500 /   5 |  sqlite |   129.436 sec (+/- 6.279) |  43.735 sec (+/- 2.676) |
|     500 /  50 |  sqlite |   397.084 sec (+/- 6.618) | 150.531 sec (+/- 1.113) |

[This script](./benchmark/speed_test.py) was run on my laptop with `--times 4`. So the times should not be taken precisely.
Even though, it is clear that this library is extremely faster than Optuna's pycma sampler (with Optuna v1.0.0 and pycma v2.7.0).

Links
-----

**Other libraries:**

I respect all libraries involved in CMA-ES.

* [pycma](https://github.com/CMA-ES/pycma) : Most famous CMA-ES implementation by Nikolaus Hansen.
* [cma-es](https://github.com/srom/cma-es) : A Tensorflow v2 implementation.

**References:**

* [1] [N. Hansen, The CMA Evolution Strategy: A Tutorial. arXiv:1604.00772, 2016.](https://arxiv.org/abs/1604.00772)
* [2] [Takuya Akiba, Shotaro Sano, Toshihiko Yanase, Takeru Ohta, Masanori Koyama. 2019. Optuna: A Next-generation Hyperparameter Optimization Framework. In The 25th ACM SIGKDD Conference on Knowledge Discovery and Data Mining (KDD ’19), August 4–8, 2019.](https://dl.acm.org/citation.cfm?id=3330701)
