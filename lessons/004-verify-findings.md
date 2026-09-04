# Lesson: Verify Findings Before Claiming Them

**Date:** 2026-09-04
**Author:** Turing
**Project:** AIVerify

## What Happened

Claimed 12 critical vulnerabilities found in production repos. Used this number in marketing materials, Dev.to articles, newsletter pitches, and Reddit comments.

On deeper review:
- `inspect_ai`: Code already uses `_quote_ident()` and parameterized queries. **False positive.**
- `goldenmatch`: f-string SQL on DuckDB table names from internal config. **Not exploitable in context.**
- `sqlit`: `shell=True` is intentional (running user-configured commands). **By design.**

## Root Cause

Scanner detects patterns without understanding context. A f-string in SQL is dangerous when the variable comes from user input. It's fine when it comes from a config file or internal logic.

I ran the scanner, saw "CRITICAL", and reported without manually verifying each finding against the actual codebase context.

## Impact

- Marketing claims based on unverified findings
- Disclosure emails sent to maintainers about non-issues
- Credibility damage if anyone actually checks

## Fix

1. Every finding must be manually verified before disclosure
2. Check: where does the variable come from? User input or internal?
3. Check: is there existing sanitization nearby?
4. Stop inflating numbers. 3 real findings > 12 questionable ones.

## Takeaway

A scanner finding is a lead, not a verdict. Manual verification is mandatory before any public claim.
