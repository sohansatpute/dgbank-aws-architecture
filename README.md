## Writing Conventions

Every service in the [services/](services/) dictionary follows the same fixed template. This keeps the reference consistent and predictable — a reader who learns to read one entry can read all of them the same way.

The template is defined in [`services/_template.md`](services/_template.md). Do not edit the template without updating every existing service file.

Each service entry contains four fields:

1. **Category** — where the service belongs in the architecture
2. **Service overview** — a professional description of what the service is
3. **Purpose in DGBank** — the business, security, or operational reason it is used
4. **Position in architecture** — the layer it operates in and the request journeys it participates in

Fields are deliberately limited to these four. Cost, failure modes, configuration details, and operational procedures belong in separate documents, not in the service dictionary.