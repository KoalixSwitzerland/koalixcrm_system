# 11. Risks and Technical Debt

This chapter summarizes architecture recommendations, prioritized risks, and technical debt
items identified during analysis of the koalixcrm codebase and its documentation.

## Contents

- [Architecture Recommendations](QQ_SD_Architecture_Recommendations.md) — Prioritized backlog
  of 46 actionable items across 9 categories, ready for import into a task management system.
- [Open Questions](open_questions.md) — Clarification questions, design decisions, and
  decision status for system-design phase items.

## Summary

| Category | Critical | High | Medium | Low | Total |
|----------|----------|------|--------|-----|-------|
| Security Gaps | 3 | 4 | 4 | 4 | 15 |
| Access Control and Authorization Gaps | 0 | 1 | 3 | 1 | 5 |
| Testing Gaps | 0 | 3 | 3 | 1 | 7 |
| Technical Debt | 0 | 2 | 3 | 0 | 5 |
| Configuration and Settings Gaps | 0 | 1 | 2 | 0 | 3 |
| Architecture Improvements | 0 | 1 | 2 | 1 | 4 |
| Operational Improvements | 0 | 1 | 1 | 1 | 3 |
| Performance Optimizations | 0 | 0 | 2 | 0 | 2 |
| Documentation Gaps | 0 | 0 | 1 | 2 | 3 |
| **Total** | **3** | **13** | **21** | **9** | **46** |

The three Critical items (REC-001, REC-002, REC-003) address known-default credentials and
always-on BasicAuthentication in the base settings. These should be resolved before any
production deployment.
