---
name: bug-seer
version: 1.2.0
description: Predictive analytics engine that identifies bug hotspots using code pattern analysis and historical defect data
maintainer: OpenClaw Team
tags:
  - predictive-analytics
  - bug-prediction
  - code-quality
  - machine-learning
  - defect-analysis
  - static-analysis
type: analysis
entrypoint: bug-seer
dependencies:
  - name: scikit-learn
    version: ">=1.3.0"
    purpose: ML models for defect prediction
  - name: pandas
    version: ">=2.0.0"
    purpose: Data manipulation and feature engineering
  - name: numpy
    version: ">=1.24.0"
    purpose: Numerical operations
  - name: tree-sitter
    version: ">=0.20.0"
    purpose: AST parsing for code structure analysis
  - name: radon
    version: ">=5.1.0"
    purpose: Cyclomatic complexity metrics
  - name: lizard
    version: ">=1.17.0"
    purpose: Code complexity and length analysis
  - name: gitpython
    version: ">=3.1.0"
    purpose: Git history mining for defect commits
system_requirements:
  python: ">=3.9"
  memory_min_mb: 512
  disk_space_mb: 100
---

# Bug Seer: Predictive Defect Analysis

## Purpose

Bug Seer analyzes codebases to predict likely bug locations by combining:
- Historical defect data from git commit history (bug-fix commits)
- Code complexity metrics (cyclomatic complexity, code churn)
- Code smell detection (long methods, duplicated code, god classes)
- Developer activity patterns and ownership hotspots
- Recent changes and code age

Real use cases:
- **Pre-release risk assessment**: Identify high-risk files before tagging a release
- **Code review prioritization**: Flag files needing extra review attention
- **Refactoring roadmap**: Find technical debt hotspots for improvement sprints
- **Team allocation**: Identify modules with high defect density that need experienced developers
- **Onboarding focus**: Show new hires which areas are most fragile

## Scope

Bug Seer provides these commands:

### `bug-seer analyze [PATH]`
Analyze codebase to predict bug hotspots
- `--model <name>`: Use trained model (default: 'default', 'ensemble', or path to custom .pkl)
- `--threshold <float>`: Risk score threshold for warnings (0.0-1.0, default: 0.7)
- `--format <json|table|sarif>`: Output format (default: table)
- `--git-depth <int>`: Number of git commits to analyze (default: 1000)
- `--features <file>`: JSON file with custom feature weights
- `--exclude <pattern>`: Glob pattern to exclude paths (can repeat)
- `--output <file>`: Write report to file instead of stdout

### `bug-seer train [DATASET]`
Train prediction model on historical data
- `--git-repo <path>`: Mine git history for defects if dataset not provided
- `--label-bug-keywords <keywords>`: Comma-separated keywords in commit messages (default: "fix,bug,defect,crash,error")
- `--test-split <float>`: Train/test split ratio (0.0-1.0, default: 0.2)
- `--model-type <rf|xgb|logistic>`: Algorithm (default: rf)
- `--cross-validate`: Perform k-fold cross-validation
- `--save <path>`: Save trained model to path
- `--feature-importance`: Show feature importance scores

### `bug-seer validate [MODEL]`
Validate model performance
- `--metrics <all|precision|recall|f1|roc>`: Metrics to calculate
- `--baseline <model>`: Compare against baseline model
- `--confusion-matrix`: Show confusion matrix
- `--threshold-analysis`: Scan across thresholds

### `bug-seer explain [FILE]`
Explain prediction for a specific file
- `--model <name>`: Model to use
- `--detail <high|medium|low>`: Explanation depth
- `--output <json|text>`: Format

### `bug-seer history [PATH]`
Show historical bug patterns for a file/directory
- `--commits <int>`: Number of past commits (default: 50)
- `--include-blame`: Include git blame for recent fixes

### `bug-seer list-models`
List available trained models

### `bug-seer import-defects [FILE]`
Import external defect dataset (CSV/JSON with: file_path, defect_type, severity, commit_hash)

## Work Process

### 1. Feature Extraction Phase
For each source file in the codebase:
- Parse AST using tree-sitter for language-specific analysis
- Calculate metrics:
  * Cyclomatic complexity (radon)
  * Lines of code, comment ratio, function count
  * Number of parameters per function
  * Duplication percentage (using lizard)
  * Code churn (git log --oneline --numstat)
  * Bug fix frequency (git log --grep="fix" for that file)
  * Developer count (number of distinct authors)
  * Code age (days since last modification)
  *最近的修改频繁度 (commits in last 30 days)
  * Ownership concentration (% of changes by top contributor)

