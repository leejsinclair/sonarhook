
# Project

A Claude code hook that attempts to call sonarqube on port 9000 for code review.

This will require:

1. pushing changes to sonarqube,
2. polling when the analysis is complete
3. requesting feedback fron sonarqube
4. saving feedback to a .md file with a the date timestamp
5. giving the feedback to the running agent

In addition handle the situation where the project is running in a devcontainer and need to communicate with the host via that port as well.

So there are 2 environments to test in:

1. on the local development environment, and
2. inside a development container.

I would like this to be a hook that run the [sonarqube cli](https://www.sonarsource.com/sonarqube/cli)

## Setup

Install speckit: [Speckit](https://github.com/github/spec-kit)

Install speckit extensions:

* [Know Your Spec](https://github.com/leejsinclair/speckit-preset-know-your-spec)
* [Grill me integrated](https://github.com/leejsinclair/speckit-grill-specify)
* [TDD integrated](https://github.com/leejsinclair/speckit-tdd-integrated)

Verification you can make calls like:

`curl -s http://localhost:9000/api/system/status`

[SonarQube API Docs](https://docs.sonarsource.com/sonarqube-server/2026.4/extension-guide/web-api)
