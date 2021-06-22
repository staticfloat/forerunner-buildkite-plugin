# forerunner-buildkite-plugin
> Dynamically trigger templated buildkite pipelines on git diffs

## Basic Usage

To use this plugin, you will typically have a family of pipelines in `.buildkite/`, each meant to be triggered if certain files have been touched in the current git commit/pull request.
If you have multiple pipelines to trigger, list multiple instances of the plugin, as each instance can only trigger a single `target`.
Each `target` watches for a set of paths, and has a certain type.
As of this writing, the three types of targets are `simple`, `template` and `comand`:

* A `simple` target is one that gets triggered if any of the files it is watching for are modified.
  There is no templating, no way to know which file triggered the pipeline, etc...
  This is most useful if you want to, for instance, trigger the tests of an overall package if any of its source files are changed.

* A `template` target is one that has a simple `sed` command applied to it to replace `{PATH}` with the path to the file that has been modified.
  If multiple paths trigger a `template` pipeline, it is triggered once for each path (subject to `path_processor` alterations, detailed below).

* A `command` target is one that invokes a command once for each path (subject to the `path_processor` alterations, detailed below) and is expected to generate pipelines and pass them to `buildkite-agent pipeline upload` itself.
  This command type is by far the most flexible and allows for truly devious pipelines to be created.

As mentioned above, `template` and `command` targets are, by default, invoked once for each modified file.
To control this behavior, the user may provide a `path_processor` argument to munge the paths in an intelligent way.
By default, `forerunner` uses the `per-file` path processor to template the pipeline once for each file modified.
However, some pipelines may wish to run upon the directories containing all modified files, or may wish to run once per overall project in a large monorepo.
To support this, the user may use the `path_processor` argument to collect/filter/transform the list of paths that will be iterated over when invoking the target.
There are currently a few built-in path processors, located within [the `lib/path_processors` directory](./lib/path_processors), however defining your own is quite simple.
Users can provide a repo-relative path to a processor of their choosing and it can perform all the repo-specific path processing necessary to properly batch target launching.

## Example pipeline definition

```yaml
steps:
  - label: ":runner: Dynamically launch Pipelines"
    plugins:
      - staticfloat/forerunner:
          # This will create one job per file
          watch:
            - path: library/**/*.jl
          path_processor: per-file
          target: .buildkite/process_julia_code.yml
          target_type: template
      - staticfloat/forerunner:
          # This will create one job per project directory
          watch:
            - path: library/**/*
          path_processor: .buildkite/path_processors/per-project
          target: .buildkite/library_project.yml
          target_type: template
      - staticfloat/forerunner:
          # This will create one job overall, throwing all path information away
          watch:
            - path: src/**/*.jl
            - path: .*\\.toml
          target: .buildkite/run_tests.yml
      - staticfloat/forerunner:
          # This will run an arbitrary command targeting each file changed
          watch:
            - path: **/*/Project.toml
          path_processor: per-file
          target: .buildkite/commands/create_update_project_pipeline.sh
          target_type: command
```
