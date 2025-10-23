# 🚀 Start kubectl Documentation Session

Copy the prompt below and paste it into a new Claude Code session:

---

## Prompt for New Session:

```
Continue the Kubernetes architecture documentation project for kubectl.

Read and analyze the progress tracking document at:
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/docs/architecture/claude/kubectl/PROGRESS.md

This file contains:
- Complete documentation plan (25 files organized in 5 phases)
- Quality standards from the completed kube-apiserver project
- Code structure reference (staging/src/k8s.io/kubectl/)
- Session tracking (5 sessions estimated)
- Instructions to analyze and improve the plan

Your tasks:
1. Read PROGRESS.md completely
2. Analyze the kubectl codebase (staging/src/k8s.io/kubectl/, cmd/kubectl/)
3. Improve the documentation plan based on actual code structure
4. Start creating documentation following the plan
5. Update PROGRESS.md continuously as you complete files

Begin with Phase 1 (Core Documentation - 4 files):
- 00-README.md
- 01-REQUIREMENTS.md
- 02-FUNCTIONAL-SPEC.md
- GLOSSARY.md

Follow the quality standards: 800-1000+ lines, 10-20 diagrams, extensive code references, real examples.

Update PROGRESS.md after each file and create SESSION-1-SUMMARY.md when done.
```

---

## Alternative Short Prompt:

```
Continue kubectl architecture documentation. Read the plan and instructions at:
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/docs/architecture/claude/kubectl/PROGRESS.md

Start with Phase 1 (4 core files). Update progress tracking as you work.
```

---

## What Makes kubectl Special:

**The CLI interface to Kubernetes!**

Covers:
- Command architecture (Cobra framework)
- Resource management (builder pattern, visitor pattern)
- Apply algorithm (three-way merge, strategic merge patch)
- Output formatting (table, YAML, JSON, custom columns, JSONPath)
- Streaming (logs, exec, port-forward)
- Plugin system

**Estimated**: 25 files, 5 sessions, 22,000+ lines, 210+ diagrams

---

**Ready when you are!** 🚀
