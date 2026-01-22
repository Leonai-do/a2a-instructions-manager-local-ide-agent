# POISE Agent: Quick Reference Card

**Version:** 1.0 | **Date:** Jan 22, 2026 | **For:** Practitioners & Implementers  
A one-page cheat sheet for working with the POISE Documentation Planner Agent

---

## 🎯 In 30 Seconds

**POISE = Documentation Conflict Resolver**

- Upload docs → Detect contradictions → Propose fixes → Get execution plan → Review & approve → Executor implements
- Uses semantic analysis + linting + dependency mapping to find contradictions
- Separates planning from execution (human approval in between)
- Communicates with executor agents via A2A protocol (JSON-RPC 2.0)

---

## 🚀 Get Started

### 1. Upload Documentation
```
"Analyze my documentation for conflicts"
→ Select files (.md, .txt, .pdf, .docx)
→ System parses & detects conflicts
```

### 2. Review Detected Conflicts
```
System shows each conflict:
├─ CONFLICT TYPE: [Factual | Temporal | Scope | Gap]
├─ SEVERITY: [LOW | MEDIUM | HIGH]
├─ DOCUMENTS INVOLVED: [file1.md, file2.md]
├─ CONFIDENCE: [0.0-1.0]
├─ AUTHORITY RANKING: [doc1:10 > doc2:5]
└─ RECOMMENDATION: [action to take]
```

### 3. Approve Plan
```
"Generate resolution plan"
→ Review proposed changes
→ Modify if needed
→ "Execute plan"
```

### 4. Get Output
```
✅ Updated documentation files
✅ Migration guides
✅ Deprecation notices
✅ Validation report
```

---

## 🔍 Conflict Types

| Type | Looks Like | Fix |
|------|-----------|-----|
| **Factual** | "Rate limit 100/s" vs "1000/s" | Use authoritative source |
| **Temporal** | "Deploy staging→prod" vs "direct→prod" | Extract dependency graph |
| **Scope** | "Requires Node 16" (service A only) | Add scope statement |
| **Gap** | Auth documented, authz missing | Flag for completion |
| **Terminology** | "Node.js" vs "NodeJS" vs "Node" | Standardize term |
| **Links** | Reference to `/docs/old.md` (deleted) | Update or remove |
| **Versioning** | Docs say "MongoDB 3.6", code uses 5.0 | Update docs to 5.0 |

---

## 🏛️ Authority Levels (SSOT Hierarchy)

**Use these to classify your documentation:**

```
┌─────────────────────────────────────────────────────┐
│ LEVEL 10: AUTHORITATIVE                            │
│ - Official API contract, schema definitions         │
│ - Synchronized with code (updated by developers)    │
│ - Single source of truth                           │
├─────────────────────────────────────────────────────┤
│ LEVEL 7: REFERENCE IMPLEMENTATION                  │
│ - Working code examples, runnable tests             │
│ - Quarterly review by maintainers                  │
├─────────────────────────────────────────────────────┤
│ LEVEL 5: GUIDANCE                                   │
│ - Best practices, architectural rationale           │
│ - Annual review by domain experts                  │
├─────────────────────────────────────────────────────┤
│ LEVEL 2: DISCUSSION                                │
│ - Exploratory notes, RFCs, decision logs           │
│ - Ad-hoc, archived after resolution               │
└─────────────────────────────────────────────────────┘

When conflicts found:
Higher authority level wins.
Lower levels updated to reference higher level.
```

---

## 🛠️ Conflict Detection Methods

### 1. Semantic Role Labeling (SRL)
```
"API throttles at 100 requests per second"
→ Extract: [Agent: API] [Action: throttles] [Object: requests] [Rate: 100/s]

"Service handles 1000 req/s"
→ Extract: [Agent: Service] [Action: handles] [Object: requests] [Rate: 1000/s]

→ CONFLICT: Different rate limits for same concept
```

### 2. Vale Linting Rules
```
Check for:
- Inconsistent terminology ("Node.js" vs "NodeJS")
- Deprecated commands in docs
- Style violations
- Missing required sections
```

