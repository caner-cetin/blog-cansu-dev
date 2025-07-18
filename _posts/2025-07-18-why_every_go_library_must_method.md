---
layout: post
title: "Why Every Go Library Should Implement Must* Methods"
---

### Rambling About err != nil

We all know there is no real error handling in Go. The entire error "handling" system is just if checks with early returns, and it gets old so quickly.

But we can fix this! Partially, at least, with `Must*` methods. We don't have to write yet another `err != nil` check in our codebase, and we shouldn't write 20 lines of if clauses in a quick ~50 LOC throwaway script.

### What is a Must* Method?

Introducing the `valyala/fastjson` package, my favorite package in the entire Go ecosystem, and [you should use it too](https://github.com/valyala/fastjson).

`fastjson` is my favorite package for one reason only: the `Must*` method. Read this beauty:

```go
// MustParse parses json string s.
//
// The function panics if s cannot be parsed.
// The function is slower than the Parser.Parse for re-used Parser.
func MustParse(s string) *Value {   
    v, err := Parse(s)  
    if err != nil {     
        panic(err)  
    }   
    return v
}
```

As seen from the code, a `MustX` method is just "do X, I don't care about errors—if there is an error, panic; if not, return me the value."
### My Library Doesn't Return That Many Errors

Your library may not, but some libraries are absolute pain to work with due to the lack of Must methods. Check these `playwright` Go bindings.

Check this TypeScript code: 
```typescript
import { chromium } from "playwright";

async () => {
  const browser = await chromium.launch();
  const context = await browser.newContext();
  const page = await context.newPage();
  const response = await page.goto(
    "https://canwestoptypingerr!=nilateveryfunctioncall.to",
  );
};
```
And its Go equivalent:
```go
package main

import (
    "log"
    "github.com/playwright-community/playwright-go"
)

func main() { 
    if err := playwright.Install(&playwright.RunOptions{Browsers: []string{"chromium"}}); err != nil {      
        log.Fatalf("failed to install chromium: %v", err)   
    }   
    pw, err := playwright.Run() 
    if err != nil {     
        log.Fatalf("failed to run chromium: %v", err)   
    }  
    browser, err := pw.chromium.Launch()    
    if err != nil {     
        log.Fatalf("failed to run browser: %v", err)    
    }   
    context, err := browser.NewContext()    
    if err != nil {     
        log.Fatalf("failed to launch new context: %v", err) 
    }   
    page, err := context.NewPage()  
    if err != nil {     
        log.Fatalf("failed to launch new page: %v", err)    
    }   
    resp, err := page.Goto("https://canwestoptypingerr!=nilateveryfunctioncall.to") 
    if err != nil {     
        log.Fatalf("failed to navigate to page: %v", err)   
    }
}
```
This could be vastly simplified with Must methods:
```go
func main() {
    pw := playwright.MustRun()
    browser := pw.Chromium.MustLaunch()
    context := browser.MustNewContext()
    page := context.MustNewPage()
    resp := page.MustGoto("https://canwestoptypingerr!=nilateveryfunctioncall.to")
    // ...
}
```
Do you want to catch errors in TypeScript? Just wrap within a `try / catch` block. This reminds me of this rejected proposal from 2019: [built-in Go error check function, "try"](https://github.com/golang/go/issues/32437#issuecomment-512035919f).

### "Go's Error Handling is a Godsend"

From a random Reddit comment:
> Man I really, really dislike the prospect of hidden control flow in the case where the block is omitted.
> I flag people in code reviews when I see this kind of thing.
> If we make that block mandatory, then I think this proposal should pass, because the rest of it looks and feels like Go.
> Implicitly returning from a function does not feel like Go at all.

I wholeheartedly disagree with this entire statement. The "hidden control flow is evil, magic statements are evil" mindset is killing Go and making it look like a toy language.

This very same mindset almost killed the generics proposal, but Go does not have to be THIS primitive. Error handling-wise, everything is lacking—and for something to be primitive, that something has to exist first. There is no error handling here. There is no god above. It is all `err != nil` checks, and I am so, so done with this. For all my new projects, I am using Bun + TypeScript, and it feels so good to not write a single `err != nil` clause.

### Summary

This was just a short `err != nil` rambling. Please, for the love of god, wrap your `X` methods in `MustX` methods so that I can write throwaway scripts. Please.

