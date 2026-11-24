---
status: accepted
---

# Treat the pinned Vercel Board protocol as the public compatibility contract

OpenBoard will implement a production backend against a pinned revision of the Vercel Board protocol and reuse the official SDK/CLI rather than forking the entire upstream repository or inventing an unrelated API. Compatibility is the distribution strategy because it lets existing harnesses adopt OpenBoard without new clients, but it couples the project to an experimental upstream contract and requires ongoing conformance testing.
