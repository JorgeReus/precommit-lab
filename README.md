# Precommit LAB test

Precommit laboratory to play around with different precommit hooks

## Conform

Conform is an application that enforces predifined policies to a git repository such as:

- Conventional commit message validation
- 1 commit ahead of default branch
- Spelling
- Enforce signed commits (GPG)
- Developer Certificate of Origin Enforcement

### Precommit

To include conform in the precommit worflow you need:

1. Define it in the config file

```yaml
- repo: https://github.com/siderolabs/conform
  rev: v0.1.0-alpha.26 # Change it to the exact revision you need
  hooks:
    - id: conform
      # Notice the commit-ref flag, conform will search a valid ref on you git repo (e.g. tree .git/refs)
      entry: "conform enforce --commit-ref refs/remotes/origin/main --commit-msg-file"
      # This will run in a non-default stage, this being commit-message
      stages:
        - commit-msg
```

2. Install it
   Run `pre-commit install --hook-type commit-msg`

### Github Action

In this example we have setup a github action that runs in every pull request to main.
The flow is as follows:

1. Checkout to the commit of the PR

```yaml
- uses: actions/checkout@v3
  with:
    ref: ${{ github.event.pull_request.head.sha }}
    fetch-depth: 0
```

2. Install conform

```yaml
- name: Setup Conform
  run: |
    curl -sLo conform https://github.com/siderolabs/conform/releases/download/v0.1.0-alpha.26/conform-linux-amd64
    chmod +x conform
    sudo mv conform /usr/bin/conform
```

3. Run conform

```yaml
- name: Run conform
  continue-on-error: true
  id: conform
  run: |
    CONFORM_LOG_PATH="conform_log.txt"
    # Notice the commit-ref flag, modify as needed
    conform enforce --commit-ref refs/remotes/origin/main |& tee -a $CONFORM_LOG_PATH
    output=$(cat $CONFORM_LOG_PATH)
    output="${output//$'\n'/\\n}"
    # Export an output containing sterr & stdout from conform
    echo "::set-output name=output::${output}"
    if grep -qE "\sFAILED\s" $CONFORM_LOG_PATH
    then
        exit 1
    else
        exit 0
    fi
```

4. (Optional) Update you pull request with the conform output

```yaml
- name: Update pull request
  uses: actions/github-script@v6
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    script: |
      const output = `#### Conform Outcome \`${{ steps.conform.outcome }}\`
      [Logs](https://github.com/${context.payload.repository.full_name}/commit/${context.payload.pull_request.head.sha}/checks)

      <details><summary>Show Output</summary>

      \`\`\`\n
        ${{ steps.conform.outputs.output }}
      \`\`\`
      </details>


      *Pushed by: @${{ github.actor }}*`;

      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: output
      })
```
