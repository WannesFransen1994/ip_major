# Integration with CircleCI

## CircleCI free credits

You can always check your free build minutes (that refresh weekly) when you press your profile icon -> organization plans -> view plan -> plan usage. Be careful with this so that you don't run out on the exam, this is solely your responsibility to manage this.

## Link to CircleCI

I assume you can work with a dashboard, add your project and start building. This will definitely fail, which is normal.

## initialising the project

### Creating it

Create a new phoenix project:

```bash
mix phx.new a_demo --umbrella --database mysql
```

Answer "Yes" to questions, it'll most likely be necessary. After that make a private github repository and push it.

### making necessary branches

Make a dev branch, we'll use the master branch as the production branch for now. __Checkout to the dev branch.__

### Creating the config files

After that create in your root folder of your project a `.circleci` folder with a `config.yml` file.

The contents of your (initial) config file should look like:

```yml
version: 2  # use CircleCI 2.0 instead of CircleCI Classic
jobs:  # basic units of work in a run
  build:  # runs not using Workflows must have a `build` job as entry point
    parallelism: 1  # run only one instance of this job in parallel
    docker:  # run the steps with Docker
      - image: circleci/elixir:1.7.3  # ...with this image as the primary container; this is where all `steps` will run
        environment:  # environment variables for primary container
          MIX_ENV: test
      - image: circleci/postgres:10.1-alpine  # database image
        environment:  # environment variables for database
          POSTGRES_USER: postgres
          POSTGRES_DB: app_test
          POSTGRES_PASSWORD:

    working_directory: ~/app  # directory where steps will run

    steps:  # commands that comprise the `build` job
      - checkout  # check out source code to working directory

      - run: mix local.hex --force  # install Hex locally (without prompt)
      - run: mix local.rebar --force  # fetch a copy of rebar (without prompt)

      - restore_cache:  # restores saved mix cache
      # Read about caching dependencies: https://circleci.com/docs/2.0/caching/
          keys:  # list of cache keys, in decreasing specificity
            - v1-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v1-mix-cache-{{ .Branch }}
            - v1-mix-cache
      - restore_cache:  # restores saved build cache
          keys:
            - v1-build-cache-{{ .Branch }}
            - v1-build-cache
      - run: mix do deps.get, compile  # get updated dependencies & compile them
      - save_cache:  # generate and store mix cache
          key: v1-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
          paths: "deps"
      - save_cache:  # make another, less specific cache
          key: v1-mix-cache-{{ .Branch }}
          paths: "deps"
      - save_cache:  # you should really save one more cache (just in case)
          key: v1-mix-cache
          paths: "deps"
      - save_cache: # don't forget to save a *build* cache, too
          key: v1-build-cache-{{ .Branch }}
          paths: "_build"
      - save_cache: # and one more build cache for good measure
          key: v1-build-cache
          paths: "_build"

      - run:  # special utility that stalls main process until DB is ready
          name: Wait for DB
          command: dockerize -wait tcp://localhost:5432 -timeout 1m

      - run: mix test  # run all tests in project

      - store_test_results:  # upload junit test results for display in Test Summary
          # Read more: https://circleci.com/docs/2.0/collect-test-data/
          path: _build/test/lib/REPLACE_WITH_YOUR_APP_NAME # Replace with the name of your :app
```

We'll adjust this later on.

## Configuring our database

### application config

First of all we'll want to adjust our database. In your `/config/dev.exs` and `test.exs` file you'll want to adjust your database settings to a user and password that is correctly configured.

### circleci config

#### environment

first of all, delete following lines:

```yml
      - image: circleci/postgres:10.1-alpine  # database image
        environment:  # environment variables for database
          POSTGRES_USER: postgres
          POSTGRES_DB: app_test
          POSTGRES_PASSWORD:
```

