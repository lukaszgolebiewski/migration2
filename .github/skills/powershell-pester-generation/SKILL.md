# Pester Test Generation Skill

## Purpose
Generate Pester 5 test scaffolds for PowerShell functions and scripts in the repository. Tests validate function behavior, parameter handling, error conditions, and integration with external tools.

If the input is a script file rather than a function file, generate tests around the script's entry point or exported helper functions. If no callable entry point exists, state that no scaffold can be generated without additional context.

If the source file path does not exist or the specified function name is not found, stop and report the invalid input instead of generating a guessed test scaffold.

## When to Use
- After generating a new PowerShell function or module
- When modernizing existing code (to lock in behavior before refactoring)
- When adding test coverage to untested helpers

## Test Structure

### File Naming and Location
Generate the test file at `tests/{SourceFileName}.Tests.ps1` relative to the repository root. Dot-source the source file using a path relative to `$PSScriptRoot`, for example `. $PSScriptRoot/../helpers/{source-file}.ps1`. Before generating tests, verify that the dot-source path is correct; if it cannot be resolved, mark the path with a `# TODO: verify path` comment.

### Placeholder Replacement Rules
Use the actual input source file name for the dot-source path, the actual input function name for `Describe`, and replace example parameter names and expected values with real values from the source code. When values are unknown, use `# TODO` comments instead of guessing.

### Pester 5 Template
```powershell
BeforeAll {
    . $PSScriptRoot/../helpers/{source-file}.ps1
}

Describe 'Function-Name' {
    Context 'When called with valid parameters' {
        It 'Should return expected result' {
            # Arrange
            $params = @{ Param1 = 'value' }

            # Act
            $result = Function-Name @params

            # Assert
            $result | Should -Be 'expected'
        }
    }

    Context 'When called with invalid parameters' {
        It 'Should throw on missing mandatory parameter' {
            { Function-Name } | Should -Throw
        }
    }

    Context 'When external tool fails' {
        It 'Should throw with descriptive error' {
            # Mock external tool
            Mock terraform { throw 'mock failure' }

            { Function-Name -Path 'test' } |
                Should -Throw '*failed*'
        }
    }
}
```

## Test Categories for This Repository

Generate tests in this priority order. Only include categories that are relevant to the input file; do not add unrelated categories.

1. **Parameter validation** — Mandatory params, ValidateSet, types → Direct invocation
2. **Happy path** — Expected behavior with valid inputs → Direct invocation or mocks
3. **Error handling** — `try/catch`, `$LASTEXITCODE` → Mock failures
4. **External tools** — `terraform`, `az`, `hcl2json` → Mock with known outputs
5. **Template processing** — `select-config` replacement logic → Temp files with known content
6. **File operations** — `Get-Content`, `Set-Content` → `TestDrive:\` temp files
7. **Security context** — `update-security-context` → Mock Key Vault responses

## Inputs
- Source file path (required)
- Function name(s) to test (required). If multiple names are provided, generate one `Describe` block per function. If no names are provided, generate tests for all exported functions in the source file.
- Known external dependencies (explicit list only): `terraform`, `az`, `hcl2json`, or other commands referenced in the source file. If a referenced external command is not listed in the input, use a placeholder mock with a `# TODO: verify mock` comment instead of inventing one.

## Outputs
- Complete `.Tests.ps1` file placed at `tests/{SourceFileName}.Tests.ps1`
- Test run instructions (e.g., `Invoke-Pester -Path ./tests/`)
