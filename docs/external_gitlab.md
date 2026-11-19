# External GitLab Support for BuildStream: Design for Approval

**Related ER:** ER-BSM-003-external-gitlab-log-observability
**Related story:** ER-BSM-003-external-gitlab-config-and-project
**Code reviewed:** `/root/latest/omnia/src/build_stream`
**Status:** Draft for lead approval. No code has been changed.

---

## 1. What we are trying to do

Today BuildStream supports one kind of GitLab: a **hosted GitLab** that Omnia installs on a host (Omnibus RPM), together with its own runner.

We want a second kind: an **external GitLab** that the customer already has. The customer installs, upgrades and secures that GitLab. Omnia only does this on it:

- Creates (or reuses) one project, `omnia-catalog` by default.
- Pushes our CI files and input files into that project.
- Creates the tokens and CI/CD variables the pipelines need.
- Checks that a runner is available.

Omnia does **not** install GitLab, does **not** install a runner, and does **not** need SSH access to the GitLab server in external mode.

Hosted mode must keep working exactly as it does today.

---

## 2. What the user sees

### 2.1 Choosing the mode

One new required input in `build_stream_config.yml`:

```yaml
gitlab_mode: "self_hosted"   # Omnia installs GitLab (today's behaviour)
# or
gitlab_mode: "external"      # customer already has GitLab
```

Existing hosted users must add `gitlab_mode: self_hosted` when they upgrade. The validator will say so clearly if it is missing.

### 2.2 Inputs in `build_stream_config.yml`

| Input | Hosted | External | Notes |
|---|---|---|---|
| `enable_build_stream` | Yes | Yes | Same as today |
| `build_stream_host_ip`, `build_stream_port` | Yes | Yes | Same as today. The runner calls this address. |
| `gitlab_mode` | Required | Required | **New** |
| `gitlab_host` | Required | Not used | Hosted: IP of the server where GitLab is installed |
| `gitlab_https_port` | Used | Not used | In external mode the port goes inside the URL |
| `gitlab_min_storage_gb`, `gitlab_min_memory_gb`, `gitlab_min_cpu_cores`, `gitlab_puma_workers`, `gitlab_sidekiq_concurrency` | Used | Not used | Install sizing only |
| `gitlab_external_url` | Not used | Required | **New.** Base URL only |
| `gitlab_project_namespace` | Not used | Required | **New.** Group or user namespace |
| `gitlab_ca_cert_path` | Not used | Optional | **New.** PEM file for an internal or self-signed CA |
| `gitlab_project_name` | Yes | Yes | Default `omnia-catalog` |
| `gitlab_project_visibility` | Yes | Yes | Default `private` |
| `gitlab_default_branch` | Yes | Yes | Default `main` |
| `gitlab_project_ci_timeout` | Yes | Yes | Default 9000 |
| `log_*` inputs | Yes | Yes | Same as today (ER-BSM-003 log work) |

### 2.3 What `gitlab_external_url` looks like

It is the **base address of the GitLab server**. It is not the project URL.

```yaml
gitlab_external_url: "https://gitlab.corp.example.com"
gitlab_external_url: "https://100.10.1.185"        # IP, default port 443
gitlab_external_url: "https://100.10.1.185:8443"   # add the port if it is not 443
```

A project URL such as `https://100.10.1.185/root/omnia-catalog` is the **output**, which Omnia builds as `<gitlab_external_url>/<gitlab_project_namespace>/<gitlab_project_name>`:

| Part of the URL | Input |
|---|---|
| `https://100.10.1.185` | `gitlab_external_url` |
| `root` | `gitlab_project_namespace` |
| `omnia-catalog` | `gitlab_project_name` |

### 2.4 What the credential utility asks (vault file `build_stream_credentials.yml`)

The credential utility (`credential_utility` role) currently asks for the same things in both modes. With this change the prompts depend on `gitlab_mode`:

| Prompt | Hosted | External | What it is |
|---|---|---|---|
| `gitlab_root_password` | Asked (mandatory) | **Not asked** | Password Omnia sets for the root user of the GitLab it installs |
| `gitlab_ssh_password` | Asked (mandatory) | **Not asked** | Used once to set up passwordless SSH to the GitLab host |
| `gitlab_api_token` | **Not asked** | **Asked (mandatory, new)** | Personal access token (PAT) the user created in GitLab (see step 4 in 3.1) |
| `build_stream_auth_username` / `build_stream_auth_password` | Asked | Asked | BuildStream API login. Omnia pushes these into GitLab as masked CI variables. The user does not give them to GitLab. |
| `postgres_user` / `postgres_password` | Asked | Asked | Same as today |

