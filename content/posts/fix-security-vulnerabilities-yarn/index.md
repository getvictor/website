+++
title = 'Fix security vulnerabilities in Yarn'
description = "Real-world examples of fixing security vulnerabilities in Yarn"
image = "cover.png"
date = 2024-04-10
categories = ["Security"]
tags = ["Yarn", "Vulnerabilities", "Application Security"]
draft = false
+++

## Why fix security vulnerabilities?

Security vulnerabilities are a common issue in software development. They can lead to data breaches, unauthorized access, and other security incidents. It is important to fix security vulnerabilities as soon as possible to protect your data and users.

## Finding vulnerabilities

Nowadays, it is possible to integrate various vulnerability scanning tools into your CI/CD pipeline. These tools can help you identify security vulnerabilities in your code and dependencies. One such tool is [OpenSSF Scorecard](https://securityscorecards.dev/), which combines multiple other tools into a single GitHub action. It uses the [OSV service](https://osv.dev/) to find vulnerabilities affecting your project's dependencies. OSV (Open Source Vulnerabilities) is a Google-based vulnerability database providing information about open-source projects' vulnerabilities.

In this article, we will focus on fixing a few recent real-world security vulnerabilities in our `yarn.lock` dependencies.

```
score is 3: 6 existing vulnerabilities detected:
Warn: Project is vulnerable to: GHSA-crh6-fp67-6883
Warn: Project is vulnerable to: GHSA-wf5p-g6vw-rhxx
Warn: Project is vulnerable to: GHSA-p6mc-m468-83gw
Warn: Project is vulnerable to: GHSA-566m-qj78-rww5
Warn: Project is vulnerable to: GHSA-7fh5-64p2-3v2j
Warn: Project is vulnerable to: GHSA-4wf5-vphf-c2xc
Click Remediation section below to solve this issue
```

### Using local tools to find vulnerabilities

In a local environment, we can use [OSV-Scanner](https://google.github.io/osv-scanner/) to find vulnerabilities in our dependencies. Running:

```bash
osv-scanner scan --lockfile yarn.lock
```

It will output the same vulnerabilities mentioned above but with additional details.
```
╭─────────────────────────────────────┬──────┬───────────┬────────────────┬─────────┬───────────╮
│ OSV URL                             │ CVSS │ ECOSYSTEM │ PACKAGE        │ VERSION │ SOURCE    │
├─────────────────────────────────────┼──────┼───────────┼────────────────┼─────────┼───────────┤
│ https://osv.dev/GHSA-crh6-fp67-6883 │ 9.8  │ npm       │ @xmldom/xmldom │ 0.8.3   │ yarn.lock │
│ https://osv.dev/GHSA-wf5p-g6vw-rhxx │ 6.5  │ npm       │ axios          │ 0.21.4  │ yarn.lock │
│ https://osv.dev/GHSA-p6mc-m468-83gw │ 7.4  │ npm       │ lodash.set     │ 4.3.2   │ yarn.lock │
│ https://osv.dev/GHSA-566m-qj78-rww5 │ 5.3  │ npm       │ postcss        │ 6.0.23  │ yarn.lock │
│ https://osv.dev/GHSA-7fh5-64p2-3v2j │ 5.3  │ npm       │ postcss        │ 6.0.23  │ yarn.lock │
│ https://osv.dev/GHSA-7fh5-64p2-3v2j │ 5.3  │ npm       │ postcss        │ 7.0.39  │ yarn.lock │
│ https://osv.dev/GHSA-7fh5-64p2-3v2j │ 5.3  │ npm       │ postcss        │ 8.4.21  │ yarn.lock │
│ https://osv.dev/GHSA-4wf5-vphf-c2xc │ 7.5  │ npm       │ terser         │ 5.12.1  │ yarn.lock │
╰─────────────────────────────────────┴──────┴───────────┴────────────────┴─────────┴───────────╯
```

Another way to find these vulnerabilities is by using the built-in [yarn audit](https://yarnpkg.com/cli/audit) command.

## Waiving vulnerabilities

In some cases, you may decide to waive a vulnerability. This approach means that you examine the vulnerability documentation and acknowledge it but decide not to fix it.

To waive a vulnerability for the OSV flow, you can create an `osv-scanner.toml` file in the root of your project. For example, to waive the `GHSA-crh6-fp67-6883` vulnerability, you can add the following to the `osv-scanner.toml` file:

```toml
[[IgnoredVulns]]
id = "GHSA-crh6-fp67-6883"
reason = "We examined this vulnerability and concluded that it does not affect our project for a very good reason."
```

In our example, we will not waive any vulnerabilities, but we will fix them by updating the dependencies.

## Updating an inner dependency

In our example, we have a vulnerability in the `@xmldom/xmldom` package. From the vulnerability URL, we know we must update this package to `0.8.4` or later.

Running `yarn why @xmldom/xmldom` will show that it is an inner dependency of another package:

```
=> Found "@xmldom/xmldom@0.8.3"
info Reasons this module exists
   - "msw#@mswjs#interceptors" depends on it
   - Hoisted from "msw#@mswjs#interceptors#@xmldom#xmldom"
```

Looking at `yarn.lock` shows:

```
"@xmldom/xmldom@^0.8.3":
  version "0.8.3"
  resolved "https://registry.yarnpkg.com/@xmldom/xmldom/-/xmldom-0.8.3.tgz#beaf980612532aa9a3004aff7e428943aeaa0711"
  integrity sha512-Lv2vySXypg4nfa51LY1nU8yDAGo/5YwF+EY/rUZgIbfvwVARcd67ttCM8SMsTeJy51YhHYavEq+FS6R0hW9PFQ==
```

We see that `0.8.4` will satisfy the dependency requirement of `^0.8.3`. We can update the package by deleting the above section from `yarn.lock` and running `yarn install`

We will then see the update:

```
"@xmldom/xmldom@^0.8.3":
  version "0.8.10"
  resolved "https://registry.yarnpkg.com/@xmldom/xmldom/-/xmldom-0.8.10.tgz#a1337ca426aa61cef9fe15b5b28e340a72f6fa99"
  integrity sha512-2WALfTl4xo2SkGCYRt6rDTFfk9R1czmBvUQy12gK2KuRKIpWEhcbbzy8EZXtz/jkRqHX8bFEc6FC1HjX4TUWYw==
```

## Upgrading an inner dependency by overriding the version

Our following vulnerability is in the `axios` package. We need to update it to `0.28.0` or later. By running `yarn why axios` we see that this package is part of a deep dependency chain:
```
=> Found "wait-on#axios@0.21.4"
info This module exists because "@storybook#test-runner#jest-playwright-preset#jest-process-manager#wait-on" depends on it.
```

The needed version `0.28.0` does not satisfy the `^0.21.4` requirement. We can override the version by adding the following to the `package.json` file:

```json
"resolutions": {
  "**/wait-on/axios": "^0.28.0"
},
```

## Upgrading the parent dependency

The following vulnerability is in the `lodash.set` package. The vulnerability URL shows that there is no fix for this vulnerability. We also see at [npmjs.com](https://www.npmjs.com/package/lodash.set) that this package was last updated eight years ago.

We need to update the parent package that uses `lodash.set`. Running `yarn why lodash.set` shows:

```
info Reasons this module exists
   - "nock" depends on it
   - Hoisted from "nock#lodash.set"
```

We update the parent by running `yarn upgrade nock@latest`. Luckily, the latest version of `nock` does not depend on `lodash.set`, and `lodash.set` is removed from `yarn.lock`.

## Removing a dependency

Sometimes the best way to fix a vulnerability is to remove the vulnerable dependency. This can be done with the `yarn remove <dependency>` command. However, this requires code changes. You must find a different library or implement the removed functionality yourself.

## Conclusion

We use the above strategies to fix the vulnerabilities in our project.
- Updating an inner dependency
- Upgrading an inner dependency by overriding the version
- Upgrading the parent dependency
- Removing a dependency

We can now rerun the vulnerability scanner to verify that we fixed the vulnerabilities.

In addition, we must run our unit test and integration test suite to ensure that the updates do not break our application.

## Fix security vulnerabilities in Yarn video

{{< youtube 59EpVz9mH_w >}}

*Note:* If you want to comment on this article, please do so on the YouTube video.