### 2. Historical Defect Mining
Bug Seer mines git repository to identify past defect commits:
```bash
git log --all --grep="fix" --pretty=format:"%H" > bug_commits.txt
git log --all --grep="bug" --pretty=format:"%H" >> bug_commits.txt
```
Then maps each defect commit to affected files:
```bash
for commit in $(cat bug_commits.txt); do
  git show --name-only --pretty=format: "$commit" | tail -n+2
done > defect_files.txt
```
Each file touched by a defect commit gets a positive label.

### 3. Model Training
- Load historical dataset (mined or provided)
- Perform feature engineering (normalize, PCA for dimensionality)
- Split into train/test sets
- Train RandomForest classifier (default) or XGBoost
- Evaluate using precision@k (top-k ranked files)
- Save model and feature importance to `~/.openclaw/bug-seer/models/`

### 4. Prediction Phase
- Extract same features for current codebase
- Load trained model
- Score each file (probability of containing defects)
- Rank files by score
- Apply threshold to filter
- Generate report with risk scores, contributing factors, and actionable recommendations

### 5. Report Generation
Output includes:
- Risk score (0-1) for each file
- Rank percentile
- Top 3 contributing features (e.g., "high complexity", "frequent changes")
- Suggested actions ("refactor", "increase review", "add tests")
- Historical defect count for that file

## Golden Rules

1. **Don't overfit to historical patterns**: Models trained on past data may miss new defect types. Validate against recent findings.
2. **Require sufficient history**: Files with < 6 months of history produce unreliable predictions. Mark as "insufficient data".
3. **Language-specific thresholds**: Complexity thresholds differ by language (Python vs C++). Use calibrated defaults per language.
4. **False positive budget**: Limit warnings to top 20% of files to avoid alert fatigue.
5. **Always cite evidence**: Every high-risk flag must include specific metric (e.g., "cyclomatic complexity=42 (>35 threshold)").
6. **Preserve git context**: Predictions must be reproducible from the exact commit SHA. Store model version with commit hash.
7. **Never modify code**: Bug Seer is read-only analysis. No automated fixes.
8. **Weight recent data higher**: Weight defects from last 90 days more heavily in training (factor of 2x).
9. **Account for file size**: Large files naturally have more defects. Normalize by lines of code.
10. **Include both likelihood and impact**: A file with high complexity and high churn gets higher risk than one with only one factor.

## Examples

### Quick analysis of current project
```bash
bug-seer analyze src/
```
Output (table format):
```
File                              Risk  Rank  Top Factors                              Historical Defects
src/game/combat.py                0.82  1%    Complex (CC=45), Frequent changes      12
src/engine/renderer.cpp           0.78  3%    High churn, Many authors               8
src/utils/helpers.py              0.41  45%   Medium complexity                      3
tests/test_core.py                0.12  89%   Low complexity                         0
```

### Train custom model for team standards
```bash
bug-seer train --git-repo . --model-type xgb --save models/team-v2.pkl --feature-importance
```
Output:
```
Training on 847 files (152 defect samples)
Features: ['cc', 'loc', 'churn', 'dev_count', 'bug_fixes', 'duplication', 'age_days']
Best CV F1: 0.73 (threshold=0.6)
Feature importance:
  1. churn (0.31)
  2. cc (0.24)
  3. bug_fixes (0.18)
  4. dev_count (0.12)
Model saved to models/team-v2.pkl
```

### CI/CD integration with SARIF output
```bash
bug-seer analyze . --format sarif --threshold 0.75 --output warnings.sarif
```
SARIF snippet:
```json
{
  "level": "error",
  "message": {"text": "High bug risk (0.82). Contributing: Cyclomatic complexity 45 (>35), 5 authors, 47 changes in 30 days"},
  "locations": [{"physicalLocation": {"artifactLocation": {"uri": "src/game/combat.py"}}}]
}
```

### Explain specific file prediction
```bash
bug-seer explain src/engine/renderer.cpp --model ensemble --detail high
```
Output:
```
File: src/engine/renderer.cpp
Predicted risk: 0.78 (High)

Contributing factors (weighted):
  × Code churn: 47 commits in 30 days (normalized score: 0.42)
  × Cyclomatic complexity: 38 (threshold: 25) (score: 0.31)
  × Developer count: 7 distinct authors (score: 0.18)
  - Lines of code: 1250 (acceptable)
  - Bug history: 8 defects (normalized: 0.15)

Recommendation: Schedule dedicated code review session and consider extracting shader management subsystem.
```

