# Self-Update

Update documentation now lives on a dedicated page: [Updating Coi](Updating-Coi).

```bash
coi update            # update the binary AND the detection databases
coi update core       # update the binary only
coi update patterns   # update the GTFOBins + Sigma databases only
```

See [Updating Coi](Updating-Coi) for how the update works, checksum verification, updating the detection databases, and what to do after updating.

## See Also

- [Updating Coi](Updating-Coi) — the full update guide
- [System Health Check](System-Health-Check) — health verification
- [Image Management](Image-Management) — rebuilding the container image after Coi updates