### 3. Link Validation
```
Check all markdown references:
- [link text](../path/to/file.md#anchor) → file must exist
- External URLs → must resolve
- Report broken links immediately
```

### 4. Dependency Mapping
```
Build graph of claims:
  "Deploy to staging first" → "Run tests" → "Deploy to prod"
  
If different docs say:
  "Deploy direct to prod" (skips testing)
  
→ CONFLICT: Missing intermediate step (circular logic? temporal inconsistency?)
```

---

## 📋 Execution Plan Phases

When you approve fixes, system generates 3-4 phases:

### PHASE 1: Consolidation
```
- Extract duplicated content to shared file
- Update all references to link instead of copy
- Mark removed sections as @deprecated
```

### PHASE 2: Enrichment  
```
- Add authority level tags to sections
- Add cross-reference links
- Add deprecation notices with migration guide
```

### PHASE 3: Validation
```
- Check all links resolve
- Verify no new contradictions introduced
- Confirm deprecation notices visible
```

### (Optional) PHASE 4: Archive
```
- Move old docs to /archive/deprecated/
- Set up HTTP redirects (301)
- Maintain historical versions in git tags
```

---

## 💬 A2A Protocol Basics (For Executor Integration)

**Manager → Executor communication:**

```json
{
  "jsonrpc": "2.0",
  "id": "msg_001",
  "method": "execute_plan",
  "params": {
    "task_id": "task_20260122_001",
    "correlation_id": "session_workflow_1",
    "checksum": "sha256_hash",
    "timestamp": "2026-01-22T12:54:00Z",
    "timeout_ms": 30000,
    "execution_context": {
      "phases": [ /* plan phases */ ]
    }
  }
}
```

**Key fields:**
- `task_id`: Unique job identifier
- `correlation_id`: Links all messages in workflow
- `checksum`: SHA-256 hash (integrity check)
- `timeout_ms`: Max execution time
- `execution_context`: The actual work (resolution plan)

**Executor → Manager response:**

```json
{
  "jsonrpc": "2.0",
  "id": "msg_001",
  "result": {
    "status": "completed|in_progress|pending_input",
    "data": { /* results */ }
  }
  // OR
  "error": {
    "code": 504,
    "message": "Timeout",
    "data": {
      "error_type": "transient",
      "retry_eligible": true
    }
  }
}
```

---

## ⚙️ Configuration (Authority Matrix)

**Set up in `.poise-config.json`:**

```json
{
  "authority_levels": {
    "api-spec.md": 10,
    "implementation.md": 7,
    "guide.md": 5,
    "discussion/": 2
  },
  "conflict_thresholds": {
    "high_confidence": 0.85,
    "medium_confidence": 0.65,
    "low_confidence": 0.40
  },
  "linting_rules": {
    "style_guide": "vale.ini",
    "custom_rules": ["rule1.yaml", "rule2.yaml"]
  },
  "deprecation_stages": {
    "warning_weeks": 2,
    "redirect_weeks": 2,
    "archive_weeks": 1
  }
}
```

---

## 🔧 Common Tasks

### "My docs contradict each other"
```
1. Upload all docs to POISE
2. Review conflict report
3. Approve resolution plan
4. Copy-paste A2A message to executor
5. Download updated docs
6. Commit to git, push to main
```

### "A tech requirement changed"
```
1. Upload updated docs + code examples
2. POISE detects version mismatch
3. Approves propagating change everywhere
4. Executor updates all references
5. Git commit lists all affected files
```

### "New docs don't match old docs"
```
1. Add new docs to repository
2. POISE detects new contradictions immediately
3. Forces resolution before merge (CI gate)
4. Prevents confusion for downstream agents
```

### "Deprecate a library version"
```
1. Create deprecation notice with timeline
2. POISE identifies all affected docs
3. Generates migration guide
4. Updates links (new version → old version migration path)
5. Archives old version after timeline expires
```

---

## 📊 Metrics to Track