Rules for the new prompt: the value is hidden while typing, it must not be empty, it is stored encrypted in the vault file only (never in `build_stream_config.yml`, never in logs), and the file is root-owned with mode 0600.

Omnia also stores `gitlab_project_id` and the trigger token in the vault file. In hosted mode it also stores `gitlab_root_token` today. In external mode it will **not** store anything powerful: see section 7.

Files that change for this: the credential schema (`build_stream_credentials.json`), the credential template (`build_stream_credential.j2`), and the `credential_utility` role.

---

## 3. What the user must do before running (one time)

This part is done by the GitLab admin and/or the person running Omnia.

### 3.1 Checklist

1. **Decide the namespace.**
   - A group such as `infra/omnia-builds`, or a user namespace such as `root`.
   - The user who will own the PAT must be allowed to create projects there.
2. **Make sure a runner is registered and online.**
   - Omnia does not install one. It can be project, group or instance scoped.
   - It must be allowed to run **untagged** jobs, because our CI files do not use `tags:`.
   - It must not be paused.
   - There is **no runner ID or runner token input**. Omnia only looks for an online runner through the project API.
3. **Make network access work** (the table in 3.2).
   - The runner must reach the OIM BuildStream API (`build_stream_host_ip:build_stream_port`) and trust the BuildStream certificate.
   - If GitLab uses an internal CA, the same CA must also be set up on the runner.
4. **Create a personal access token (PAT).**
   - In GitLab: User settings, then Access tokens.
   - Scopes: `api` and `write_repository`.
   - The token's user needs rights to create projects in the namespace, and Maintainer on the project.
   - No instance administrator rights are needed.
   - Copy the token when GitLab shows it, because it is shown only once.
5. **If GitLab uses a private CA:** put the CA PEM file on the OIM and set `gitlab_ca_cert_path`.
6. **Edit the config** (`gitlab_mode: external`, URL, namespace) and **run the credential utility** to enter the PAT.

### 3.2 What must be able to reach what

| From | To | Port | Why | Who makes sure |
|---|---|---|---|---|
| OIM (Omnia playbooks) | External GitLab | 443 or the port in the URL | All setup API calls | Customer network team |
| OIM (playbook watcher, cadence) | External GitLab | Same | Cadence catalog fetch and update | Customer network team |
| OIM (BuildStream upload step) | External GitLab | Same | Uploads `failed_nodes.json` after a restart | Customer network team |
| **GitLab runner** | **OIM BuildStream API** | `build_stream_port` (default 8010) | CI jobs call BuildStream | Customer (Omnia cannot test this from the OIM) |
| GitLab runner | External GitLab | Same as above | Runner polls for jobs | Customer GitLab admin |
| GitLab runner | Container registry for the CI job images | Registry port | Pulls the images used by CI jobs | Customer (matters for air-gapped sites) |
| User's browser | External GitLab | Same | To create the PAT | The user |

Honest limit: from the OIM, Omnia can check that GitLab is reachable and that a runner is online. It cannot check that the **runner can reach the OIM**. This must be in the customer documentation. We can also offer an optional smoke-test pipeline later.

---

## 4. Checks that run before anything is created (external mode)

These are new "preflight" checks. They run in order and **stop at the first failure**, with a message that says what to fix. Nothing is created in GitLab until all of them pass.

| # | Check | How | Error example |
|---|---|---|---|
| 1 | Config is complete | Schema and semantic validation | `gitlab_external_url is required when gitlab_mode is 'external'` |
| 2 | Token is in the vault | Credential file check | `gitlab_api_token not found` |
| 3 | CA file exists and is valid PEM (if set) | Local file check | `CA certificate file not found at <path>` |
| 4 | GitLab is reachable from the OIM | DNS and TCP connect to host:port | `Cannot reach <url>` |
| 5 | TLS works | HTTPS handshake with the custom CA or system CA | `TLS certificate validation failed` |
| 6 | GitLab version is supported | `GET /api/v4/version`, accept 17.x to 19.x | `GitLab 16.x is below minimum 17.0` |
| 7 | Token is valid, not expired, and has the right scopes | `GET /api/v4/personal_access_tokens/self` | `Token is missing scope write_repository` |
| 8 | Who owns the token | `GET /api/v4/user` | `Token is invalid or expired` |
| 9 | Namespace exists | `GET /api/v4/namespaces/:path` | `Namespace infra/omnia-builds not found` |
| 10 | Token's user can create projects in the namespace | Group or user-namespace permission and the group's project-creation rule | `Cannot create project in namespace: check the token user's project-creation rights` |
| 11 | Project name collision | Look up `<namespace>/<project>` | Not found: will be created. Found with the Omnia marker: adopted. Found without the marker: stop with `Project exists but is not Omnia-managed` |
| 12 | BuildStream services are healthy on the OIM | Existing precheck tasks (BSM, Postgres, watcher, SSL cert) | Same as today |

