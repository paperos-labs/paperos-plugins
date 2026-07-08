---
description: Deploy a repository to PaperOS infrastructure
argument-hint: <repo-url-or-path>
---

Deploy the repository indicated by the user to PaperOS.

Use the `deploy-app` skill from this plugin. It walks the deploy-mcp server's workflow correctly:

1. Call `list_templates` to discover available stacks
2. Detect the stack from the repo (`package.json` → node, `requirements.txt` → python-fastapi, `go.mod` → go)
3. Call `get_template(name=<chosen_name>)` to fetch the template content
4. Fill in ONLY the `Agent: fill in these values` section
5. Call `deploy_app` with `app_name`, `repo_url` (SSH form), `setup_script` (the completed template), and `env_vars`
6. Poll `get_deploy_status` until done — never ask the user to wait

Target repo: $ARGUMENTS

If `$ARGUMENTS` is a local path, inspect the files there to detect the stack and collect env vars, then deploy using the repo's git remote URL (SSH form). If it's already a URL, clone or shallow-fetch only what's needed to detect the stack and grep for env vars.

Critical rule (see the `deploy-app` skill for the full list):
- **NEVER** construct `setup_script` from scratch. It must be the verbatim `content` returned by `get_template(name)` with only the marked values section modified.
