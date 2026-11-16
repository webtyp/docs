# Stage 3 — `gotest` fails a non-app repo that requires `appkit`

← [Stage 2](STAGE_2_APPKIT.md) | Next → [Stage 4](STAGE_4_MODULES.md)

Repo: `webtyp/devflow` (backend tooling: stdlib `os`, `path/filepath` are legitimate here; do NOT «fix» them).
The check lives in `gotest` because `gocheck` has no engine yet (PLAN «Design gate» 4).

## 3.1 Dependency

```bash
cd ~/Dev/Project/webtyp/devflow
go get webtyp.com/server@v0.2.127   # adds server; gobuild, gorun come indirect. Verified: server's graph has no devflow (no cycle)
go mod tidy
```

## 3.2 `gotest_appkit_gate.go` (new, verbatim — compiled and tested 2026-11-16)

```go
package devflow

import (
	"path/filepath"

	"webtyp.com/server"
)

// appkitModulePath is the app kit: it carries every implementation, so only an app may require it (ARCHITECTURE D62).
const appkitModulePath = "webtyp.com/appkit"

const (
	appkitGateLabel   = "appkit only in apps"
	appkitGateProblem = "go.mod requires webtyp.com/appkit but config/ declares no `var App`: only an app imports appkit — a module imports webtyp.com/appmodule (skill app-modules)"
)

// CheckAppkitOnlyInApps returns the problem when the repo at rootDir requires appkit without being an app; "" when it is fine.
// Example: if p := g.CheckAppkitOnlyInApps("."); p != "" { ... }
func (g *Go) CheckAppkitOnlyInApps(rootDir string) string {
	if !g.HasDependency(filepath.Join(rootDir, "go.mod"), appkitModulePath) || server.HasApp(rootDir) {
		return ""
	}
	return appkitGateProblem
}
```

## 3.3 Wire it into `gotest.go`, right after the «Domain tests index» block (today `gotest.go:437-448`)

```go
	// ARCHITECTURE D62: only an app (config/ declares `var App`) may require webtyp.com/appkit.
	if problem := g.CheckAppkitOnlyInApps(g.rootDir); problem != "" {
		addMsg(false, appkitGateLabel)
		testStatus = "Failed"
		g.consoleOutput(problem)
	}
```

Silent when it passes (like the tests index when `docs/TESTS.md` is absent): no new line in the summary of repos that never
required appkit.

## 3.4 `tests/gotest_appkit_gate_test.go` (new, verbatim — passes)

```go
package devflow_test

import (
	"os"
	"path/filepath"
	"testing"
)

const appkitRequireGoMod = "module github.com/veltylabs/some_module\n\ngo 1.26\n\nrequire webtyp.com/appkit v0.2.36\n"

type repoFile struct{ name, body string }

func writeRepo(t *testing.T, files ...repoFile) string {
	t.Helper()
	root := t.TempDir()
	for _, f := range files {
		path := filepath.Join(root, f.name)
		if err := os.MkdirAll(filepath.Dir(path), 0o755); err != nil {
			t.Fatal(err)
		}
		if err := os.WriteFile(path, []byte(f.body), 0o644); err != nil {
			t.Fatal(err)
		}
	}
	return root
}

func TestAppkitGate_ModuleRequiringAppkitFails(t *testing.T) {
	root := writeRepo(t, repoFile{"go.mod", appkitRequireGoMod})
	g := newGoHandlerWithMockBackup(t, &MockGitClient{})
	if got := g.CheckAppkitOnlyInApps(root); got == "" {
		t.Fatal("a repo without config/ var App that requires appkit must fail the gate")
	}
}

func TestAppkitGate_AppPasses(t *testing.T) {
	root := writeRepo(t,
		repoFile{"go.mod", appkitRequireGoMod},
		repoFile{"config/app.go", "package config\n\nvar App = 1\n"},
	)
	g := newGoHandlerWithMockBackup(t, &MockGitClient{})
	if got := g.CheckAppkitOnlyInApps(root); got != "" {
		t.Fatalf("an app may require appkit, got %q", got)
	}
}

func TestAppkitGate_RepoWithoutAppkitPasses(t *testing.T) {
	root := writeRepo(t, repoFile{"go.mod", "module github.com/veltylabs/some_module\n\ngo 1.26\n"})
	g := newGoHandlerWithMockBackup(t, &MockGitClient{})
	if got := g.CheckAppkitOnlyInApps(root); got != "" {
		t.Fatalf("got %q, want no problem", got)
	}
}
```

Rows in `devflow/docs/TESTS.md` (create the table if the file lacks it):

```markdown
| [TestAppkitGate_ModuleRequiringAppkitFails](../tests/gotest_appkit_gate_test.go) | a repo without `var App` that requires appkit fails gotest |
| [TestAppkitGate_AppPasses](../tests/gotest_appkit_gate_test.go) | an app may require appkit |
| [TestAppkitGate_RepoWithoutAppkitPasses](../tests/gotest_appkit_gate_test.go) | a repo without appkit is not touched |
```

## 3.5 Docs

`devflow/README.md`, section that lists what `gotest` checks: one line «appkit only in apps — a repo whose `go.mod`
requires `webtyp.com/appkit` and whose `config/` declares no `var App` fails (ARCHITECTURE D62)».

## Done when

`gotest` green in `devflow`. Then prove the gate **without installing it** (an installed `gotest` would turn every module
red before stage 4 fixes it):

```bash
cd ~/Dev/Project/webtyp/devflow && go build -o "$TMPDIR/gotest-gate" ./cmd/gotest
cd ~/Dev/Project/veltylabs/modules/business_calendar && "$TMPDIR/gotest-gate"   # must print «appkit only in apps» as failed
```

The new `gotest` is installed only at gate 18, after every module is fixed.
