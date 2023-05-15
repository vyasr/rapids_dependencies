# About

This repository defines metapackages that pin common dependencies across RAPIDS.
These metapackages provide a centralized set of version pinnings to ensure that RAPIDS packages are compatible and may be installed into the same environment.
They also provide a single source of truth from which to perform updates across RAPIDS.

# Usage

The pip metapackage may be used in another package like so:
```toml
[build-system]
requires = ["rapids_dependencies[build]>=23.06.00a0", "other_build_dependency"]

[project]
dependencies = ["rapids_dependencies[numpy,cupy]>=23.06.00a0", "other_run_dependency"]
```

In a conda recipe, the equivalent usage would be

```yaml
requirements:
  host:
    rapids-build-dependency >=23.06.00a0

  run:
    rapids-numpy-dependency >=23.06.00a0
    rapids-cupy-dependency >=23.06.00a0
```

The conda syntax is a bit more verbose due to the lack of support for optional dependencies a la pip, but we can mitigate that to some extent with composite metapackages like the `rapids-build-dependency` above that itself pulls in multiple packages.

## Pinning the Metapackage Version in its Dependents

The metapackages should follow the same CalVer scheme as the rest of RAPIDS.
During the development cycle, all RAPIDS packages using one of the metapackages in this repository should use a `>=*alpha` pinning so that the latest nightly will always be picked up.
For instance, in the examples above we are using `23.06.00a0`.
At release time, those pinnings should be updated to be `==stable` pinnings, e.g. `==23.06.00`.
When a new branch is cut for the next release, the pinnings should be updated to the next nightly version, e.g. `>=23.08.00.a0`.

The reason for this approach is that while conda will install nightly packages by default if no stable versions are available, pip will not do so unless either 1) the `--pre` parameter is provided to a `pip install` command, or 2) the version constraint [explicitly allows nightlies](https://pip.pypa.io/en/stable/cli/pip_install/#pre-release-versions) as above.

### Mitigating Churn

This approach will introduce some additional churn to each dependent repo.
Every repo will need one PR to pin the dependency at release, and then another to unpin.
We currently do this for dask dependencies, so if the metapackage handles dask (which it must) then the churn shifts to the metapackage dependency itself.
However, by using a consistent formulaic name for our metapackages we should be able to mitigate this by making the updates easily scriptable using the [GH CLI](https://cli.github.com/).
Then it would simply be a matter of running those scripts as part of our release process (and we may be able to automate this further when RAPIDS eventually gets around to changing our branching strategy).
We can already largely do this with [r3](https://github.com/ajschmidt8/r3/), or possibly with [git-xargs](https://github.com/gruntwork-io/git-xargs).

## Updating Pinnings

If a common dependency version is to be updated, we have two paths:

1. Safe but tedious: Open PRs to each package using that particular dependency (e.g. `numpy`) replacing use of the metapackage with a direct pinning on the package (e.g. `rapids-dependency[numpy]` -> `numpy>=X,<Y`). Once the PRs all pass tests, update and re-release the metapackage and close the PRs. This strategy is probably what we'll typically leverage in practice. Note that it is essentially how we update dependencies in rapids-cmake, and it has generally served us well despite being a bit tedious.
2. Unsafe but simple: Simply bump the dependency in the metapackage, make a new release, and monitor builds of RAPIDS packages to see if anything breaks. I would not generally recommend this, but for dependencies that are typically very stable it may be acceptable. It may also be an OK strategy to employ very early in the RAPIDS release cycle. On the plus side, it is significantly less work when it works.

## Patch Releases

If we need to update a version pinning for a dependency (say scikit-build releases a breaking change) after a release, we need to release a patch version of the metapackage.
If we need the patch version numbers to synchronize across the metapackage and RAPIDS packages (e.g. `cudf==$version` must always depend on exactly the same `rapids-dependency==$version`), then we'll need to make patch releases of all RAPIDS packages in lockstep.
If we are OK with those not being exactly synchronized (e.g. `cudf==23.06.02` depends on `rapids-dependency==23.06.01` while `cuml==23.06.03` depends on `rapids-dependency==23.06.02`) then we can freely re-release the metapackage as needed and just bump its version in the dependency lists of its dependents.
The correct answer here depends on our general release strategy for RAPIDS.

## Dask

Currently unless there is a breaking change in the HEAD of a dask repo that we cannot yet support, our typical strategy is to have dask pinnings that look something like `dask>=YYYY.MM.DD` in our package specs but then install the latest nightly (from source for pip, via the special dask nightly label in conda) for testing.
Then, when we make a RAPIDS release we pin to the latest dask release.
This bifurcated approach during development leads to an inaccurate statement of our dask dependency since in practice we cannot guarantee compatibility with anything except the HEAD of the `dask*` repos.
Therefore, I propose that we eschew `>=` pinnings altogether for dask in this repo and instead always require the latest HEAD.
We can do this by using a `git+https...` dependency on dask for pip, and for conda we can simply nest that inside a `pip:` section.
That will ensure that our dependency is always consistent.
Of course, if dask introduces a breaking change that will break our nightly packages as well, but that is no different from the current situation since ultimately any breaking change in dask means that users of our nightlies have to update to the latest nightlies of both `dask*` and RAPIDS packages.

Concretely, the solution would look like replacing:
```yaml
  - name: rapids_dask_dependency
    requirements:
      run:
        - dask >=2023.3.2
        - dask-core >=2023.3.2
        - distributed >=2023.3.2
```
with 
```yaml
  - name: rapids_dask_dependency
    requirements:
      run:
        pip:
            - git+https://github.com/dask/dask.git@main
            - git+https://github.com/dask/distributed.git@main
```
and then reverting to a hard pin like `==2023.3.4` at release time.

# Notes

- I see no reasonable way to make pip and conda metapackage names align exactly, but I think that's fine.
- For conda if we want to include a Python version dependency that will come in the form of a separate metapackage (whereas for pip it goes into the `requires-python` key of pyproject.toml.
- After a release is completed, the `>=alpha` pinnings could cause problems because installing a nightly from a previous release would pull nightly dependencies corresponding to the current release. I do not think that this is a use case that we need to support, though, especially since we eventually purge our nightlies anyway
- We could store these metapackages into a separate repo, or move them into the integration repo with our other metapackages. These metapackages are a bit different from those though since these are for our dependencies while those are for our dependents and users.
In the long run we could manage both the pip and conda metapackage using [rapids-dependency-file-generator](https://pypi.org/project/rapids-dependency-file-generator/) so that we only have to maintain pinnings in one place. That requires that dfg support meta.yaml files, which is in progress.
