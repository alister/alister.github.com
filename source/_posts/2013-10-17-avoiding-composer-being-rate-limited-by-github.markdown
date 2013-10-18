---
layout: post
title: "Avoiding Composer being Rate-limited by Github"
date: 2013-10-17 21:30
comments: true
categories: php continuous-integration jenkins composer
published: true
---
I've had a response to my post about "[Your Own 'Jenkins' for PHP With Composer](http://alister.github.io/blog/2013/06/03/your-own-jenkins-for-php-with-composer/)", via [a Github issue](https://github.com/alister/alister.github.com/issues/2) :), so I thought I'd answer that right here as well.

    in http://alister.github.io/blog/2013/06/03/your-own-jenkins-for-php-with-composer/
    it is using `--prefer-dist` which causes a hit on
    `api.github.com/repos/USER/REPO/zipball/SHA1` eg:

        https://api.github.com/repos/serbanghita/Mobile-Detect/zipball/.....

    which has a rate limit of 60 per hour. Unfortunately, we currently
    have 61 dependencies! Switching to `--prefer-source` works.

    Thanks for the blog post, got me a long way on the right track :thumbsup:

This is a similar problem to one I've had before, not from too many for a single repo, but while I was testing a composer-based deployment (with Capistrano). In either event, Github has a limit on the number of requests/downloads allowed in a period of time - here, 60 per hour.

There are two ways to solve this though. The first is quick and easy, but with that ease it also just expands on the potential long-term problems that could happen. It's certainly the first thing to do for a fix though.

[Quick, Easy and good enough](#solution-oauth) OR [Solid, reliable and local](#solution-satis)

<a id="solution-oauth"></a>
### Solution One, Quick, Easy and good enough for most individuals

##### Composer.json's github-oauth option.

It's not too well known, but is documented on the [composer schema page](http://getcomposer.org/doc/04-schema.md#config).

    github-oauth: A list of domain names and oauth keys. For example using
    {"github.com": "oauthtoken"} as the value of this option will use
    oauthtoken to access private repositories on github and to circumvent
    the low IP-based rate limiting of their API.

I've not seen the ability to use an Oauth token to be able to download from a VCS before (I've just looked on [Bitbucket.org](https://bitbucket.org/) for example), but according to the docs, that option in composer.json isn't limited to requests from Github.com.

With the OAuth token, the max-rate is [increased substantially](http://developer.github.com/v3/#rate-limiting), to 5000 per hour.

If you search for some online help to be able to generate the token from the command line, it will tell you how to run Curl against Githubs Authorization API:

    curl -u 'githubuser' \
    -d '{"note":"GitHub OAuth token for composer"}' https://api.github.com/authorizations

It will also ask your Github password. However, if you have the [2-factor authentication]() enabled for logins (and you should) - that won't help you. You also need to provide it with the request, which is hard work.

    curl -u 'alister' -d '{"note":"test token"}' https://api.github.com/authorizations
    Enter host password for user 'alister': __secret-password__
    {
      "message": "Must specify two-factor authentication OTP code.",
      "documentation_url": "http://developer.github.com/v3/auth#working-with-two-factor-authentication"
    }

There is a far easier way though. [On the web.](https://github.com/settings/applications). Click 'Create New Token', give it a description, and save it. Be sure to save a copy (put it into your composer.json file) right now though - as soon as you refresh the page, it will never be shown again.
[{% img right /assets/github-personal-asset-tokens.png "Create an Oaut token on the website"  %}](/assets/github-personal-asset-tokens.png)

And that's a lot easier.

<a id="solution-satis"></a>
### Solution Two, Solid, reliable and local.

##### Your own mini-packagist.org, with Composer-Satis

The solid solution is [Satis](https://packagist.org/packages/composer/satis), a related-project to the main Composer. This allows you to download a local version of potentially all the dependencies in your composer.json file(s). You also have the option to even have Satis automatically resolve and add all of the sub-dependencies for your projects as well. It's packages all the way down....

If you have local, private Github repository (and not just a private repo - an OAuth key can fetch that), or you want to be able to lock down the network to restrict which packages are allowed to be used (or equally, the machines where the packages would be installed are not able to access the 'outside world'), then Satis will be your best, or maybe even only option.

Setting up Satis and using it, is a bigger event though, so for now I'll just leave you with a few links, and come back to this another time....

 * http://getcomposer.org/doc/articles/handling-private-packages-with-satis.md
 * https://packagist.org/packages/composer/satis
 * https://packagist.org/search/?q=satis
 * https://github.com/yohang/satis-admin

Alister

----

My latest project has just been announced - [contractavailability](http://contractavailability.com/). It does pretty much what it says on the proverbial tin, but take a look at [my own profile](http://beta.contractavailability.com/u/alister) there for a little more info.  Sign up to the maining list, and I'll keep you up to date as it comes along.
