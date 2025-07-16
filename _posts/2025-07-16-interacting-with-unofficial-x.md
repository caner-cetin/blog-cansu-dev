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

this blog post is about Helicon, a project that I started quite a while ago and didnt go so far after authentication. Now that I am picking up this project again, I wanted to write a blog post about progression.

### why?

why Helicon, why unofficial API? one and only reason: official API pricing. 100$ is far too step for an API that once was completely free. you shouldnt pay 100$ for posting peugeot 206 pics daily, and, besides all fun things you could do with gimmick accounts, X API is not worth 100$. value of content you get out of 100$ is not even comparable to other pricegated trade // astronomy APIs. 

also, people are paying "unofficial x API resellers" way too much money. like, idk, 0.10$/1000 tweet information. unofficial API has no rate limits, no restriction, all you need is army of bot accounts to become a reseller. and GQL API is not a top secret information! its not like "unofficial API is too hard to use i must pay resellers", no, its actually pretty easy!

### how?

we will(try to) interact with unofficial API using our library, Helicon. lets begin with the most fundamental and often the trickiest part: Authentication. Unlike traditional APIs that might use simple API keys or OAuth flows, Twitter/X's internal mechanisms is a bit different.

### more than just a login

authenticating with X API isnt as simple as sending tokens, now you got the callback, here is your access tokens, etc., no, unofficial API is more than just a normal OAuth2 flow.

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

before we can even think about logging in, we need two tokens that grant us basic access to the Twitter/X API endpoints: an **Anonymous Bearer Token** and a Guest Token.

**Anonymous Bearer Token** is a static token embedded within Twitter/X's main JavaScript bundle. It's used for initial client-to-server communication, even before a user logs in.

at our library code, `FindTwitterMainJavascriptUrl` function scrapes the login page HTML to find the URL of this dynamic JavaScript file.

*`htmlContent` is html dump of login page, you can dump any pre-auth page html for this purpose*
![findsrc](/assets/x-api-1.png)

Once found, 
`FindAnonymousBearerToken` then fetches JS file and extracts the Bearer token in same fashion.
![findbearer](/assets/x-api-2.png)

Similarly, the `Guest Token` is also obtained by making an initial request to the login page. This token is often found in a `document.cookie` assignment within the HTML or as a `Set-Cookie` header, like `gt=12345...`

![findguest](/assets/x-api-3.png)

now we have two necessary tokens in hand, lets proceed with `https://api.x.com/1.1/onboarding/task.json`, main authentication endpoint.

### auth flow

authentication flow is done through one single endpoint, `task.json`.
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

flow is specified with query parameters:

```go
req, err := http.NewRequest(http.MethodPost, "https://api.x.com/1.1/onboarding/task.json", strings.NewReader(body))
q := req.URL.Query()
q.Set("flow_name", "login")
req.URL.RawQuery = q.Encode()
```

means that flow we are initiating is login flow.

after that, attach the required tokens at headers and send the constructed request with default http client
```go
req.Header.Set("Authorization", *anonymousToken)
req.Header.Set("x-guest-token", *guestId)
req.Header.Set("User-Agent", h.UserAgent)
req.Header.Set("Content-Type", "application/json")
resp, err := http.DefaultClient.Do(req)
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

here is what I did:

- launch  a headless Chrome instance.
- inject the custom challenge JS snippet into the browser. snippet overrides `document.getElementsByName` to capture the ui_metrics value that the challenge script tries to set.
- execute the challenge script within the headless browser.
- wait for and extract the computed ui_metrics value.

[you can read my implementation here](https://github.com/caner-cetin/helicon/blob/fa437f057bc94036009facf0021f75993970408c/twitter.go#L334)

[and how I used the final captcha (?) value here](https://github.com/caner-cetin/helicon/blob/fa437f057bc94036009facf0021f75993970408c/twitter.go#L268)

### submit credentials and cookies

from this point and on its fairly easy. all we need to do is sending credentials for last step, [see here](https://github.com/caner-cetin/helicon/blob/fa437f057bc94036009facf0021f75993970408c/twitter.go#L488)

### success?

upon a successful password submission, Twitter/X returns session cookies in the Set-Cookie headers of the HTTP response. Specifically, we need:

ct0: This is your CSRF (Cross-Site Request Forgery) token, vital for making subsequent authenticated requests.
auth_token: This is your primary session token, indicating you are logged in.


...wip.