**Document Health Score:**
```
DDR (Documentation Debt Ratio) = 
  (Hours to fix docs) / (Total dev hours) × 100%

Healthy: < 2% (less than 1 day/week)
Monitor: 2-5% (noticeable friction)
Critical: > 5% (more than 1 day/week fixing docs)
```

**Contradiction Detection:**
```
Before POISE: 3-5 major contradictions discovered per quarter (after problems)
After POISE: 100% discovered proactively (within 10 min of doc change)
```

**Execution Agent Success:**
```
Before POISE: 12% task failure rate (due to conflicting docs)
After POISE: <0.5% task failure rate
```

---

## 🎯 Best Practices

### DO ✅
- Mark documentation with authority levels upfront
- Run POISE analysis on every doc PR (CI/CD integration)
- Review conflict reports carefully (don't auto-approve)
- Keep decision logs (decisions.md) alongside code
- Archive deprecated docs (don't delete)
- Version documentation with code (same git tag)

### DON'T ❌
- Mix authority levels in single file (split into levels)
- Ignore low-confidence conflicts (might catch real issues)
- Copy-paste docs (use transclusion: `docs/shared/auth.md#L10-L20`)
- Delete old docs without deprecation period
- Leave contradictions unresolved
- Assume verbal agreements replace written docs

---

## 🚨 Error Handling Quick Guide

| Error | Meaning | Action |
|-------|---------|--------|
| **Checksum Mismatch** | Message corrupted in relay | Re-copy from agent, re-paste |
| **Timeout Exceeded** | Relay took too long | Executor auto-retries in 5-7 seconds |
| **Invalid JSON** | Message malformed | Check delimiters (=== START/END ===) |
| **Task ID Not Found** | Wrong message pasted | Find correct message, paste again |
| **Permanent Error** | Input schema invalid | Fix input, no auto-retry |
| **Transient Error** | Temporary issue | Auto-retry up to 3 times |

---

## 🎓 Learning Path

**Level 1: Operator** (5 min read)
→ This quick ref card

**Level 2: User** (30 min read)
→ `EXECUTIVE-SUMMARY.md`

**Level 3: Implementer** (2-3 hour read)
→ `POISE-Agent-Research-Report.md`
→ `A2A-Documentation-Agent-Architecture.md`

**Level 4: Architect** (full project)
→ Setup POISE planner agent
→ Configure A2A protocol
→ Deploy executor agent
→ Integrate with CI/CD

---

## 📞 Quick Help

**"Which document should be the source of truth?"**  
→ Use highest authority level from config (usually `api-spec.md`)

**"What if multiple docs have same authority level?"**  
→ Look at confidence score; higher = more likely correct; manual review if tied

**"Can I add custom conflict detection rules?"**  
→ Yes, create `.vale` rules or custom YAML rules in `linting_rules/`

**"How do I integrate with CI/CD?"**  
→ Add to GitHub Actions: `curl -X POST https://poise-agent/api/analyze -F documents=@docs/`

**"What happens to deprecated docs?"**  
→ Stage 1 (Week 1): Warning banner. Stage 2 (Week 2): Redirect. Stage 3 (Week 3): Archive. Stage 4: Delete (if expired).

---

## 📎 Key Files Reference

| File | Purpose | Read Time |
|------|---------|-----------|
| `EXECUTIVE-SUMMARY.md` | For leaders & PMs | 10 min |
| `POISE-Agent-Research-Report.md` | Technical deep-dive | 60 min |
| `A2A-Documentation-Agent-Architecture.md` | Protocol spec | 45 min |
| `QUICK-REFERENCE-CARD.md` | This file! | 5 min |
| `.poise-config.json` | Your config | Ongoing |
| `decisions.md` | Architecture decisions | Ongoing |
| `.vale.ini` | Linting rules | Ongoing |

---

**Status:** ✅ Ready to Use  
**Last Updated:** 2026-01-22  
**Questions?** Refer to full documentation or EXECUTIVE-SUMMARY.md
