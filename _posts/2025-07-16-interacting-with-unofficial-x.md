---
layout: post
title: "Using Unofficial Twitter/X API (Part 1): How to Authenticate with Go"
---

### read this /// disclaimer

This project is an UNOFFICIAL library for interacting with Twitter/X's internal APIs. It is NOT affiliated with, endorsed, or sponsored by Twitter, Inc. or X Corp.

Using this library may violate Twitter/X's Terms of Service. This can lead to actions against your account, including but not limited to rate-limiting, suspension, or permanent banning.

### repo
[right here](https://github.com/caner-cetin/helicon)

### intro

This blog post is about Helicon, a project that I started quite a while ago and didn't go so far after authentication. Now that I am picking up this project again, I wanted to write a blog post about the progression and share how the authentication flow works.

### why?

Why Helicon, why unofficial API? One and only reason: official API pricing. $100 is far too steep for an API that once was completely free. You shouldn't pay $100 for posting Peugeot 206 pics daily, and besides all fun things you could do with gimmick accounts, X API is not worth $100. The value of content you get out of $100 is not even comparable to other price-gated trade or astronomy APIs.

Also, people are paying "unofficial X API resellers" way too much money - like $0.10 per 1000 tweet information. The unofficial API has no rate limits, no restrictions, all you need is an army of bot accounts to become a reseller. And the GraphQL API is not top secret information! It's not like "unofficial API is too hard to use, I must pay resellers" - no, it's actually pretty easy!

### how?

We will (try to) interact with the unofficial API using our library, Helicon. Let's begin with the most fundamental and often the trickiest part: Authentication. Unlike traditional APIs that might use simple API keys or OAuth flows, Twitter/X's internal mechanisms are a bit different.

### more than just a login

Authenticating with X API isn't as simple as sending tokens - "now you got the callback, here are your access tokens, etc." No, the unofficial API is more than just a normal OAuth2 flow.

here is a general overview of how authentication with unofficial API looks:

```
+--------------------------+          +------------------------+
| 1. Find Anonymous Bearer |<--------->| X.com Login Page (HTML)|
|    Token & Guest Token   |          +------------------------+
+--------------------------+
             |                                    |
             |                                    |
             V                                    V
+--------------------------+          +------------------------+
| 2. Start Login Flow      |<--------->| X.com Main JS File     |
|    (with anon token,     |          | (Contains Bearer Token)|
|    guest token)          |          +------------------------+
+--------------------------+
             |
             | (Receives JS Challenge URL)
             V
+--------------------------+
| 3. Solve JS Challenge    |
|    (headless browser)    |
+--------------------------+
             |
             | (Submits Solution)
             V
+--------------------------+
| 4. Submit Username       |
+--------------------------+
             |
             |
             V
+--------------------------+
| 5. Submit Password       |
|    (captures auth_token, |
|    ct0 cookies)          |
+--------------------------+
             |
             |
             V
+--------------------------+
| 6. Save Tokens to Keyring|
+--------------------------+
```

### initial: anonymous token

Before we can even think about logging in, we need two tokens that grant us basic access to the Twitter/X API endpoints: an **Anonymous Bearer Token** and a **Guest Token**.

The **Anonymous Bearer Token** is a static token embedded within Twitter/X's main JavaScript bundle. It's used for initial client-to-server communication, even before a user logs in.

at our library code, `FindTwitterMainJavascriptUrl` function scrapes the login page HTML to find the URL of this dynamic JavaScript file:

```go
func (h *Helicon) FindTwitterMainJavascriptUrl() (*string, error) {
	req, _ := http.NewRequest("GET", "https://x.com/i/flow/login/", nil)
	req.Header.Set("User-Agent", h.UserAgent)
	resp, _ := http.DefaultClient.Do(req)
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	htmlContent := string(body)

	re := regexp.MustCompile(`src=["'](https://abs\.twimg\.com/responsive-web/client-web-legacy/main\.[\w.-]+\.js)["']`)
	matches := re.FindStringSubmatch(htmlContent)
	if len(matches) == 0 {
		re = regexp.MustCompile(`src=["'](https://abs\.twimg\.com/responsive-web/client-web/main\.[\w.-]+\.js)["']`)
		matches = re.FindStringSubmatch(htmlContent)
	}

	scriptUri := strings.TrimPrefix(strings.TrimSuffix(matches[0], `"`), `src="`)
	return &scriptUri, nil
}
```

Once found, `FindAnonymousBearerToken` then fetches JS file and extracts the Bearer token:

```go
func (h *Helicon) FindAnonymousBearerToken() (*string, error) {
	mainScriptUrl, _ := h.FindTwitterMainJavascriptUrl()
	req, _ := http.NewRequest("GET", *mainScriptUrl, nil)
	req.Header.Set("User-Agent", h.UserAgent)
	resp, _ := http.DefaultClient.Do(req)
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	re := regexp.MustCompile("(Bearer)(.*?)(\"| \\z)")
	matches := re.FindAll(body, -1)
	match := string(matches[len(matches)-1])
	match = strings.TrimPrefix(strings.TrimSuffix(match, `"`), `"`)
	return &match, nil
}
```

Similarly, the `Guest Token` is obtained by making an initial request to the login page and parsing the HTML for the cookie assignment:

```go
func (h *Helicon) GenerateGuestToken() (*string, error) {
	req, _ := http.NewRequest("GET", "https://x.com/i/flow/login/", nil)
	req.Header.Set("User-Agent", h.UserAgent)
	resp, _ := http.DefaultClient.Do(req)
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	htmlContent := string(body)
	re := regexp.MustCompile(`document\.cookie="gt=([0-9]+)`)
	matches := re.FindStringSubmatch(htmlContent)
	return &matches[1], nil
}
```

now we have two necessary tokens in hand, lets proceed with `https://api.x.com/1.1/onboarding/task.json`, main authentication endpoint.

### auth flow

The authentication flow is done through one single endpoint, `task.json`. This endpoint handles the entire login process through a series of subtasks.
```json
{
	"input_flow_data": {
		"flow_context": {
			"debug_overrides": {},
			"start_location": {
				"location": "manual_link"
			}
		}
	}
}
```

The login flow is initiated with a POST request to the task endpoint:

```go
func (h *Helicon) StartLoginFlow() (*LoginFlow, error) {
	anonymousToken, _ := h.FindAnonymousBearerToken()
	guestId, _ := h.GenerateGuestToken()

	body := `{
		"input_flow_data": {
			"flow_context": {
				"debug_overrides": {},
				"start_location": {
					"location": "manual_link"
				}
			}
		}
	}`

	req, _ := http.NewRequest(http.MethodPost, "https://api.x.com/1.1/onboarding/task.json", strings.NewReader(body))
	q := req.URL.Query()
	q.Set("flow_name", "login")
	req.URL.RawQuery = q.Encode()

	req.Header.Set("Authorization", *anonymousToken)
	req.Header.Set("x-guest-token", *guestId)
	req.Header.Set("User-Agent", h.UserAgent)
	req.Header.Set("Content-Type", "application/json")

	resp, _ := http.DefaultClient.Do(req)
	// Parse response into LoginFlow struct...
}
```
afterwards, initiated flow details will be sent from X in the format of:
```go
type LoginFlow struct {
	FlowToken string `json:"flow_token"`
	Status    string `json:"status"`
	Subtasks  []struct {
		SubtaskId         string `json:"subtask_id"`
		JsInstrumentation struct {
			Url       string `json:"url"`
			TimeoutMs int    `json:"timeout_ms"`
			NextLink  struct {
				LinkType string `json:"link_type"`
				LinkId   string `json:"link_id"`
			} `json:"next_link"`
		} `json:"js_instrumentation"`
	} `json:"subtasks"`
	AnonymousBearerToken string
	GuestToken           string
	GuestId              string
	UserAgent            string
	Att                  string
	CFBM                 string
}
```

### anti-bot challenge

when we start the login flow, X responds with a `JsInstrumentationSubtask`, as a anti-bot measurement, providing a URL to challenge JS file. this script, when executed in a browser environment, computes a unique "solution" string (often an ui_metrics value) that must be submitted back to the API.

we can't just eval this JS in Go and we cant use JS engines, we need a full browser environment due to dependency on document API. we will use chromedp, a Go library for controlling Chrome DevTools Protocol, to spawn a headless Chrome instance.

Here's how the JS challenge is solved:

```go
func (f *LoginFlow) solveJSInstrumentationChallenge(userAgent string) (*string, error) {
	// Fetch the challenge script from X's servers
	target := f.Subtasks[0].JsInstrumentation.Url
	resp, _ := http.Get(target)
	defer resp.Body.Close()
	script, _ := io.ReadAll(resp.Body)
	scriptContent := string(script)

	// Setup headless Chrome
	allocOpts := append(chromedp.DefaultExecAllocatorOptions[:],
		chromedp.Flag("headless", true),
		chromedp.Flag("disable-gpu", true),
		chromedp.UserAgent(userAgent),
		chromedp.NoSandbox,
	)
	allocCtx, cancelAlloc := chromedp.NewExecAllocator(context.Background(), allocOpts...)
	defer cancelAlloc()
	taskCtx, cancelTask := chromedp.NewContext(allocCtx)
	defer cancelTask()
```

The core technique involves overriding `document.getElementsByName` to capture the ui_metrics value:

```go
	jsToEvaluate := fmt.Sprintf(`
		(() => {
			return new Promise((resolve, reject) => {
				let resultToCapture;
				const originalGetElementsByName = document.getElementsByName;
				document.getElementsByName = function(name) {
					if (name === 'ui_metrics') {
						return [{ 
							set value(val) {
								resultToCapture = val;
								resolve(resultToCapture);
							}
						}];
					}
					return originalGetElementsByName.apply(this, arguments);
				};
				eval(%s); // Execute the challenge script
			});
		})();
	`, "`"+scriptContent+"`")

	// Execute in browser and return the captured ui_metrics value
	chromedp.Run(taskCtx, chromedp.Evaluate(jsToEvaluate, &evaluationResult))
	return &result, nil
}
```

### submit credentials and cookies

from this point and on its fairly easy. all we need to do is sending credentials:

{% raw %}
```go
func (f *LoginFlow) SubmitUsernameAndPassword(helicon *Helicon) error {
	// First, submit username
	submitUsernameBody := SubmitUsernameRequest{
		FlowToken: f.FlowToken,
		SubtaskInputs: []SubtaskInputs{{
			SubtaskID: "LoginEnterUserIdentifierSSO",
			SettingsList: SettingsList{
				Link: "next_link",
				SettingResponses: []SettingResponses{{
					Key: "user_identifier",
					ResponseData: ResponseData{
						TextData: TextData{
							Result: helicon.Credentials.Username,
						},
					},
				}},
			},
		}},
	}

	// Submit username and get response
	req, _ := http.NewRequest(http.MethodPost, "https://api.x.com/1.1/onboarding/task.json", body)
	req.Header.Set("Authorization", f.AnonymousBearerToken)
	req.Header.Set("x-guest-token", f.GuestToken)
	resp, _ := http.DefaultClient.Do(req)
	// Parse response...
```

Then submit the password in a similar fashion:

```go
	// Submit password
	submitPasswordRequest := SubmitPasswordRequest{
		FlowToken: submitUsernameResponse.FlowToken,
		SubtaskInputs: []SubmitPasswordSubtaskInput{{
			EnterPassword: EnterPassword{
				Link:     "next_link",
				Password: helicon.Credentials.Password,
			},
			SubtaskID: "LoginEnterPassword",
		}},
	}

	// Submit password and capture session cookies
	req, _ = http.NewRequest(http.MethodPost, "https://api.x.com/1.1/onboarding/task.json", body)
	resp, _ = http.DefaultClient.Do(req)
	// Extract auth_token and ct0 from Set-Cookie headers
}
```
{% endraw %}

### success?

upon a successful password submission, Twitter/X returns session cookies in the Set-Cookie headers of the HTTP response. Specifically, we need:

- **ct0**: This is your CSRF (Cross-Site Request Forgery) token, vital for making subsequent authenticated requests.
- **auth_token**: This is your primary session token, indicating you are logged in.

These tokens are then saved to the system keyring for future use:

```go
// Extract cookies from response headers
for _, cookie := range resp.Cookies() {
	switch cookie.Name {
	case "ct0":
		h.Credentials.CSRFToken = cookie.Value
	case "auth_token":
		h.Credentials.AuthToken = cookie.Value
	}
}

// Save to keyring for persistence
func (h *Helicon) SaveTokensToKeyring() error {
	pass := fmt.Sprintf("%s|%s|%s", 
		h.Credentials.CSRFToken, 
		h.Credentials.BearerToken, 
		h.Credentials.AuthToken)
	return keyring.Set("helicon", h.Credentials.Username, pass)
}
```

### next steps

With these tokens in hand, you can now make authenticated requests to Twitter/X's internal GraphQL endpoints. 

### wait, how???

well, thats the hardest part, and, hopefully I will cover this sooner or later.

but I have an example for you! which you can [access here](https://github.com/caner-cetin/helicon/blob/master/tweet_detail.go)

hitting API itself is not the hardest part. all you have to do is setting headers properly:

```go
// anonymous bearer token
// dont mix with your own auth token
req.Header.Set("Authorization", h.Cookies.BearerToken)
req.Header.Set("X-Csrf-Token", h.Cookies.CSRFToken.Value)
req.Header.Set("X-Twitter-Auth-Type", "OAuth2Session")
req.Header.Set("X-Twitter-Active-User", "yes")
req.Header.Set("X-Twitter-Client-Language", "en")
req.Header.Set("Cookie", fmt.Sprintf("auth_token=%s; ct0=%s", h.Cookies.AuthToken.Value, h.Cookies.CSRFToken.Value))
req.Header.Set("Accept", "*/*")
req.Header.Set("User-Agent", h.UserAgent)
```

hardest part is constructing the URL for request because GQL is one giant mess.

```go
func (h *Helicon) hitApi(url string) ([]byte, error) {
	req, _ := http.NewRequest(http.MethodGet, url, nil)
	// see the code block above
	h.setCommonHeaders(req)
	resp, _ := http.DefaultClient.Do(req)
	var respBody []byte
	respBody, _ = io.ReadAll(resp.Body)
	return respBody, nil
}
```
this is all you need to hit the API. set headers and you are good to go. but.

internal GQL calls are shaped like this
```
https://x.com/i/api/graphql/XXXX/TweetDetail
```
where the `X` is GraphQL query ID. Thats the hardest part. Its not a static query ID. With every update query ID changes. Good news is that, query IDs are also stored at main JS bundle so all we need is 600 lines of regex and some black magic to find all GQL ids.

I highly recommend you to read this file to understand how messy API request // responses are: [read me pls](https://github.com/caner-cetin/helicon/blob/master/tweet_detail.go)

### closing

This is how far I have got with the library, ill start working on it again. Full implementation is available in the [Helicon repository](https://github.com/caner-cetin/helicon) - feel free to contribute or report issues!