---
title: Set up an (almost) free static site with hugo and netlify
date: 2019-06-30 17:28:59
categories: guide
---

This will be a small guide on how I set up this site with hugo and (Netlify)[https://netlify.com]. I previously used Github Pages but I'm no friend of Jekyll, plus customizations options where limited.

Netlify is free for small teams, and will hapilly produce the HTML/CSS output from your repository with many supported static site generators. I chose Hugo because it has a binary so I don't need to worry about dependencies, and besides I'm friendly with the Go language in case I need to hack something.

## Domain 
So, first things first. We need to get a domain. This is the "almost" free part, and is where unfortunately Github pages is better. The subdomain Netlify gives you is something like `random-word-hyphen-numbersomething.netlify.com`, which is uglier than `myblog.github.io`. But having a domain is useful for many things and ~U$10 a year is not a lot. Go to your favorite domain provider like (NameCheap)[https://namecheap.com] or let Netlify sell you one. 

I used (Name.com)[https://name.com] but this turned out to be a bad decision, I'd use Namecheap now.

Everything works

## Set up your repository

I created a repository on Github, but Netlify supports Gitlab and other options too. Mine is a simple (Github repository)[https://github.com/joaquinlpereyra/blog].

## Download Hugo

(Install Hugo)[https://gohugo.io/getting-started/installing/]. Mind which version you install, it'll be important in a while.

## Create a new site in Netlify.

Go to your Netlify account and create a new site from Git. This part is pretty straightforward. It will ask you if you already have a domain and you'll say yes.

Just mind the `Build Options` part. In the `Enviorenment variables` section, you'll need to specify which version of Hugo you are using.  The key is `HUGO_VERSION` and the value should be the full numerical Hugo version, like `0.55.6`. 

## Set up the DNS records and nameservers for your domain

Things vary a bit here depending on if you chose your domain on Netlify to be with or without `www.` (Netlify docs)[https://www.netlify.com/docs/custom-domains/] do a great job here, though, so I won't repeat them, but in short, you need to change your authorative nameservers and create two new DNS records pointing to Netlify.

## Download a theme for Hugo and create some content!

This site uses the great (Hugo Xmin)[https://github.com/yihui/hubo-xmin] as theme. Go look for something you like. This is probably the messiest part of the ordeal, as theme configuration can be quite confusing. Try to just make it work and tweak it as little as possible in the beginning until you're more confortable with how Hugo does things.

Now just create some content! 

It should be as easy as `hugo new post/YYY-MM-DD-Your-Title.md`. Edit that file, write whatever you want, and push to Github! Netlify should process the files for you and create the HTML out of them.