Checks that need the project to exist, so they run **right after the project is created or adopted** and before CI files are pushed:

| # | Check | Error example |
|---|---|---|
| 13 | Token can create the project access token (Maintainer) | `Cannot create project access token: user needs Maintainer on the project` |
| 14 | Token can protect branches, set variables, create triggers | Operation-specific message |
| 15 | At least one online, unpaused runner that accepts untagged jobs (`GET /projects/:id/runners`) | `No online runner found for the project. Register a runner and make sure it is online.` |

If any of 13 to 15 fail, the compensation step (section 6) removes what this run created.

---

## 5. How the flow works

### 5.1 Hosted GitLab (unchanged)

1. Validate the config and the credentials (`validate_build_stream_input`, `credential_utility`).
2. Prepare BuildStream (`prepare_build_stream.yml`: Postgres, BSM API, watcher).
3. Check the GitLab host details and set up SSH (`prereq_checks`, `gitlab_passwordless_ssh`).
4. Check OIM prerequisites (`check_oim_prerequisites`).
5. Install and configure GitLab on the host (`hosted_gitlab`: packages, firewall, TLS certs, Omnibus RPM, `gitlab.rb`, root password).
6. Create the root token with `gitlab-rails`, then create the project.
7. Create the `omnia-ci` token and the trigger, set CI variables, push CI files.
8. Deploy the runner container and wait until it is online.
9. Show the summary.

### 5.2 External GitLab (new)

1. Validate the config and the credentials (the same roles, now mode-aware).
2. Run the external preflight checks 1 to 12 (section 4).
3. Prepare BuildStream (`prepare_build_stream.yml`, unchanged).
4. Create or adopt the project (checks 13 and 14 happen here), and write the journal.
5. Protect the default branch.
6. Create the `omnia-ci` project token, and revoke the old one after the new one is in place.
7. Create the trigger token and store it in the vault file.
8. Set the CI/CD variables.
9. Push the CI files, `omnia.env`, input files and example catalogs.
10. Verify the runner (check 15).
11. Show the summary, including the project URL.
12. If any step from 4 onward fails, compensate: undo what this run created, in reverse order. An adopted project is never deleted.

Everything in external mode runs from the OIM (`localhost`) using the GitLab API. There is no SSH to the GitLab server.

### 5.3 Other playbooks

| Playbook | Hosted | External |
|---|---|---|
| `sync_gitlab_files.yml` (push updated CI and input files only) | Reads the root token from the GitLab host over SSH | Uses the API with the project token. No SSH. |
| `cleanup_gitlab.yml` | Removes the installation, runner, TLS and credentials | Removes only Omnia-owned resources (CI variables, tokens, trigger, journal). Never removes the runner or an adopted project, and never touches the customer's GitLab. |
| Cadence (playbook watcher) | Uses the stored token and `gitlab_host` | Uses the project token and `gitlab_external_url`, plus the CA path |

---

## 6. Role and task design

The ER proposes a new `external_gitlab` role with the shared API code factored out of `hosted_gitlab`. In this design the shared tasks go to one **shared role**. The name `gitlab_common` is a proposal, and an open question for review.

### 6.1 Existing roles that stay the same for both modes

| Role or playbook | What changes |
|---|---|
| `build_stream_setup` | No change |
| `validate_build_stream_input` and `build_stream_config_validator.py` | Becomes mode-aware (section 8) |
| `credential_utility` | Prompts depend on `gitlab_mode` (section 2.4) |
| `precheck_build_stream` | No change |
| `postgres`, `deploy_bsm` (prepare phase) | No change |

### 6.2 Common tasks (run in both modes): moved into the shared role

