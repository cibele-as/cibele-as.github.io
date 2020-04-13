---
title: Testing CircleCI jobs locally
tags: [CircleCI]
style: fill
color: info
---

This quick tutorial will give you a basic understanding of how to execute CircleCI jobs locally not only to validate the
yaml sintax but also to validate the functionality before executing it on CircleCI's environment.

## The setup

Download the CircleCI CLI, refer to the installation docs [here](https://circleci.com/docs/2.0/local-cli/#installation)

As a MAC/Linux user I've just needed to execute the command below:
```bash
curl -fLSs https://circle.ci/cli | bash
```

The basic configuration for CircleCI is to have a `config.yml` file inside the folder `.circleci`, for the sake of brevity
I will use one of my public repos that you can find on [github](https://github.com/rribeiro1/forum-kotlin-spring-boot)

Let's considering the file below, it is basically building, testing and uploading test metrics to CodeClimate. 

```yaml
# Kotlin Gradle CircleCI 2.0 configuration file
#
version: 2
jobs:
  build:
    work_directory: ~/forum-app
    docker:
      - image: circleci/openjdk:8-jdk
      - image: circleci/postgres:9.4

    environment:
      # Customize the JVM maximum heap limit
      JVM_OPTS: -Xmx3200m

    steps:
      - checkout

      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "build.gradle" }}
            # fallback to using the latest cache if no exact match is found
            - v1-dependencies-

      - run:
          name: Install Dependencies
          command: |
            wget https://codeclimate.com/downloads/test-reporter/test-reporter-latest-linux-amd64 -O ./cc-test-reporter
            chmod +x ./cc-test-reporter
      - run:
          name: Build & Tests
          command: gradle clean build jacocoTestReport jacocoTestCoverageVerification jacocoFixForCodeClimate

      - run:
          name: Upload Tests to CodeClimate
          command: |
            ./cc-test-reporter format-coverage -d -t jacoco -o jacoco.xml ./build/reports/jacoco/test/jacoco.xml
            ./cc-test-reporter upload-coverage -i jacoco.xml
      - store_test_results:
          path: ./build/test-results

      - save_cache:
          paths:
            - ~/.gradle
          key: v1-dependencies-{{ checksum "build.gradle" }}
```

Once the CircleCI cli is installed we can use the `validate` command by executing: 

```bash
circleci config validate -c .circleci/config.yml

# Config file at .circleci/config.yml is valid.
```

And to run a job locally, use the local execute command:

```bash
circleci local execute --job "jobname"
```

In our case, the job is `build`, but you can eventually have more jobs.

```bash
circleci local execute --job "build"

# Downloading latest CircleCI build agent...
# Docker image digest: sha256:2c2fd6a4655b259ac40f3c168b82200d4a7cebdc8287c4eb866a348b0191403d
# ====>> Spin Up Environment
# Build-agent version  ()
# Docker Engine Version: 19.03.5
# Kernel Version: Linux 4c318118bfc4 4.19.76-linuxkit #1 SMP Thu Oct 17 19:31:58 UTC 2019 x86_64 Linux
...
```

If your job depends on `encrypted variables` which we configure using CircleCI's UI, we can setting those environment variables via the CLI:

```bash
circleci local execute --job "jobname" --env MY_VARIABLE=MY_VALUE
```

That's it. 