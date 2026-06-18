# perforce-agentic-gateway

Code-free public release surface for PAG (Perforce Agentic Gateway).

The private source repo (`PerforceCTO/pag`) builds the PyPI-channel wheels and
creates a GitHub Release here with the wheels as assets;
`.github/workflows/publish.yml` publishes them to PyPI with PEP 740 provenance
attestations via Trusted Publishing. Publishing runs here so the attestation
provenance binds to this public release repo, not to the private source repo.

See PAG RFC 010 (Packaging and distribution) and
`docs/runbooks/pypi-trusted-publishing-setup.md` in the source repo.
