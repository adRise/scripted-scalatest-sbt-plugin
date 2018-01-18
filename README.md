# scripted-scalatest-sbt-plugin

[![Codacy Badge](https://api.codacy.com/project/badge/Grade/244276b4573e4ae899443fa79c34822b)](https://www.codacy.com/app/daniel-shuy/scripted-scalatest-sbt-plugin?utm_source=github.com&utm_medium=referral&utm_content=daniel-shuy/scripted-scalatest-sbt-plugin&utm_campaign=badger)

| Plugin Version | SBT Version         | ScalaTest Version |
| -------------- | ------------------- | ----------------- |
| 0.1.X          | 0.13.X              | 3.X.X             |
| 0.2.X          | 0.13.X, 1.X.X       | 3.X.X             |

A SBT plugin to use [ScalaTest](http://www.scalatest.org/) with scripted-plugin to test your SBT plugins

Traditionally, to test a SBT plugin, you had to create subprojects in `/sbt-test`, then in the subprojects, create SBT tasks to perform the testing, then specify the tasks to execute in a `test` file (see http://www.scala-sbt.org/0.13/docs/Testing-sbt-plugins.html).

This is fine when performing simple tests, but for complicated tests (see http://www.scala-sbt.org/0.13/docs/Testing-sbt-plugins.html#step+6%3A+custom+assertion), this can get messy really quickly:
- It sucks to not be able to write tests in a BDD style (except by using comments, which feels clunky).
- Manually writing code to print the test results to the console for each subproject is a pain.

This plugin leverages ScalaTest's powerful assertion system (to automatically print useful messages on assertion failure) and its expressive DSLs.

This plugin allows you to use any of ScalaTest's test [Suites](http://www.scalatest.org/user_guide/selecting_a_style), including [AsyncTestSuites](http://www.scalatest.org/user_guide/async_testing).

## Note
- Do not use ScalaTest's [ParallelTestExecution](http://doc.scalatest.org/3.0.0/index.html#org.scalatest.ParallelTestExecution) mixin with this plugin. `ScriptedScalaTestSuiteMixin` runs `sbt clean` before each test, which may cause weird side effects when run in parallel.
- When executing SBT tasks in tests, use `Project.runTask(<task>, state.value)` instead of `<task>.value`. Calling `<task>.value` declares it as a dependency, which executes before the tests, not when the line is called.

## Usage

### Step 1: Include the scripted-plugin in your build

Add the following to your main project's `project/scripted.sbt` (create file it if doesn't exist):
```scala
libraryDependencies += { "org.scala-sbt" % "scripted-plugin" % sbtVersion.value }
```

### Step 2: Configure scripted-plugin

Recommended settings by SBT:

#### SBT 0.13 (http://www.scala-sbt.org/0.13/docs/Testing-sbt-plugins.html#step+2%3A+scripted-plugin)

```scala
// build.sbt
ScriptedPlugin.scriptedSettings
scriptedLaunchOpts := { scriptedLaunchOpts.value ++
  Seq("-Xmx1024M", "-XX:MaxPermSize=256M", "-Dplugin.version=" + version.value)
}
scriptedBufferLog := false
```

If you are using [sbt-cross-building](https://github.com/jrudolph/sbt-cross-building) (SBT < 0.13.6), don't add scripted-plugin to `project/scripted.sbt`, and replace `ScriptedPlugin.scriptedSettings` in `build.sbt` with `CrossBuilding.scriptedSettings`.

#### SBT 1.X (http://www.scala-sbt.org/1.x/docs/Testing-sbt-plugins.html#step+2%3A+scripted-plugin)

```scala
// build.sbt
scriptedLaunchOpts := { scriptedLaunchOpts.value ++
  Seq("-Xmx1024M", "-Dplugin.version=" + version.value)
}
scriptedBufferLog := false
```

### Step 3: Create the test subproject

Create the test subproject in `sbt-test/<test-group>/<test-name>`.

Include your plugin in the build.

See http://www.scala-sbt.org/0.13/docs/Testing-sbt-plugins.html#step+3%3A+src%2Fsbt-test for an example.

### Step 4: Include sbt-scripted-scalatest in your build

Add the following to your `sbt-test/<test-group>/<test-name>/project/plugins.sbt`:
```scala
addSbtPlugin("com.github.daniel-shuy" % "sbt-scripted-scalatest" % "0.2.0")
```

### Step 5: Configure `test` script

Put __only__ the following in the `sbt-test/<test-group>/<test-name>/test` script file:

`> scriptedScalatest`

### Step 6: Configure project settings for the plugin

In `sbt-test/<test-group>/<test-name>/build.sbt`, create a new ScalaTest Suite/Spec, mixin `ScriptedScalaTestSuiteMixin` and pass it into `scriptedScalaTestSpec`. When mixing in `ScriptedScalaTestSuiteMixin`, implement `sbtState` as `state.value`.

Example:
```scala
import com.github.daniel.shuy.sbt.scripted.scalatest.ScriptedScalaTestSuiteMixin
import org.scalatest.WordSpec

scriptedScalaTestSpec := Some(new WordSpec with ScriptedScalaTestSuiteMixin {
  override val sbtState: State = state.value

  "my-task" should {
    "do A" in {
      Project.runTask(myTask, state.value)
      // ...
      // assert(...)
    }

    "do B" in {
      Project.runTask(myTask, state.value)
      // ...
      // assert(...)
    }
  }
})
```

Another alternative is to move the ScalaTest Suite/Spec into another file (the file must reside within the `project` folder) and instantiate it in your `build.sbt` instead. Add a constructor argument that takes in a `sbt.State`, then implement `sbtState` as that constructor argument. Pass `state.value` into the constructor when instantiating the ScalaTest Suite/Spec.

Example:
```scala
// project/ExampleSpec.scala
import com.github.daniel.shuy.sbt.scripted.scalatest.ScriptedScalaTestSuiteMixin
import org.scalatest.WordSpec
import sbt.State

class ExampleSpec(state: State) extends WordSpec with ScriptedScalaTestSuiteMixin {
  override val sbtState: State = state

  "my-task" should {
    "do A" in {
      Project.runTask(myTask, state.value)
      // ...
      // assert(...)
    }

    "do B" in {
      Project.runTask(myTask, state.value)
      // ...
      // assert(...)
    }
  }
}
```

```scala
// build.sbt
scriptedScalaTestSpec := Some(new ExampleSpec(state.value))
```

See [Settings](#settings) for other configurable settings.

### Step 7: Use the scripted-plugin as usual

Append `-SNAPSHOT` to the main project `version` before running `scripted-plugin`.

Eg. Run `sbt scripted` on the main project to execute all tests.

## Settings

| Setting                    | Type                                           | Description                                                                                                                                                                                                                     |
| -------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| scriptedScalaTestSpec      | Option[Suite with ScriptedScalaTestSuiteMixin] | __Required__. The ScalaTest Suite/Spec. If not configured (defaults to `None`), no tests will be executed.                                                                                                                      |
| scriptedScalaTestDurations | Boolean                                        | __Optional__. If `true`, displays durations of tests. Defaults to `true`.                                                                                                                                                       |
| scriptedScalaTestStacks    | NoStacks / ShortStacks / FullStacks            | __Optional__. The length of stack traces to display for failed tests. `NoStacks` will not display any stack traces. `ShortStacks` displays short stack traces. `FullStacks` displays full stack traces. Defaults to `NoStacks`. |
| scriptedScalaTestStats     | Boolean                                        | __Optional__. If `true`, displays various statistics of tests. Defaults to `true`.                                                                                                                                              |

## Tasks

| Task               | Description                                                                                                                                                                                                                                                                 |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| scripted-scalatest | Executes all test configured in `scriptedScalaTestSpec`. This task must be [configured for scripted-plugin to run in the `test` script file](#step-5-configure-test-script). |

## Known Issues

Currently, mixing in `BeforeAndAfter` into your ScalaTest Suite/Spec and implementing `before` will cause errors. This will be fixed in future versions.

## Roadmap

I would like to create test cases for this plugin, ideally eventually using this plugin itself. Unfortunately, I just can't seem to figure out how to convert `scripted` from an `InputKey` into a `TaskKey` so that I can execute it with `Project.runTask`. I keep getting errors no matter what arguments I pass into `scripted.toTask`. I'll be really grateful if anyone can point me in the right direction.

## Licence

Copyright 2017 Daniel Shuy

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