Today these are `hosted_gitlab` tasks that use the root token and the host address. We change them to use generic variables: `gitlab_api_url`, `gitlab_auth_token`, `gitlab_ca_cert`, `gitlab_project_id`.

| # | Shared task | Comes from today | Notes |
|---|---|---|---|
| 1 | `create_or_find_project.yml` | The project part of `create_project.yml` | Hosted creates the project. External adds the namespace and ownership marker through its own wrapper. |
| 2 | `protect_branch.yml` | The first half of `create_ci_token.yml` | Needed for protected variables |
| 3 | `create_ci_token.yml` | `create_ci_token.yml` | `omnia-ci`, `api` scope, Maintainer, 360 days, old token revoked afterwards |
| 4 | `create_trigger.yml` | `create_trigger.yml` | Where the trigger token is stored differs by mode (below) |
| 5 | `set_pipeline_variables.yml` | `set_pipeline_variables.yml` | `GITLAB_API_TOKEN`, `BSM_API_URL`, `BSM_API_USERNAME`, `BSM_API_PASSWORD`, `BSM_API_CERT`, `BSM_LOG_TAIL_LINES` |
| 6 | `push_ci_files.yml` | `push_ci_files.yml` | Pushes `.gitlab-ci*.yml` (the files move to a shared `files/` folder) and `omnia.env` |
| 7 | `sync_input_file.yml` | `sync_input_file.yml` | Pushes the input files |
| 8 | `push_example_catalogs.yml` and `push_single_example_catalog.yml` | Same | Example catalogs |
| 9 | `verify_runner.yml` | The "wait for an online runner" end of `deploy_runner.yml` | Same check in both modes |
| 10 | `display_summary.yml` | `display_summary.yml` | Mode-specific text |

Two shared changes apply to all of these:

- **TLS:** `validate_certs: false` is hardcoded today. It becomes "use `gitlab_ca_cert_path` if set, otherwise the system trust store". Hosted mode keeps using its own generated CA.
- **Output:** the project ID and project URL are set as facts for the next steps.

### 6.3 Hosted-only tasks (stay in `hosted_gitlab`)

| # | Task |
|---|---|
| 1 | `prereq_checks.yml` (GitLab host name check, SSH credentials) |
| 2 | `check_oim_prerequisites.yml` |
| 3 | `gitlab_passwordless_ssh` role (pin host key, bootstrap key, validate SSH) |
| 4 | `validate_prerequisites.yml` |
| 5 | `install_packages.yml` |
| 6 | `configure_firewall.yml` |
| 7 | `create_directories.yml` |
| 8 | `generate_tls_certs.yml` |
| 9 | `install_gitlab.yml` |
| 10 | `configure_gitlab.yml` |
| 11 | `root_password_change.yml` |
| 12 | Root token creation with `gitlab-rails` (the first part of `create_project.yml`) |
| 13 | `podman_login.yml` |
| 14 | `deploy_runner.yml` (the part that installs and registers the runner) |
| 15 | `cleanup_gitlab` role (install removal, services, runner, TLS) |
| 16 | The SSH-based parts of `pxe_buildstream_manager` (`gitlab_host.yml`) |

### 6.4 External-only tasks (new `external_gitlab` role)

| # | Task | Purpose |
|---|---|---|
| 1 | `main.yml` | Runs the external flow in order |
| 2 | `preflight.yml` | Checks 3 to 11 from section 4 (reachability, TLS, version, token scopes, namespace permission) |
| 3 | `create_or_adopt_project.yml` | Create the project in the namespace with the ownership marker, adopt it if the marker is there, stop if it is not |
| 4 | `verify_permissions.yml` | Checks 13 and 14 from section 4, once the project exists |
| 5 | `library/journal.py` | Setup journal (`external_gitlab_journal.json`): records resource IDs only (never secrets), atomic writes, file lock, recovers from a corrupt file |
| 6 | `compensate_on_failure.yml` | Undo this run's changes in reverse order. An adopted project is never deleted. |
| 7 | `cleanup_external_gitlab.yml` | Cleanup of Omnia-owned resources only |
| 8 | `store_credentials.yml` | Saves the trigger token and project ID in the vault file on the OIM |

### 6.5 Playbook changes

| File | Change |
|---|---|
| `setup_gitlab.yml` | Mode dispatch with `when: gitlab_mode == ...`. The `gitlab_server` SSH plays run only for hosted. For external, a `localhost` play runs the `external_gitlab` role. |
| `sync_gitlab_files.yml` | Mode dispatch as above |
| `cleanup_gitlab.yml` | Mode dispatch as above |
| `build_stream.yml` | No new tags. The existing `execute` and `cleanup` tags work for both modes. |

