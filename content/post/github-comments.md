+++
date = "2017-04-21T20:30:00+01:00"
draft = false
title = "Replacing Disqus with Github Comments"
tags = [ "blog" ]
ghcommentid = 1
+++

I've been considering removing comments from this blog for a while; mainly because the site doesn't
trigger much discussion and I didn't like keeping the overhead of [Disqus](https://disqus.com/)
around. After looking into Disqus load-time behaviour I was pretty shocked what I was forcing
on people loading the site (although you really should be using the likes of [Privacy Badger](https://www.eff.org/privacybadger) and [uBlock Origin](https://github.com/gorhill/uBlock)).

<!--more-->

This post is Hugo-specific but the very little code required can be adapted to any website.

##### What's Wrong with Disqus?

This is a typical request log with Disqus *enabled*:

<center>[![d](/img/GithubComments/DisqusLoadLog.png)](/img/GithubComments/DisqusLoadLogHigh.png)</center>

This is the same request log with Disqus *disabled*:

<center>[![d](/img/GithubComments/LoadLog.png)](/img/GithubComments/LoadLogHigh.png)</center>

<center>***WAT!?***</center>

Relevant points are:

* Load-time goes from 6 seconds to 2 seconds.
* There are 105 network requests vs. 16.
* There are a lot of non-relevant requests going through to networks that will be tracking your movements.

Among the networks you can find:

* `disqus.com` - Obviously!
* `google-analytics.com` - Multiple requests; no idea who's capturing your movements.
* `connect.facebook.net` - If you're logged into Facebook, they know you visit this site.
* `accounts.google.com` - Google will also map your visits to this site with any of your Google accounts.
* `pippio.com` - LiveRamp identify mapping for harvesting your details for commercial gain.
* `bluekai.com` - Identity tracking for marketing campaigns.
* `crwdcntrl.net` - Pretty suspect site listed as referenced by viruses and spyware.
* `exelator.com` - More identity and movement tracking site which even has a virus named after it!
* `doubleclick.net` - We all know this one: ad services and movement tracking, owned by Google.
* `tag.apxlv.net` - Very shady and [tricky to pin-point an owner](https://better.fyi/trackers/apxlv.com/) as they obsfuscate their domain (I didn't even know this was a thing!). Adds a tracking pixel to your site.
* `adnxs.com` - More tracking garbage, albeit slightly [more prolific](https://www.theguardian.com/technology/2012/apr/23/adnxs-tracking-trackers-cookies-web-monitoring).
* `adsymptotic.com` - Advertising and tracking that suppposedly uses machine learning.
* `rlcdn.com` - Obsfuscated advertising/tracking from Rapleaf.
* `adbrn.com` - *"Deliver a personalized customer journey across devices, channels and platforms with Adbrain customer ID mapping technology."*
* `nexac.com` - Oracle's Datalogix, their own tracking and behavioural pattern rubbish.
* `tapad.com` - OK, I cant't be bothered to search to look this up anymore.
* `liadm.com` - More? Oh, ok, then...
* `sohern.com` - Yup. Tracking.
* `demdex.net` - Tracking. From Adobe.
* `bidswitch.net` - I'll give you one guess...
* `agkn.com` - ...
* `mathtag.com` - Curious name, maybe it's... no. It's tracking you.

I can't visit many of these sites because I have them blocked in uBlock Origin so information was
gleaned from google crawl results of the webpages and 3rd parties. Needless to say, it's a pretty disgusting insight into how certain free products turn you into
the product. What's more worrying are the services that go to lengths to hide who they are and
what their purposes are for tracking your movements.

Anyway. Delete it. I'm sorry for poisoning this site with these networks. Foward to better things...

##### Using Github for Comments

I was discussing removing Disqus with others and [@HarryDenholm](https://twitter.com/HarryDenholm)
mentioned that a friend managed to get comments on their Github-hosted static blog using Pull Requests. I thought this was pretty genius and wanted to figure out if I could do something
similar for Hugo on external sites.

The answer came from the [Github Issue API](https://developer.github.com/v3/issues/). The process is quite simple and is already active on this blog post:

1. For each blog post you make, open an Issue on some repo on Github. The Issue for this post is [here](https://github.com/dwilliamson/donw.io/issues/1).
2. All comments are posted directly on Github.
3. Add some Javascript to your site that reads Github's JSON description of the Issue comments and displays them.

The benefits are immediate:

* There is zero tracking of you as a reader to this website. Github themselves only see network read requests from unnamed IPs.
* All comments are written in [Markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet), allowing inline code, images, lists and formatting.
* You can use Github's notifications to informed of responses; you don't even have to visit this site to read and contribute to discussion.
* Although somewhat superflous, you can also integrate [Github Reactions](https://developer.github.com/v3/reactions/) (maybe useful on larger sites).

You don't need an API key to read Github JSON data; it's free for all to access. The comments for
this post can be read as JSON [here](https://api.github.com/repos/dwilliamson/donw.io/issues/1/comments). The first comment looks like:

```json
{
    "url": "https://api.github.com/repos/dwilliamson/donw.io/issues/comments/295004846",
    "html_url": "https://github.com/dwilliamson/donw.io/issues/1#issuecomment-295004846",
    "issue_url": "https://api.github.com/repos/dwilliamson/donw.io/issues/1",
    "id": 295004846,
    "user": {
        "login": "dwilliamson",
        "id": 1532903,
        "avatar_url": "https://avatars3.githubusercontent.com/u/1532903?v=3",
        "gravatar_id": "",
        "url": "https://api.github.com/users/dwilliamson",
        "html_url": "https://github.com/dwilliamson",
        "followers_url": "https://api.github.com/users/dwilliamson/followers",
        "following_url": "https://api.github.com/users/dwilliamson/following{/other_user}",
        "gists_url": "https://api.github.com/users/dwilliamson/gists{/gist_id}",
        "starred_url": "https://api.github.com/users/dwilliamson/starred{/owner}{/repo}",
        "subscriptions_url": "https://api.github.com/users/dwilliamson/subscriptions",
        "organizations_url": "https://api.github.com/users/dwilliamson/orgs",
        "repos_url": "https://api.github.com/users/dwilliamson/repos",
        "events_url": "https://api.github.com/users/dwilliamson/events{/privacy}",
        "received_events_url": "https://api.github.com/users/dwilliamson/received_events",
        "type": "User",
        "site_admin": false
    },
    "created_at": "2017-04-18T22:39:16Z",
    "updated_at": "2017-04-18T22:39:16Z",
    "body": "This is a comment"
},    
```

The first step is to add a new template to your partials directory that reads and displays Github
comments (`comments.html`). Here's the code I use:

```js
var url = "https://github.com/dwilliamson/donw.io/issues/" + {{ $.Params.ghcommentid }}
var api_url = "https://api.github.com/repos/dwilliamson/donw.io/issues/" + {{ $.Params.ghcommentid }} + "/comments"

$(document).ready(function () {
    $.ajax(api_url, {
        headers: {Accept: "application/vnd.github.v3.html+json"},
        dataType: "json",
        success: function(comments) {
            $("#gh-comments-list").append("Visit the <b><a href='" + url + "'>Github Issue</a></b> to comment on this post");
            $.each(comments, function(i, comment) {

                var date = new Date(comment.created_at);

                var t = "<div id='gh-comment'>";
                t += "<img src='" + comment.user.avatar_url + "' width='24px'>";
                t += "<b><a href='" + comment.user.html_url + "'>" + comment.user.login + "</a></b>";
                t += " posted at ";
                t += "<em>" + date.toUTCString() + "</em>";
                t += "<div id='gh-comment-hr'></div>";
                t += comment.body_html;
                t += "</div>";
                $("#gh-comments-list").append(t);
            });
        },
        error: function() {
            $("#gh-comments-list").append("Comments are not open for this post yet.");
        }
    });
});
```

This can then be invoked from your blog post html:

```
{{ partial "comments.html" . }}
```

The variables it references need to be added to the header of your html page. In this case it's
the single variable `ghcommentid` which is set to the number of the Issue used for comments.

##### Summary

There's nothing more beyond that. Technically, you could actually write your blog posts as Github
Issues themselves and forgo the need for a blog engine. But that seems too much like using a system
in ways it was never intended to be used.

This website is managed on Github as [dwilliamson/donw.io](https://github.com/dwilliamson/donw.io).
It contains the complete code for using Github for your comments.