According to the great [documentation](https://circleci.com/docs/2.0/postgres-config/#example-mysql-project) of CircleCI, we should be able to replace this with:

```yml
      - image: circleci/mysql:latest  # database image
        environment:  # environment variables for database
          MYSQL_ROOT_PASSWORD: t
```

_Yes we are lazy and we will just configure our app with root access for the test environment. Feel free to properly configure this when you're going to use your project in a production environment._

Make sure that the `config.yml` file parameters match with your `config/test.exs` file parameters.

#### port

Default mysql runs on port 3306, so replace the port `5432` with `3306`. Know what this part of the config does.

## other CircleCI config settings

### workspace

Default (in CircleCi) steps are ran in the /home/circleci/project folder. In this folder, circleci creates by default the app folder. We'll not do this and just use the home folder to clone our app in. This means that the following config:

```yml
    working_directory: ~/app  # directory where steps will run
```

will be changed to:

```yml
    working_directory: ~  # directory where steps will run
```

Remember that when you have a huge project, you'll likely want to use this feature to properly organize. You will automatically checkout / `git clone` your code to the above folder with the following command:

```yml
- checkout  # check out source code to working directory
```

### Update your elixir image version

Title says it all. Change your version from:

```yml
      - image: circleci/elixir:1.7.3  # ...with this image as the primary container; this is where all `steps` will run
```

to:

```yml
      - image: circleci/elixir:1.9  # or 1.10, up to you
```

### [INCOMPLETE] update your test result path as suggested by the docs:

You'll see en caps that you have to replace a variable with your project name. Let's do that for now, so that we'll see that this doesn't work properly with umbrella apps and default configurations.

```yml
      - store_test_results:  # upload junit test results for display in Test Summary
          # Read more: https://circleci.com/docs/2.0/collect-test-data/
          path: _build/test/lib/a_demo # Replace 
```

_Note that you have to use junit-formatted test results. We'll configure this later._

## Test locally

Execute the following commands:

```bash
mix test
```

Should give the following output:

```text
==> a_demo


Finished in 0.00 seconds
0 failures

Randomized with seed 135691
==> a_demo_web
...

Finished in 0.07 seconds
3 tests, 0 failures

Randomized with seed 135691

```

You'll see that there are no tests to be ran in the `a_demo` application, so let us create a dummy test. In your `a_demo/test` folder, create a dummy test like so:

```elixir
# Filename: /apps/a_demo/test/dummy_test.exs
defmodule DummyTest do
  use ExUnit.Case, async: true

  test "the truth" do
    assert true
  end
end
```

Now run the `mix test` command again. It should give something like:

```text
==> a_demo
.

Finished in 0.02 seconds
1 test, 0 failures

Randomized with seed 633151
==> a_demo_web
...

Finished in 0.07 seconds
3 tests, 0 failures

Randomized with seed 633151
```

Great! Now we'll push it onto the dev branch. Feel free to update your [gitignore file](http://gitignore.io/api/node,elixir,phoenix).

```bash
    git push --set-upstream origin dev
```

## Our first successful push

After we've pushed, we'll see our project building. You can see a lot of inbetween steps such as saving cache etc..., this is to reduce the build time on the CI/CD platform so that we don't have to compile the same stuff over and over again.

If everything went as suspected, we should see the test result. When you go to a build and see the details, there are 3 tabs:

* steps
* tests
* artifacts

For now we want to see a huge check mark at the tests tab. When you click it, you'll not see a check mark but a link to the documentation. So what happened? Let us check our steps.

When you see the mix test step, you'll notice that everything went as suspected. The only problem is with the "Uploading test results" step. Here we see that no files could be found, even though we've got the `store_test_results` step. This is because it expects a `.xml` file and it should be `junit` formatted. That's our next step.

## JUnit formatting our ExUnit tests

There's a useful library for this, [check it out](https://github.com/victorolinasc/junit-formatter). We're also using an umbrella application, which doesn't make things easier. Though the junit formatter provides a nice configuration for this. First of all, let us get those junit formatted reports.

### Adding the dependency

Both in `apps/a_demo/mix.exs` and `apps/a_demo_web/mix.exs` add the following line to your `deps`:

```elixir
{:junit_formatter, "~> 3.0", only: [:test]}
```

### Configuring our `ExUnit` testing library

After which we'll have to configure our `ExUnit` library to actually use this formatter. We do this in the `apps/a_demo/test/test_helper.exs` and `apps/a_demo_web/test/test_helper.exs` files. Prepend the following line:

```elixir
ExUnit.configure formatters: [JUnitFormatter, ExUnit.CLIFormatter]
```

Now run your tests and see what happens. The ouput is still the same, but now we have 2 `.xml` files in the following folders:

* `_build/test/lib/a_demo/test_junit_report.xml`
* `_build/test/lib/a_demo_web/test_junit_report.xml`

### Aggregating our test reports

While we could point our CircleCI config to the upperlying folder and let it recursively scan for all test `.xml` files, that's a bit error prone. We'll aggregate our test reports in a single folder. We do this with the following config:

```elixir
config :junit_formatter,
  report_file: "report_file.xml",
  # this is imported in your app! hence the double ..
  report_dir: "../../test-reports",
  print_report_file: true,
  prepend_project_name?: true
```

Now what do these lines mean? First we say that we'll configure the `:junit_formatter` application. After that we say what the file name needs to be. In order to actually aggregate our files, we'll need a common report directory. Note that this config file gets imported in each `apps/project` in our apps folder. That is why we go 2 folders up. The commands will be executed in e.g. `apps/a_demo_web`, which will then go 2 folders up, which is our root folder. After which we go into the test-reports directory. Finally we print where the file is reported to and we prepend the project name so that we don't have naming conflicts with our generated test reports.

You can freely experiment with placing the config in different places, do project specific config, etc... .

Though there's one more thing, actually generating the folder.

#### [PROJECT] Configuring the test reports folder

First make the test report folder locally. Then run mix test. You should see the following output (_note that it can differ slightly, such as user names, bash look-and-feel, etc... just pointing out the obvious here._):

```elixir
> $ ls -lash test-reports                              [±dev]
total 16K
4,0K drwxr-xr-x  2 wannes wannes 4,0K Feb  4 08:12 .
4,0K drwxr-xr-x 10 wannes wannes 4,0K Feb  4 08:12 ..
4,0K -rw-r--r--  1 wannes wannes  339 Feb  4 08:12 a_demo-report_file.xml
4,0K -rw-r--r--  1 wannes wannes  793 Feb  4 08:12 a_demo_web-report_file.xml
```

What we don't want to do though is push this on our version control! As they have no added value.

#### Ignoring our test reports folder

Add the following lines to your `.gignore` file:

```text
# Custom ignore rules
/test-reports
```

Validate with `git status`. Note that our test report folder isnt listed.

#### [CircleCI] Configuring the test reports folder

While we have this folder locally, we don't have this yet on our CircleCI build environment. Add the following before your mix test command:

```yml
- run: mkdir test-reports
```

Note that you'll want to do that after the `checkout` command and before the test command. In order to provide some insights into our CircleCI environment, we have the following folder structure:

```text
/
| home
| ---- circleci
| -------- project   => this folder contains your application code.
```

While `~` refers to the `/home/circleci` folder, when we execute `checkout`, it'll go into our `/home/circleci/project` folder (_if you've configured your working directory as `~` that is. Note that you can get different results based on this configuration._) So when you run `mkdir test-reports`, it'll make this folder similar to your local environment.

While this is all good and well, we'll need to update our `store-test-results` step as well like so:

```yml
      - store_test_results: 
          # Read more: https://circleci.com/docs/2.0/collect-test-data/
          path: test-reports
```

Wonderful! Push the changes onto your dev branch and let us see what happens. You should see a nice check mark in the "test" tab.

## Uploading our reports as artifacts

You'll often ~~want~~ have to communicate with your product owner / project manager what the progress is. One way to do this is with reports. ([Automatically generated test reports is not living documentation from BDD!](https://johnfergusonsmart.com/living-documentation-not-just-test-reports/))

We simply add this with the following lines in your CircleCI config:

```yml
      - store_artifacts:
          path: test-reports
```

Push and you'll be able to see your `.xml` files in your artifact step.

## Notes / TODO

* use environment variables with circleci and the config test.exs file. This will provide better "switchability" between local testing environment and circleci testing environment
* properly configure db environment
* properly document db port section
* You can create a pdf version of this document with the command `pandoc guide.md -o guide.pdf`
