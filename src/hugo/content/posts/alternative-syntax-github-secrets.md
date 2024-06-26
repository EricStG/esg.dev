---
title: "Alternative syntax for accessing GitHub secrets in actions"
date: 2024-06-26T14:01:55-04:00
date: 2024-06-26T14:01:55-04:00
summary: An alternative syntax to access GitHub secrets dynamically within an action
tags:
- GitHub
keywords: GitHub, Actions, Secrets, Matrix, Workflows
---

If you have worked at all with GitHub actions, you might be familiar with the [common syntax](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions) for how to access them:
```yaml
${{ secrets.SuperSecret }}
```

But did you know there's an alternative syntax?
```yaml
${{ secrets[SuperSecret] }}
```

This can be useful in areas where the secret context is not accessible. For example, if you have a [matrix strategy](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs) where you need different secrets for different jobs, you can set the secret name in the matrix definition, and use this syntax to access it in the job itself.
```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        secret: secret1, secret2
    steps:
      - run: echo ${{ secret[matrix.secret] }}
```

The same strategy can be applied if you have secrets based on environment names. Consider you have the a secret called `SuperSecret_<environment>`, you can use something like:
```yaml
on:
  workflow_call:
    inputs:
      environment:
        type: string

jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - run: echo ${{ secret[format('SuperSecret_{0}', inputs.environment)] }}
```