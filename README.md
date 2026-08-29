# ai-control-plane

The spine of an LLM control plane: an HTTP gateway that LLM traffic routes through, a
provider abstraction behind it, and OpenTelemetry GenAI spans out the other side. Every
request's tokens, latency, provider and cost are recorded whether or not the caller
instruments anything.

Status: **v1 in progress.** Currently a walking skeleton — one route, one in-process fake
provider, spans printed to stdout. No persistence, no cost engine, no real provider yet.

Not implemented and not implied: authentication, rate limiting, tenancy *enforcement*.
`tenant_id` is a dimension the plane records, not a principal it authenticates.

Go 1.25 · stdlib only · `go run ./cmd/gateway`