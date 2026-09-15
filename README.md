# GitHub Advanced Security backend demo

This repository is a safe, disposable demonstration for:

- Secret scanning and push protection
- Code scanning with CodeQL
- Dependency review
- Copilot cloud agent-assisted backend test coverage

The `main` branch is the clean baseline. The `demo/*` branches contain intentional,
synthetic examples. Do not copy any value from this repository into a real system,
and do not merge the demonstration branches into production code.

## Repository setup

The examples use Python 3.12, Flask, SQLite, pytest, and GitHub Actions.

1. Open **Settings → Advanced Security**.
2. Enable **Dependency graph**, **Dependabot alerts**, **Dependabot security updates**,
   **Secret scanning**, and **Push protection** where your licence supports them.
3. Open **Security and quality → Code scanning**.
4. Select **Set up CodeQL** and use the generated or committed CodeQL workflow.
5. Open **Actions** and confirm that **Backend CI**, **CodeQL**, and **Dependency
   review** can run.

Private repositories may require GitHub Advanced Security or GitHub Secret Protection
to be enabled for the organisation. Public repositories can use the applicable
public-repository security features without exposing real credentials.

## Application reference

The demo application is a small Flask service with an in-memory SQLite database.
Start it locally after installing the requirements:

```bash
flask --app app.main run
```

