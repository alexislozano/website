+++
title = "Self-hosting, the basics"
date = 2020-12-02
+++

With the annoucement of Whatsapp and Facebook sharing their data with eachother, I wanted to talk a bit about self-hosting. Ok, so first question...

## How does the Internet work?

To answer this question, let's dive into the past, to a land of ghouls and ghosts. Let us go back to the beginning of the Internet. The Internet, back then, was technically almost the same as now: interconnected websites. If you wanted to go to a website, you could use a search engine or write down somewhere the address of the website. After a while, you were welcomed with text and sometimes pixelated images, yay! Internally, your web browser sends a request to a server which sends it to another server and so on until the request comes to a server hosting the website. The request is then handled, and a response is sent back to your web browser which is finally able to display the website you wanted to browse for so long.

Actually, it still works like this. Browsing Facebook ? Requests sent. Browsing Twitter? Requests sent. I could go all day, but it won't be much interesting. Now we get another question.

## Hosting?

Yes, Facebook, Instagram, every website or webapp is hosted somewhere, on a server. A server is basically a computer handling requests. If you are Facebook, you have a lot of servers. If you have just a simple website like the one you are reading this article on, you probably only use one server. And you know what? Serving a website is something pretty simple nowadays. Okay, you'll need to understand some things, but it is not this complicated ;)

## Why do I need to self host?

That is the real question. Actually, you do what you want. You can use a service which is hosted by others. Facebook, Gmail, all these services are hosted by companies and you can create an account often for free. Now you may wonder: How do they make maney if it is free? Simple, they sell information about you for announcers to be able to create ads that target you. By self-hosting some services (yes, you won't be able to host your own Facebook), you keep your data on your server and no one sells it. You can sell it if you want actually, but it is not mandatory.

## Okay, and are you self-hosting services?

Hmmm... Oh, you are asking me? Well, here are the services I self-host and what I use them for:

- [nextcloud](https://nextcloud.com/) : sync files between devices
- [jellyfin](https://jellyfin.org/) : watch movies and series (and animes!)
- [miniflux](https://miniflux.app/) : read news thanks to rss feeds
- [bitwarden](https://bitwarden.com/) : store and create passwords
- [nginx](https://www.nginx.com/) : serve this website and forward requests to the previous services

I am finished for today, let us talk about self-hosting more thoroughly next time ;)