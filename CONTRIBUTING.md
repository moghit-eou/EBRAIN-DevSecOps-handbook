# Contributing

1. Open an issue before sending a PR.
2. Match the section type when editing:
   - tutorials/: step by step, must work start to finish
   - how-to/: task oriented, one task per file
   - reference/: tables and facts only, no narrative
   - explanation/: the why, no step by step instructions
3. Two gate models exist now. When you write reference or how-to content,
   say explicitly which model applies: CVSS score (Trivy, OSV-Scanner) or
   rule severity (OpenGrep, Hadolint). Do not assume the reader knows which
   one a given tool uses.
4. Keep root sections generic. Concrete specifics belong in case-studies/.
