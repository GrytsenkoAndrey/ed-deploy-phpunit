**TOC**


- [PHPUnit and Deploy](#ed-deploy-phpunit)
- [PHPUnit tips](#phpunit-tips)


# ed-deploy-phpunit

PHPUnit testing is a industry-standard tool nowadays. PHPStan is a genius sidekick. Everyone (I guess) love them. So, what’s the catch?

Developers absolutely hate running these tools locally.

Why? First off, it’s time-consuming. Lots of tests? That’s a lot of time. Big codebase? Static analysis takes ages too. Eventually, the team starts ignoring tests and analysis, becoming less productive.

How do we fix this? Automation, of course. So everyone on the team can check in and see the status of the project’s main branch.

Sure, CI/CD isn’t something new. There are plenty of ways and third-party services for storing test results and analysis. But I want to tell you about a sort-of-free method using GitHub’s CI/CD and wiki features. It can do a great job for a small team with a lack of devops resources.

I’ll repeat it — the following solution is not a silver bullet and definitely not the best one in the world. It’s just a quick and comparatively easy way to set up auto-testing in case you already have GitHub action runners.

The idea is simple: after changing the code in the master branch, we can automatically run tests and analysis and add the results to the project’s wiki.

There are just three steps to make this happen:

1 Have GitHub Action Runners, minimal know-how about YAML configuration, and familiarity with PHPUnit and PHPStan. That’s beyond this article, but I suppose you know what it is if you’re still reading this :)
2 Create a YAML configuration for GitHub Actions
3 Create custom Markdown formatters for PHP Unit and PHPStan (as they don’t support that out-of-the-box)


## Ok, show me how to do that!

Let’s start from the last point of the list — Markdown formatters.

Unfortunately, neither PHP Unit nor PHPStan can generate reports in Markdown format. It’s bad because the GitHub wiki allows the use of Markdown as the primary format. But it’s good that both tools allow for creating custom Markdown generators.

For PHPUnit, we need to write code that processes the SebastianBergmann\CodeCoverage\CodeCoverage class and generates Markdown based on it.

I won’t provide the code example since it’s quite simple but extensive, but I would like to mention that the documentation for this method is quite weak, so I recommend simply Googling it (or finding it in the PHP Unit codebase).

Methods that can be useful for you:

- getReport
- numberOfTestedClassesAndTraits
- numberOfClassesAndTraits
- numberOfTestedMethods
- numberOfMethods
- numberOfExecutedLines
- numberOfExecutableLine

You can test your code by running the command


```./vendor/bin/phpunit --stop-on-failure --coverage-php=coverage.php```


Then simply require this file into a variable, and you’ll get an instance of the SebastianBergmann\CodeCoverage\CodeCoverage class.

Also, you need to create a formatter for PHPStan. It’s simpler because PHPStan provides an interface (PHPStan\Command\ErrorFormatter\ErrorFormatter) that needs to be implemented.

I recommend finding the methods getNotFileSpecificErrors and getFileSpecificErrors and utilizing PHPStan\File\RelativePathHelper for nicer files’ paths.

Then register your class in the phpstan.neon configuration file:

```
services:
  errorFormatter.md:
    class: App\Console\MarkdownFormatter
```

And test it using a command like this:

```
./vendor/bin/phpstan analyse --configuration=phpstan.neon --error-format=md --no-progress >> 'Static Analysis.md'
```

This will run PHPStan analysis with the specified Markdown error format and store the output in a file named “Static Analysis.md”


### YAML configuration for GitHub Actions is left.

Let’s break it down step by step:

1 Give a name to our configuration.

```
name: PHPStan Analysis and PHPUnit Coverage Report
```

2. We want it run only on a specific branch, or when the configuration changes occurs

```
on:
  push:
    branches: [ "master" ]
    paths:
      - '.github/reporting.yml'
      - '**.php'
```

3. We need grant a write access to be able to update the wiki.

```
permissions:
  contents: write
```

4. Let’s set up Composer and necessary packages to run our tests. Just use PHP version that works for you, and add or remove packages from “php_extensions” line

```
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Cache Composer dependencies
        uses: actions/cache@v3
        with:
          path: ./vendor
          key: composer-${{ runner.os }}-${{ hashFiles('**/composer.lock') }}

      - name: Composer Init
        uses: php-actions/composer@v6
        with:
          php_version: "8.2"
          php_extensions: soap intl gd
```

5. Run PHPUnit in coverage mode

```
- name: PHPUnit
  run: |
    ./vendor/bin/phpunit --stop-on-failure --coverage-php=coverage.php
  env:
    XDEBUG_MODE: coverage
```

6. Now we can generate a Markdown report using our custom class

```
- name: Generate Coverage report
  run: |
    php CoverageGenerate.php >> Coverage.md
  env:
    XDEBUG_MODE: coverage
```

7. Run a static report

```
- name: PHPStan Analysis
  run: |
    ./vendor/bin/phpstan analyse --configuration=phpstan.neon --memory-limit=2048M --error-format=md --no-progress >> 'Static Analysis.md'
```

8. Checkout the wiki. Surprise, we can manage the wiki through Git!

```
- name: Wiki checkout
  uses: actions/checkout@v4
  with:
    repository: ${{github.repository}}.wiki
    path: 'wiki'
```

9. It’s left just to add our markdown files and push them to the wiki repository.

```
- name: Push
  run: |
    mv Coverage.md wiki
    mv 'Static Analysis.md' wiki
    cd wiki
    ls -la
    git config user.name actions-runner
    git config user.email actions-runner@github.com
    git add .
    git commit -m "Add Coverage and Analysis files" 
    git push
```

To summarise, whole YAML file will look like this:

```
name: PHPStan Analysis and PHPUnit Coverage Report

on:
  push:
    branches: ["master"]
    paths:
      - '.github/reporting.yml'
      - '**.php'

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Cache Composer dependencies
        uses: actions/cache@v3
        with:
          path: ./vendor
          key: composer-${{ runner.os }}-${{ hashFiles('**/composer.lock') }}

      - name: Composer Init
        uses: php-actions/composer@v6
        with:
          php_version: "8.2"
          php_extensions: soap intl gd

      - name: PHPUnit
        run: |
          ./vendor/bin/phpunit --stop-on-failure --coverage-php=coverage.php
        env:
          XDEBUG_MODE: coverage

      - name: Generate Coverage report
        run: |
          php CoverageGenerate.php >> Coverage.md
        env:
          XDEBUG_MODE: coverage

      - name: PHPStan Analysis
        run: |
          ./vendor/bin/phpstan analyse --configuration=phpstan.neon --memory-limit=1024M --error-format=md --no-progress >> 'Static Analysis.md'

      - name: Wiki checkout
        uses: actions/checkout@v4
        with:
          repository: ${{github.repository}}.wiki
          path: 'wiki'

      - name: Push
        run: |
          mv Coverage.md wiki
          mv 'Static Analysis.md' wiki
          cd wiki
          ls -la
          git config user.name actions-runner
          git config user.email actions-runner@github.com
          git add .
          git commit -m "Add Coverage and Analysis files" 
          git push
```


## PHPUnit tips

1. Run specific suite ```./vendor/bin/phpunit --testsuite <name>```
2. Run specific test with method ```./vendor/bin/phpunit --filter MyClassTest::myMethod```
3. Run specific test method ```./vendor/bin/phpunit --filter myTestMethodName```
4. Skip empty test ```phpunit --dont-report-useless-tests```
5. Mark incomplete ```$this->markTestIncomplete('This test has to be implemented later');```
6. One test depends on another
- use `PhpUnit\Framework\Attributes\Depends`
- return from the first test result
- use `#[Depends('firstTestName')]` on the second, pass parameter
7. [Fixture](https://docs.phpunit.de/en/10.5/fixtures.html)
8. Dataset - https://phpunit-documentation-russian.readthedocs.io/ru/latest/database.html#datasets-datatables