### Import external defect tracking data
```bash
bug-seer import-defects jira-export.csv
```
CSV format:
```csv
file_path,defect_type,severity,commit_hash
src/auth/login.py,security,critical,abc123
src/api/user.py,logic,major,def456
```

### Analyze with custom feature weights
```bash
cat > custom_weights.json <<EOF
{
  "cyclomatic_complexity": 2.5,
  "code_churn": 1.8,
  "developer_count": 1.2
}
EOF
bug-seer analyze . --features custom_weights.json
```

### Validate model before production use
```bash
bug-seer validate models/team-v2.pkl --metrics precision,recall,f1 --confusion-matrix
```
Output:
```
Model: models/team-v2.pkl (trained 2026-01-15)

Confusion Matrix (threshold=0.6):
          Predicted
          Yes   No
Actual Yes  42   8
       No   15  782

Precision: 0.74 (42/57)
Recall:    0.84 (42/50)
F1:        0.79
```

## Dependencies

### Python Packages
Install via pip:
```bash
pip install scikit-learn pandas numpy tree-sitter radon lizard gitpython
```

For language-specific parsers:
```bash
# Python support
pip install tree-sitter-python

# C/C++ support
pip install tree-sitter-c

# JavaScript/TypeScript
pip install tree-sitter-javascript tree-sitter-typescript
```

### System Dependencies
- `git`: Available in PATH for history mining
- `python3.9+`: With pip
- 500MB RAM minimum for medium codebases

### Data Requirements
- Git repository with at least 100 commits for reliable training
- Minimum 30 defect-labeled files for meaningful model
- 6 months of history recommended

## Environment Variables

- `BUG_SEER_MODEL_DIR`: Override default model storage location (~/.openclaw/bug-seer/models)
- `BUG_SEER_CACHE`: Enable/disable caching (0=disabled, 1=enabled, default: 1)
- `BUG_SEER_JOBS`: Number of parallel feature extraction processes (default: CPU count)
- `BUG_SEER_GIT_TIMEOUT`: Seconds for git operations (default: 30)
- `OPENCLAW_DEBUG`: Set to 1 for verbose logging

Example:
```bash
export BUG_SEER_MODEL_DIR=/custom/models
export BUG_SEER_JOBS=4
bug-seer analyze .
```

## Verification Steps

After installing Bug Seer, verify correct operation:

1. **Check installation**:
```bash
bug-seer --version
# Should print: bug-seer 1.2.0
```

2. **Verify dependencies**:
```bash
bug-seer check-deps
# Output: All dependencies satisfied (tree-sitter, radon, lizard, git)
```

3. **Test on small codebase**:
```bash
cd /path/to/small/project
bug-seer analyze . --format json --output test-report.json
# Should produce JSON without errors
cat test-report.json | python3 -m json.tool  # Validate JSON
```

4. **Validate model training** (requires git history):
```bash
bug-seer train --git-repo . --test-split 0.3 --cross-validate
# Should report CV F1 score > 0.6 with default model
```

5. **Check model persistence**:
```bash
bug-seer list-models
# Should show at least 'default' if training succeeded
```

6. **CI integration test**:
```bash
bug-seer analyze . --format sarif --threshold 0.8 > sarif-output.txt
# Should exit code 0 even with no findings (no findings = valid empty SARIF)
```

## Troubleshooting

### "No git repository found"
**Cause**: Running in directory without git history.
**Fix**: cd to git repo or specify path: `bug-seer analyze /path/to/repo`

### "Insufficient defect samples for training"
**Cause**: Less than 20 defect files identified in git history.
**Fix**: Increase `--git-depth` or adjust `--label-bug-keywords` to capture more defect commits. Manually label defects via `import-defects`.

### "Tree-sitter language not supported"
**Cause**: Missing language parser for file extension.
**Fix**: Install parser: `pip install tree-sitter-<language>`. Check support: `bug-seer languages`

### "Memory exhausted during analysis"
**Cause**: Very large codebase without parallel limits.
**Fix**: Set `BUG_SEER_JOBS=2` or use `--exclude` to skip directories. Consider analyzing in batches.

### "Model predictions are all 0.0"
**Cause**: Model not trained or feature mismatch.
**Fix**: Train model first: `bug-seer train --git-repo .`. Ensure model path exists and matches codebase language.

### "Cyclomatic complexity parsing failed"
**Cause**: radon cannot parse certain syntax.
**Fix**: Upgrade radon: `pip install --upgrade radon`. Check syntax errors in problematic files manually.

