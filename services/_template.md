# Service Entry Template

Every service in this dictionary follows this exact template. Do not deviate — consistency is what makes the dictionary usable.

---

### Service Name (official AWS or vendor name)

| Field | Description |
|---|---|
| **Category** | Networking / Security / Compute / Data / Monitoring / External |
| **Plain English** | One sentence a non-technical person can understand |
| **Why DGBank uses it** | The business, security, or operational reason |
| **Where it sits in the flow** | Which request journeys it participates in (reference `docs/03-request-journeys.md`) |
| **Real-world analogy** | An everyday comparison that makes the concept click |
| **Managed or self-hosted** | Does AWS/vendor run it, or does DGBank run it? |
| **Cost model** | How DGBank is billed (per request, per hour, per GB, etc.) |
| **Failure impact** | What happens if this service is unavailable |
| **Key configuration notes** | Any DGBank-specific settings worth knowing |

**Extended notes (only if needed):**
If the service is unusual, vendor-specific, or commonly misunderstood, add a short paragraph here.

---