The service exposes these endpoints:

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/health` | Returns `{"status": "ok"}`. |
| `GET` | `/users?name=<value>` | Searches the sample users by name. The `name` parameter is required and must be 50 characters or fewer. |

Example requests:

```bash
curl http://127.0.0.1:5000/health
curl 'http://127.0.0.1:5000/users?name=Ada'
```

The application is intentionally simple and resets its sample data for each
request. It is not intended for production use.

## Project layout

| Path | Purpose |
| --- | --- |
| `app/main.py` | Flask application and sample API endpoints |
| `tests/test_main.py` | Backend tests |
| `demo/` | Isolated fixtures for security demonstrations |
| `.github/workflows/` | Backend CI, CodeQL, and dependency review workflows |

## Demo branches

| Branch | Demo |
| --- | --- |
| `demo/secret-scanning` | A synthetic value for a custom secret-scanning pattern |
| `demo/codeql-sql-injection` | An intentionally unsafe SQL query |
| `demo/dependency-review` | A dependency downgrade that should produce a review finding |

Each branch is deliberately separate from `main`, so the audience can see a pull
request fail, then see the fix pass.

## Demo 1: secret scanning and push protection

The `demo/secret-scanning` branch contains the documented AWS example access key.
It is not a live credential:

```text
AKIAIOSFODNN7EXAMPLE
```

1. Open **Settings → Advanced Security** and confirm **Secret scanning** and
   **Push protection** are enabled.
2. Open **Security and quality → Secret scanning** and check for the alert from
   `demo/secret-scanning`.
3. If the placeholder is not detected, enable **Generic patterns** in
   **Settings → Advanced Security** when that option is available. Repository
   custom patterns and deterministic demo alerts may require GitHub Secret
   Protection in an organisation.

To demonstrate push protection:

```bash
git switch main
git pull --ff-only origin main
git switch -c demo/secret-push-blocked
printf '%s\n' 'AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE' > demo-secret.env
git add demo-secret.env
git commit -m "demo: add synthetic credential"
git push -u origin demo/secret-push-blocked
```

GitHub should reject the push when the provider or generic pattern is available.
Show the detected pattern and remediation link. If the documented example is
treated as a known test value, use a temporary provider-issued test credential
only in an untracked local file, revoke it immediately after the demonstration,
and never commit it to this repository.
Then remove the file, commit, and push again:

```bash
git rm demo-secret.env
git commit -m "fix: remove synthetic credential"
git push
```

For a real credential, revoke or rotate it immediately. Never bypass push
protection for a real secret.

## Demo 2: CodeQL

1. Open **Pull requests → New pull request**.
2. Select `main` as the base and `demo/codeql-sql-injection` as the compare branch.
3. Create the pull request.
4. Open the **Checks** tab and select **CodeQL** or **Code scanning results**.
5. Open the alert and show the source-to-sink data flow and CWE information.

The branch changes a parameterised SQLite query to an unsafe f-string query.
Restore the parameterised query in `app/main.py`:

```python
rows = db.execute(
    "SELECT id, name FROM users WHERE name LIKE ?",
    (f"%{name}%",),
).fetchall()
```

Commit the fix:

```bash
git add app/main.py
git commit -m "fix: parameterise user search query"
git push
```

Return to the pull request and show the new scan result.

## Demo 3: dependency review

1. Open a pull request from `demo/dependency-review` to `main`.
2. Open the **Checks** tab and select **Dependency review**.
3. Show the dependency name, old and new versions, advisory, and severity.
4. Open **Actions → Dependency review → the run → Summary** to show the job
   summary.

The branch changes `requests` from the current safe baseline to an old version
with known advisories. The exact advisory set can change, so show the advisory
reported by the GitHub Advisory Database at demo time.

Restore the safe version in `requirements.txt`, then commit:

```bash
git add requirements.txt
git commit -m "fix: upgrade vulnerable dependency"
git push
```

## Demo 4: Dependabot vulnerability and malware alerts

These two manifests are intentionally isolated from the application:

- `demo/vulnerable/requirements.txt` contains `requests==2.19.1`, which has
  known security advisories.
- `demo/malware/package.json` contains `spoint@0.1.700`, which is listed in the
  GitHub Advisory Database as malware under `GHSA-72h3-pwwh-68cx`.

Do not run `pip install` or `npm install` against either demo directory. The
packages are declared only so the dependency graph can identify them.

1. Open **Settings → Advanced Security** and confirm **Dependabot alerts** and
   **Dependabot security updates** are enabled.
2. Open **Security and quality → Dependabot**.
3. Open the vulnerability alert and show the package, severity, advisory, and
   fixed version when one exists.
4. Open the malware alert and show the malware classification and advisory.
5. Use the files in `demo/vulnerable` and `demo/malware` to explain that
   dependency alerts are generated from manifests, not from running packages.

The `.github/dependabot.yml` file has separate update entries for these
directories so Dependabot can maintain the fixtures without changing the safe
runtime dependencies. Malware alerts currently apply to npm packages, so the
malware fixture intentionally uses `package.json`.

## Demo 5: Copilot agent-assisted coverage

The baseline tests intentionally omit the validation and no-results branches in
`app/main.py`.

1. Open **Actions → Backend CI**.
2. Open the latest run and show the `pytest-cov` output and `coverage.xml` artifact.
3. Open **Issues → New issue**.
4. Use this title: `Improve backend test coverage for user search`.
5. Assign the issue to Copilot or start a task from **Copilot → Agents → Cloud
   agent**, if that entry point is enabled for your organisation.

Use this task text:

```text
Review the backend user-search endpoint and add focused tests for its uncovered
branches. Cover the missing name response, the name-too-long response, and the
no-results response. Use the existing pytest and coverage commands. Do not change
production behaviour, lower a coverage threshold, add external network calls, or
use real credentials. Keep the tests deterministic. Report the before and after
coverage in the pull request.
```

Review the resulting pull request as normal:

- Confirm that the tests are meaningful and mostly under `tests/`.
- Confirm that no credentials or external network calls were added.
- Open **Checks → Backend CI** and compare the coverage output.
- Download the `backend-coverage` artifact if the detailed report is needed.

Agent-generated code still requires developer review and approval.

## Local commands

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pytest --cov=app --cov-report=term-missing --cov-report=xml:coverage.xml
```

## Safety

All credentials in this repository are synthetic test strings. The intentionally
unsafe code and dependency are isolated on demo branches. Use a disposable
repository, use synthetic data only, and delete the repository after the demo if
it is no longer needed.