### "Feature importance shows negative values"
**Cause**: PCA or normalization produced non-intuitive weights.
**Fix**: Disable feature scaling: use `--no-normalize` during train (not recommended) or inspect raw metrics.

### "Analysis stuck on one file"
**Cause**: Massive single file or parser timeout.
**Fix**: Increase `--git-timeout` or exclude file: `--exclude "**/generated/**"`

## Rollback Commands

Bug Seer is analysis-only and makes no modifications. Rollback refers to reverting model changes or analysis state.

### Remove specific model
```bash
rm ~/.openclaw/bug-seer/models/<model-name>.pkl
rm ~/.openclaw/bug-seer/models/<model-name>.json  # metadata
```

### Restore default model
```bash
bug-seer delete-model team-custom
# Next analysis will fall back to 'default'
```

### Clear analysis cache
```bash
rm -rf ~/.openclaw/bug-seer/cache/*
# Next analysis will recompute all features
```

### Revert to previous model version (if kept)
```bash
# Models are snapshotted with timestamps
ls ~/.openclaw/bug-seer/models/backups/
bug-seer use-model models/backups/team-v1-20260101.pkl
```

### Undo dataset import
```bash
# Manual: edit defect database if you know commit hashes
# Or retrain without the imported data:
bug-seer train --git-repo . --exclude-defects jira-export.csv
```

### Disable Bug Seer in CI/CD
Remove the `bug-seer analyze` command from your CI script. No persisted state.

## Advanced: Feature Engineering Details

Bug Seer computes these features per file:

| Feature | Calculation | Weight (default) | Rationale |
|---------|-------------|------------------|-----------|
| `cc` | avg cyclomatic complexity (radon) | 1.0 | High complexity → harder to understand/test |
| `loc` | lines of code | -0.3 (neg) | Larger surface area but also more tested |
| `churn` | commits in last 30 days | 1.5 | Churn indicates instability/ongoing work |
| `dev_count` | distinct authors (git) | 1.2 | Many authors → coordination issues |
| `bug_fixes` | number of defect commits touching file | 2.0 | Past bugs predict future bugs |
| `duplication` | % duplicated lines (lizard) | 0.8 | Copy-paste carries over defects |
| `age_days` | days since creation | -0.4 (neg) | Older code generally more stable |
| `params_avg` | average function parameters | 0.5 | Long parameter lists are error-prone |
| `comment_ratio` | comment lines / total lines | -0.2 (neg) | Well-documented code has fewer bugs |
| `test_coverage` | if tests/ exists and test files found | -1.0 (neg) | Tested code is more reliable |

Feature weights can be overridden via `--features` JSON file.

## Integration Examples

### Pre-commit hook (warn on risky files)
```bash
#!/bin/bash
# .git/hooks/pre-commit
CHANGED=$(git diff --name-only --cached)
echo "$CHANGED" | xargs bug-seer explain --detail low --threshold 0.75
# Exit 0 to allow commit; non-zero would block (not recommended)
```

### GitHub Actions (comment on PR)
```yaml
- name: Bug Seer Analysis
  run: |
    bug-seer analyze . --format sarif --threshold 0.8 > bugs.sarif
    # Upload as security scanning results
- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: bugs.sarif
```

### VS Code Extension (hypothetical)
The extension runs `bug-seer explain` on current file and shows inline risk badge.

## Performance Characteristics

- **Small codebase** (< 100 files): ~2-5 seconds
- **Medium codebase** (100-1000 files): ~30-60 seconds
- **Large codebase** (> 1000 files): ~2-5 minutes with parallel feature extraction
- Training time: 10-30 seconds after feature extraction
- Cache hit: Subsequent runs on unchanged files near-instant

## Limitations

- **Language support**: Fully supported: Python, C/C++, Java, JavaScript/TypeScript, Go. Partial: Ruby, PHP, Swift (AST only, no complexity metrics).
- **Monorepo detection**: May double-count if same file appears under multiple roots. Use `--exclude` to avoid.
- **Binary files**: Skipped automatically (no parser).
- **Generated code**: Must exclude via glob patterns (e.g., `--exclude "**/dist/**"`).
- **Non-git VCS**: Not supported. Requires git history.
- **Real-time**: Not suitable for real-time suggestions; suited for periodic analysis.

## Future Enhancements

- Inclusion of code review comments text analysis
- Embodied defect prediction (using code embeddings from LLMs)
- Cross-project transfer learning
- Time-series prediction of defect introduction rates
- Integration with issue tracker (Jira, GitHub Issues) for richer labels
```