### 6.6 Count summary

| Group | Count | Notes |
|---|---|---|
| Existing roles shared by both modes (setup, validate, credentials, precheck, postgres, deploy_bsm) | 6 roles | Only validate and credentials change |
| Common GitLab tasks (shared role) | 10 tasks | Used by both hosted and external |
| Hosted-only tasks | 16 items | Install, SSH, runner deployment, hosted cleanup |
| External-only tasks | 8 items | Preflight, adopt, permission checks, journal, compensation, cleanup |
| **Tasks that run in the hosted flow** | **10 common + 16 hosted-only = 26** | |
| **Tasks that run in the external flow** | **10 common + 8 external-only = 18** | |

---

## 7. Other code that must change

These are not in the story, but the code depends on GitLab being hosted:

| Area | Problem today | Change |
|---|---|---|
| Cadence manager (`cadence_manager.py`) | Builds the URL from `gitlab_host` and the port. `gitlab_host` is validated as IPv4 only. Uses `gitlab_root_token` and `verify=False`. | Use `gitlab_external_url`, the `omnia-ci` project token and the CA path in external mode. |
| Token in credentials | Hosted `create_project.yml` writes `gitlab_root_token` into the vault file, and the watcher reads it. | In external mode the PAT is used only during setup, reconcile and cleanup. It is **not** stored for the watcher and **not** put into CI variables. The watcher uses the `omnia-ci` project token. |
| `pxe_buildstream_manager` (`gitlab_host.yml`, `upload_to_gitlab.yml`) | SSH to the GitLab host and read the root token. | API-only path in external mode, using the project token. |
| Trigger token | Written to `/root/.gitlab_trigger_token` on the GitLab host. | Stored on the OIM in the vault file in external mode. |
| `sync_gitlab_files.yml` | Reads `/root/.gitlab_root_token` from the GitLab host. | External variant uses the API. |
| Token expiry | The `omnia-ci` token expires in 360 days. | A warning when under 30 days, and automatic rotation when the playbook runs. |

---

## 8. Validation and schema changes

| File | Change |
|---|---|
| `build_stream_config.json` | Add `gitlab_mode`, `gitlab_external_url`, `gitlab_project_namespace`, `gitlab_ca_cert_path`. Add `if/then` rules: external mode requires URL and namespace, hosted mode requires `gitlab_host`. |
| `build_stream_config_validator.py` | `gitlab_host` is required only in hosted mode. The "BuildStream IP must differ from `gitlab_host`" check is hosted-only. Add the external checks: HTTPS URL, namespace format, CA file exists. |
| `build_stream_credentials.json` | `gitlab_api_token` required in external mode. `gitlab_root_password` and `gitlab_ssh_password` required in hosted mode. |
| `build_stream_messages.py` | New error messages |
| `cadence_manager.py` loader | Mode-aware (section 7) |

Backward compatibility: existing hosted deployments only need `gitlab_mode: self_hosted` added. Nothing else changes for them.

---

## 9. Security notes

- The PAT lives only in the vault file (root-owned, mode 0600). It is never in `build_stream_config.yml`, never in CI variables, never in logs (`no_log` on every task that handles it).
- The journal stores resource IDs only, never token values.
- Omnia removes or revokes only what it created. It never deletes a project it adopted, and never unregisters a runner it did not create.
- A project that exists without the Omnia ownership marker is never modified.
- CI variables holding secrets are masked and protected.

---

## 10. Decisions needed from the lead

1. **Role layout:** a new `external_gitlab` role plus a shared role (`gitlab_common` as a proposed name) as described, or the other options in the HLD? The story says "dispatch inside `hosted_gitlab`" and the HLD says "dispatch in `setup_gitlab.yml`". This design follows the HLD.
2. **Runner tags:** support only untagged runners (current ER), or add an optional `gitlab_runner_tags` input later?
3. **Cadence token:** confirm that the watcher uses the `omnia-ci` project token and that the PAT is never kept outside setup.
4. **Smoke test:** do we add an optional pipeline test that proves the runner can reach the OIM?
5. **Story scope:** the cadence, `pxe_buildstream_manager` and `sync_gitlab_files.yml` changes are not in the current story. Should they be added to that story or put in a new one?
