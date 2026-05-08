---
# Trigger - when should this workflow run?
on:
  workflow_dispatch:
  schedule: weekly on monday around 7 a.m. (UTC - 3)

# Permissions - what can this workflow access?
# Write operations (creating issues, PRs, comments, etc.) are handled
# automatically by the safe-outputs job with its own scoped permissions.
permissions:
  contents: read
  issues: read
  pull-requests: read

# Tools - GitHub API access via toolsets (context, repos, issues, pull_requests)
tools:
  github:
    toolsets: [default]

# Network access
network: defaults

# Outputs - what APIs and tools can the AI use?
safe-outputs:
  create-issue:          # Creates issues (default max: 1)
    max: 5               # Optional: specify maximum number
  # actions:
  # activation-comments:
  # add-comment:
  # add-labels:
  # add-reviewer:
  # allowed-github-references:
  # assign-milestone:
  # assign-to-agent:
  # assign-to-user:
  # autofix-code-scanning-alert:
  # call-workflow:
  # close-discussion:
  # close-issue:
  # close-pull-request:
  # concurrency-group:
  # create-agent-session:
  # create-agent-task:
  # create-code-scanning-alert:
  # create-discussion:
  # create-project:
  # create-project-status-update:
  # create-pull-request:
  # create-pull-request-review-comment:
  # dispatch-workflow:
  # dispatch_repository:
  # environment:
  # failure-issue-repo:
  # footer:
  # group-reports:
  # hide-comment:
  # id-token:
  # link-sub-issue:
  # mark-pull-request-as-ready-for-review:
  # max-bot-mentions:
  # max-patch-files:
  # mentions:
  # merge-pull-request:
  # missing-data:
  # missing-tool:
  # noop:
  # push-to-pull-request-branch:
  # remove-labels:
  # reply-to-pull-request-review-comment:
  # report-failure-as-issue:
  # report-incomplete:
  # resolve-pull-request-review-thread:
  # scripts:
  # set-issue-type:
  # steps:
  # submit-pull-request-review:
  # threat-detection:
  # unassign-from-user:
  # update-discussion:
  # update-issue:
  # update-project:
  # update-pull-request:
  # update-release:
  # upload-artifact:
  # upload-asset:

---

# update-projects

Esse processo é responsavel por atualizar o arquivo dentro do diretorio profile/README.md deste repo, onde tem o link para todos os projetos que tem o prefixo gn-web, gn-api, gn-lib e gn-cli

## Instructions

Esse processo é responsavel por atualizar o arquivo dentro do diretorio profile/README.md deste repo, onde tem o link para todos os projetos que tem o prefixo gn-web, gn-api, gn-lib e gn-cli.
Os projetos estao separados em sessoes e por ordem alfabetica, entao o processo deve manter a ordem alfabetica e a separacao por sessoes.

Passos:
- Buscar por todos os repositorios que tem o prefixo gn-web, gn-api, gn-lib e gn-cli.
- Gerar o markdown com o nome do projeto e o link para o repositorio.
- Atualizar o arquivo profile/README.md com o markdown gerado.
- Criar um pull request com as mudanças feitas no arquivo profile/README.md, caso haja mudanças.
