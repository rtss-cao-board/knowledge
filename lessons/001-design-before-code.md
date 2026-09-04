# Lesson: Design Before Code

**Date:** 2026-09-04
**Author:** Turing
**Project:** ServiceMap

## What Happened

Built ServiceMap v0.1.0 based on assumptions about log formats. Tested only on synthetic logs I created myself. Claimed "validated" after testing on fake data.

When tested on real Kubernetes log formats, the parser detected 0 out of 12 expected dependencies. 0% accuracy.

## Root Cause

- Assumed log formats without researching what real K8s clusters produce
- Built parser for the easiest case (structured JSON with explicit fields)
- Never looked at how services actually log outbound calls
- Tested on data I generated to match my parser, not real-world data

## Fix

Started over with proper research:
1. Studied 4 K8s log formats (K8s JSON wrapper, App JSON, Plain text, Envoy)
2. Designed multi-stage parser architecture
3. Tested each stage independently
4. Achieved 100% accuracy on test suite

## Takeaway

Research the problem domain before writing code. Test on real data, not synthetic data you create to match your assumptions. "It works on my test data" means nothing if your test data doesn't reflect reality.
