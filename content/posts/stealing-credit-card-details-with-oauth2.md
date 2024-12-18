+++
title = 'Stealing Credit Card Details with the OAuth2 State Parameter'
date = 2024-12-17T00:00:00Z
draft = false
+++

I recently discovered a vulnerability in the login flow for a large company with multiple offerings, one of which is a credit card. The vulnerability allowed an attacker to craft an open redirect via the OAuth2 `state` parameter - redirecting the end-user to a disguised post-login URL from the authority of the company's root domain. Coupled with a sophisticated phishing attack, this flaw could have allowed attackers to steal sensitive information from users, such as credit card details.

The vulnerability was immediately disclosed to the company and fixed promptly - which means I can responsibly dissect the gory details, resolution and preventative measures with you. I'm going to point out all the mistakes made and provide some fixes and preventative measures for each of them.

## The Login Flow
This article assumes basic knowledge of the OAuth2 Standard. If you're unfamiliar, the [Auth0 Auth 2.0 overview](https://auth0.com/intro-to-iam/what-is-oauth-2) is a good start.

The attack was made possible via the OAuth2 `state` parameter, which serves a dual purpose; maintaining `state` between the client and authorisation server, and (ironically) preventing [Cross Site Request Forgery](https://owasp.org/www-community/attacks/csrf) attacks.

The login flow seems typical at first glance; it mostly follows the OIDC flow using Azure AD as the Authorization Server:

1. The user clicks 'Log In' when browsing any page from the company's main website (e.g. https://company.co.uk/clothing/t-shirts)
2. The main website kicks off the Authorization Code Flow with an Azure AD instance, and redirects the user to Azure AD to log-in (e.g. https://login.company.co.uk/login...)
3. The user enters their credentials and is then redirected back to the company's callback URL (e.g. https://company.co.uk/callback)
4. The authorization code grant is completed, and the end-user is then redirected back to the page from which they began the log-in flow (e.g. https://company.co.uk/clothing/t-shirts)

## The Attack
The URL in step two looks something like this (details omitted):

> https://login.company.co.uk/oauth2/v2.0/authorize?client_id=xxx&scope=openid%20profile&redirect_uri=https%3A%2F%2Fwww.company.co.uk%2Fcallback&response_type=code&client_info=1&code_challenge=xxx&code_challenge_method=S256&state=%7B%22redirect_uri%22%3A%22https%3A%2F%2Fcompany.co.uk%2Fclothing%2Ft-shirts%22%7D

If you take a closer look at the `state` parameter:

`%7B%22redirect_uri%22%3A%22https%3A%2F%2Fcompany.co.uk%2Fclothing%2Ft-shirts%22%7D`

You'll notice it's just a URI-encoded JSON object (mistake number one). It's trivial to decode, and looks like this:

```JSON
{"redirect_uri":"https://company.co.uk/clothing/t-shirts"}
```

In the OAuth2 standard, the `redirect_uri` provided to the authorization server should be validated - but this is different! This URI isn't even parsed by the IDP, it's just passed back to the company in the `state` parameter. Surely, I can't just change this, can I?

I discovered that crafting my own object by replacing the `redirect_uri` with my blog domain, URI encoding the object and replacing it in the login URL would redirect me to https://mostlyinaccurate.com with a login URL like this:

> https://login.company.co.uk/oauth2/v2.0/authorize?client_id=xxx&scope=openid%20profile&redirect_uri=https%3A%2F%2Fwww.company.co.uk%2Fcallback&response_type=code&client_info=1&code_challenge=xxx&code_challenge_method=S256&state=%7B%22redirect_uri%22%3A%22https%3A%2F%2Fmostlyinaccurate.com%22%7D

Not only does this mean that the `state` isn't being bound to my client (mistake number two), but the login callback URL allows me to redirect the end-user to any URL I like (mistake number three)!

### What about PKCE?
The eagle-eyed amongst you may have noticed that the OAuth implementation is making use of PKCE - for which the verifier must be bound to the client. In theory, this should make a cross-site request forgery attack more difficult. If I craft a malicious URL and send it to an unsuspecting user, the flow should be unable to complete - as the verifier will not be present on the client. This almost stopped the attack in its tracks, but some well intentioned functionality to recover from failure meant an attack was still possible. 

The initial login attempt would fail from a missing code verifier, and the user would be presented with a failure screen on the company's main website with a "Try again" button. But if the user clicked the "Try Again" button, the flow would re-start with the persisted `state` from the maliciously crafted URL, and the new PKCE verifier persisted to the user agent.

PKCE is beyond the scope of this post, but you should [become familiar with the standard](https://oauth.net/2/pkce/) if you aren't already.

### Mistake Number One: Making the state Transparent
The first mistake was making the `state` transparent. I'm not a big believer in reliance on security through obscurity, but the OAuth 2 RFC recommends that the `state` parameter [should be opaque](https://datatracker.ietf.org/doc/html/rfc6749). Injection attacks are much more difficult if the value isn't guessable, so store the actual `state` securely on your client or backend (Redis cache, local storage etc.) and use a cryptographically random secure string lookup key in the `state` parameter.

This is potentially controversial advice - the attack could have been mitigated by `state` validation on the user agent (opaque or not), but in this case I'd recommend following the advice in the OAuth2 specification, which gives some defence in depth.

### Mistake Number Two: Not binding State to the User Agent
Even if we're generating an opaque `state` value, there's still plenty that can go wrong in the middle. The user agent could be falling victim to a cross-site request forgery attack for example, so we need to be sure the user agent completing the flow is the one that initiated it. The OAuth2 RFC recommends that [the user-agent's authenticated state (e.g., session cookie, HTML5 local storage) MUST be kept in a location accessible only to the client and the user-agent](https://datatracker.ietf.org/doc/html/rfc6749#section-10.12).

In practice, this might mean that the opaque `state` parameter we generate at the start of our flow (e.g. when a user clicks 'Log In') could be returned in a `Set-Cookie` header with the redirect request to our identity provider. When the client redirect endpoint is hit with the authorization code, we first ensure that the `state` parameter exists in the request cookies - or we deny the request.

> PKCE (originally developed for public OAuth Clients) can also help to prevent other attacks like authorization code injection, even for non-public clients.

## Mistake Number Three: Trusting the State
At the end of the day, the `state` parameter returned in an OAuth callback request is input from a third party - and shouldn't be trusted. In this case, the input was a redirect URL - and an assumption was made that the URL could be trusted. The CWE that the business fell victim to is known as an [Open Redirect](https://cwe.mitre.org/data/definitions/601.html) - whereby an input parameter is used to redirect a request to an arbitrary URL.

The mitigation advice for open redirect attacks is to assume all input is malicious - which is a useful mindset to have when handling any input parameter of the OAuth2 flow. In this case the attack could be mitigated using an 'allow list' of acceptable URLs - only allowing redirects to a set of URLs on the `company.co.uk` domain. This should be an absolute list of URLs if possible - pattern matching/parsing can still be vulnerable to injection attacks, but equality comparisons not so much!

## Summary
This attack was trivial to carry out but threatened potentially devastating consequences. The functionality no doubt went through manual code reviews and automated static analysis but still made its way into a production environment, potentially impacting millions of end users. 

Alongside the three main takeaways (making your `state` opaque, always binding `state` to the initiating user agent and assuming all input is malicious), you should also endeavour to have at least one engineer on your team with a [security mindset](https://owasp.org/www-project-security-culture/v10/4-Security_Champions/). Try hacking yourself first, you might be surprised at what you find!