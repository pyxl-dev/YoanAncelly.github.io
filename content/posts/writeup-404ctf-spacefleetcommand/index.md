---
title: "Writeup 404CTF - Space Fleet Command"
description: "A web challenge combining SSRF, CSRF and XS-Leaks"
date: 2025-06-06T12:19:13.562Z
draft: true
tags: ["404CTF", "web", "writeup", "ctf", "csrf", "ssrf", "xs-leaks"]
---

## Overview

Space Fleet Command is a web challenge from 404CTF created by **\_hdrien**. It was released as a "hard" difficulty challenge and has been resolved by 27 players out of over 2500.

That was a fun challenge and I learned a lot from it because I've never done hard difficulty challenge before. Just for information it took me around 60 hours to resolve it and I'm proud of it.

In this writeup, I’ll walk you through my thought process, the tools I used, and how I ultimately captured the flag.

![Challenge](challenge.png)

## Analysis

First thing first, I started the challenge in a black box way for a simple reason : I absolutely didn't saw that the source code was available. No judgement, I'm proud that I've found a few things in this way haha.

### Pages

The challenge is a web application to manage spaceships and report bugs.

#### Home page

> Route : `/`

On this page there is an input to search for a spaceship by name. It will display the spaceship page in the list if it exists.

![Home page](home.png)

#### Report page

> Route : `/report`

On this page there is a form to report a bug. Once you submit the form, it says that an admin will visit the page.

![Report page](report.png)

#### Spaceship page

> Route : `/spaceship/:id`

On this page you can see the details on the selected spaceship.

![Spaceship page](spaceship.png)

### First tries

I started by trying some simple XSS attacks and SQL injection but it didn't work.

After some research, I succeeded to trigger an error on the `/spaceship/:id` page by using a malformed URL. I used `spaceship/%90` because it's not decoded as a character but I quickly understood that was a rabbit hole.

![Error](error-spaceship.png)

Then I tried to attack the `/?q=` query parameter and found that I was able to trigger an error by using [HTTP Parameter Pollution](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/04-Testing_for_HTTP_Parameter_Pollution) which is a technique where you put a parameter multiple times with different values. One more time it was a rabbit hole.

![Error](error-q.png)

It was time to dig onto `/report` page and I had a good feeling about it. When you fill the two input fields you have a request like this `url=https%3A%2F%2Fspacefleetcommand.404ctf.fr%2Fspaceship%2F0&issue=check+by+yourself`. The message `The page was visited by our admin. We'll get back to you soon.` is displayed and it's a big hint for an incoming CSRF attack.

Few hours later I came back on the challenge page and found that the source code was available 🤡. Don't do like me and read carefully the information given by the challenge creator.

### Digging the source code

#### Understanding the flag

I wanted to know how the flag is set and I found it in the `app.js` file.

![Source code of the app.js file](source-code-app.png)

As you can see, the flag is set as the `name` of a spaceship which is set as `classified`. Spaceships are stored in a JSON file.

![Source code of the spaceships.json file](spaceships.png)

On the sample of spaceships above you can see a spaceship with `classified` set to true, it means that its name is set to the flag.

#### Checking some protections

![CSP](csp.png)

The server has a super strict [Content Security Policy (CSP)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP) that disallows scripts from running and allows only the `self` by default. It means that we can't use any javascript based attacks.

#### Looking for vulnerabilities

Now as I know the `/report` route is a potential CSRF attack, I started to dig onto the source code.

![Source code of the report page](source-code-report.png)

First of all, we use only the `url` parameter on the backend. Then we can see at line 64 the `extractPath()` function which takes the `url` parameter and returns the path of the URL. This function is in the `utils.js` file.

![Source code of the extractPath function](source-code-extractPath.png)

Here the things are super interesting because on line 5 my IDE tells me `url.parse()` is deprecated with a strikethrough text and the comment `//who cares about deprecation anyway` go on this way.

The url we provide must start with `http://` or `https://` and the path is extracted from the url.

The big part from line 9 to 12 shows us that a check on the provided hostname was done but commented out for some reason. That's a big hint for a SSRF attack.

Once the path is correct, the `visit()` function is called with the path as parameter. This function is in the `bot.js` file.

![Source code of the visit function](source-code-visit.png)

An admin puppeteer bot visits the page crafted as `http://${challenge_host}${path}`. It will visit the page while the `token` cookie is set for `challenge_host` domain. Which means the cookie is accessible only on `spacefleetcommand.404ctf.fr` and it's not stealable due to the CSP.

#### SSRF attack

At this point I decided to open a `webhook.site` page and put the URL in the `url` parameter. Of course it didn't work as is but I quickly found the trick by testing multiple things.

Here is the trick : `http:@webhook.site/{random-id}`

Let me explain.

![URL structure](url-structure.png)
_Source: [https://dmitripavlutin.com/parse-url-javascript/](https://dmitripavlutin.com/parse-url-javascript/)_

A URL can have a username and a password. For example `http://username:password@example.com` or `http://username@example.com`.

When you put this URL in the `url` parameter, the `extractPath()` function will return `@example.com` as the path because the `url.parse()` function will consider `@` as a path separator like `/`.

Knowing that, the URL becomes `http://{challenge_host}@example.com` and at this point the `{challenge_host}` part is considered as the username of the URL.

So at the end, the URL will be visited as `http://example.com` with `{challenge_host}` as the username.

Let's put `https://@webhook.site/7e7b947e-0d2b-43ec-8317-d10ac4599b66` in the `url` parameter and **BOOM !**

![Webhook.site](webhook-site.png)

At this point I was super excited because I knew that I was able to perform a SSRF attack and make the bot visit any URL.

#### CSRF + XS-Leaks attack
