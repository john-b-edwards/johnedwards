---
date: "2021-10-16"
description: An introduction to web-scraping with RSelenium
draft: true
keywords:
- r
- web_scraping
- rselenium

math: true
pagination: true
slug: rselenium_scraping
tags:
- r
title: Scraping with RSelenium
toc: false
---

Despite what the majority of tutorials may demonstrate, web scraping is rarely as elegant or straightforward as we may like it to be. Web-page structures are often messy, complex, lacking a straightforward API, and are sometimes deliberately designed to be hostile to scraping. When all else fails, RSelenium is an impressive tool to have in your toolbox for scraping -- a nuclear option that is almost certain to give you access to the data you need in a fairly brute-force way. Today, we'll go over how to get set up with RSelenium and showcase how to use it for scraping. Next week, I'll show you how to use RSelenium to automate any tasks you have involving the web. But before we go into RSelenium, a word to the wise...

##  READ THIS BEFORE TRYING TO USE RSELENIUM

It is important to understand that RSelenium _is_ the nuclear option for web scraping. RSelenium is much less elegant than other scraping techniques, runs a good bit more slowly, and can be fairly finnicky. Before using RSelenium to scrape something, you should check that the source you are attempting to scrape:
* *Does not have a readily accessible API.* A website may have a readily accessible API available and documented -- friend of the blog CollegeFootballData.com [has an easy to access, free API with extensive documentation](https://api.collegefootballdata.com/api/docs/?url=/api-docs.json). It would be absurd to use RSelenium to scrape CFBD.com when such an API exists.
* *Does not have a hidden API that you can access.* Many websites use an API as a backbone but do not make the documentation for the API publicly available -- generally because the API is not designed to be used by anything other than the website. Regardless, you can generally find API requests that a website makes by going to the page you are attempting to scrape in your web browser, pressing `F12`, selecting the "Network" table on the window that opens, then refreshing the browser page -- you will see the requests the page is making of any APIs in real time. You can read a more in-depth tutorial on how to do this [here.](https://ianlondon.github.io/blog/web-scraping-discovering-hidden-apis/)
![ESPN does not make their API documentation publicly availabe, but you can still access example requests with the developer window in your browser](images\rselenium_scraping\espn_screenshot.png).
* *Does not play well with existing scraping packages*. There are a variety of reasons as to why a page might not work with something like `rvest`, many of them due to unusual ways of loading in data, password protection, etc. In general, however, if `rvest` works for what you're trying to do, you should use `rvest`.

Ensure that the page you are trying to scrape meets the criteria above before trying to use RSelenium.

## An unscrapable website!

As an example where it would be appropriate to use RSelenium, we're going to work with an "unscrapable" website that is still very clearly open and cool with us accessing it to download the data. In this example, we will be grabbing Overwatch League data from the [OWL's website](https://overwatchleague.com/en-us/statslab). Before proceeding, let's go ahead and make sure the OWL website fits our criteria for using RSelenium.

### Does this site have a readily accessible API? 

![](images\rselenium_scraping\owl_api_searching.png).

A cursory glimpse reveals that there is no mention of a public, documented API on the Overwatch League website for accessing OWL data.

### Does this site have an accessible hidden API?

![](images\rselenium_scraping\owl_hidden_api.png).

When we go to access the OWL website, there are no immediately available APIs that would give us the kind of stats data we're looking for. Instead, the values are stored in compressed csv files linked to on the website.

### Does this site play well with rvest?

So, the files are linked to on the OWL website already -- that should make them fairly easy to access with `rvest`, right? Let's fire up a quick scraping script and see if we can grab them here.

```r
if (!requireNamespace('rvest', quietly = TRUE)) {
  install.packages('rvest')
}
owl_statslab <- rvest::read_html('https://overwatchleague.com/en-us/statslab')
owl_statslab |>
  rvest::html_nodes('a')
#> {xml_nodeset (0)}
```

Hrm. We didn't find any links, which is odd because the OWL page obviously has links on it.

```r
owl_statslab <-
  rvest::read_html('https://overwatchleague.com/en-us/statslab')
owl_statslab |>
  rvest::html_node('body')
#> {html_node}
#> <body>
#>  [1] <div id="__next"><div class="indexstyles__PageContainer-pu1n5t-1 cvzdyC" ...
#>  [2] <script id="__NEXT_DATA__" type="application/json">{"props":{"pageProps" ...
#>  [3] <script nomodule="" src="/_next/static/runtime/polyfills-0953d39a5ed293a ...
#>  [4] <script async="" data-next-page="/_app" src="/_next/static/SGyqsgIqgrfxe ...
#>  [5] <script async="" data-next-page="/" src="/_next/static/SGyqsgIqgrfxeX_2a ...
#>  [6] <script src="/_next/static/runtime/webpack-a8873ab33cb397f9c2f1.js" asyn ...
#>  [7] <script src="/_next/static/chunks/framework.dbe2955122a46e30de87.js" asy ...
#>  [8] <script src="/_next/static/chunks/commons.50d4e0098be0a1e7bd23.js" async ...
#>  [9] <script src="/_next/static/runtime/main-9e628749a686592eb4fb.js" async=" ...
#> [10] <script src="/_next/static/chunks/ee94fed1a5db69db22cf056cb84606f24f5b27 ...
#> [11] <script src="/_next/static/SGyqsgIqgrfxeX_2aUvFm/_buildManifest.js" asyn ...
#> [12] <script src="/_next/static/SGyqsgIqgrfxeX_2aUvFm/_ssgManifest.js" async= ...
```

Upon closer examination, this page is composed not of standard html elements, but of multiple scripts that dynamically render content when the page is loaded. Unfortunately, `rvest` can't access this data alone. We can use RSelenium to find what we're looking for!

## Setting up RSelenium

Setting up RSelenium is a twofold process. The first step is to install the RSelenium package if you haven't done so already:

```r
if (!requireNamespace('RSelenium', quietly = TRUE)) {
  install.packages('RSelenium')
}
```

The second step is to set up a server for RSelenium. You have two options for this:

1. You can use a Docker container to run a Selenium server
2. You can locally run Selenium and rely on a browser to run Selenium

RSelenium recommends that you use the Docker option, but I personally prefer the browser version, as it requires less time to set up and works more easily with things like file uploads -- we'll use the second version for this tutorial, but you can see how to set up Docker [here](https://docs.ropensci.org/RSelenium/articles/basics.html) and browsers [here](https://docs.ropensci.org/RSelenium/articles/saucelabs.html). We are using Firefox, which works out of the box for Selenium.

```r

```



