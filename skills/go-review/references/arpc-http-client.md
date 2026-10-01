---
description: Pattern for a Go HTTP client that calls an arpc service and unwraps its envelope
appliesWhen: a package defines a typed client that POSTs JSON to an arpc service and decodes the `{ok, result, error}` envelope (e.g. service-to-service calls, CLI/worker calling the partner/backoffice API)
---

# arpc HTTP Client Pattern

Use when building a typed Go client over an arpc service (one that returns the
`{ok, result, error}` envelope). Reference implementation: `api/pkg/partner/client.go`.

## Shape

- **Client struct** holds connection config only — `baseURL` + `apiKey`. Construct via
  `NewClient(baseURL, apiKey string) *Client`. No global state.
- **`invoke` is a METHOD on the client**, not a package-level generic function:
  `func (c *Client) invoke(ctx context.Context, method string, body, out any) error`.
  Go methods can't take type parameters, so decode into `out any` rather than a generic `[T]`.
- **The arpc method name IS the path segment** — `c.baseURL + "/" + method` (e.g. `match.list`).
  Do not pass a separate `path` argument.
- **One exported method per arpc call** (e.g. `ListMatches`, `FindByRoomMatchID`, `SetGameResult`),
  each wrapping `invoke`. Define the request body as a small inline anonymous struct with json
  tags; pass a local result struct (or `nil` to discard) as `out`.

## invoke body

1. Encode the request body with `json.NewEncoder(&buf).Encode(body)` into a `bytes.Buffer`
   (not `json.Marshal` + `bytes.NewReader`); pass `&buf` to `http.NewRequestWithContext`.
2. Set `Content-Type: application/json` and the auth header (`X-API-Key`).
3. Send via `httpclient.Do` — never `http.DefaultClient.Do`.
4. Non-200 → return `fmt.Errorf("... returned %d: %s", status, body)` after reading the body.
5. Decode the envelope with `Result json.RawMessage`. When `!ok`, return the typed `*APIError`
   (or a generic error if the envelope omits one). When `ok` and `out != nil` and the result is
   non-empty, `json.Unmarshal(result, out)`.

## Error type

- Provide an exported `APIError struct { Message string; Code string }` (json `message`/`code`)
  with an `Error()` method.
- Provide classification helpers that assert with `errors.AsType[*APIError](err)` (Go 1.26),
  e.g. `func IsNotFound(err error) bool { ... ae.Code == "not_found" }`.

## Reference skeleton

```go
type Client struct {
	baseURL string
	apiKey  string
}

func NewClient(baseURL, apiKey string) *Client { return &Client{baseURL: baseURL, apiKey: apiKey} }

func (c *Client) ListMatches(ctx context.Context, tournamentID string) ([]Match, error) {
	var r struct {
		Items []Match `json:"items"`
	}
	err := c.invoke(ctx, "match.list", struct {
		TournamentID string `json:"tournamentId"`
	}{TournamentID: tournamentID}, &r)
	if err != nil {
		return nil, err
	}
	return r.Items, nil
}

func (c *Client) invoke(ctx context.Context, method string, body, out any) error {
	var buf bytes.Buffer
	if err := json.NewEncoder(&buf).Encode(body); err != nil {
		return err
	}
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, c.baseURL+"/"+method, &buf)
	if err != nil {
		return err
	}
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("X-API-Key", c.apiKey)

	resp, err := httpclient.Do(req)
	if err != nil {
		return fmt.Errorf("http request: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		respBody, _ := io.ReadAll(resp.Body)
		return fmt.Errorf("api returned %d: %s", resp.StatusCode, string(respBody))
	}

	var envelope struct {
		OK     bool            `json:"ok"`
		Result json.RawMessage `json:"result"`
		Error  *APIError       `json:"error"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&envelope); err != nil {
		return fmt.Errorf("decode response: %w", err)
	}
	if !envelope.OK {
		if envelope.Error != nil {
			return envelope.Error
		}
		return fmt.Errorf("api error")
	}
	if out != nil && len(envelope.Result) > 0 {
		if err := json.Unmarshal(envelope.Result, out); err != nil {
			return fmt.Errorf("decode result: %w", err)
		}
	}
	return nil
}
```
