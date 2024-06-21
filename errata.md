# Errata

Here's a little follow-up/postmortem on my web scraping patterns talk.

I made a minor fix: I added `const el = ` to "Using untrusted clicks when scraping" section.

That section on untrusted events generated some good feedback from [Joel](https://www.browserconference.com/speakers/joel-griffith/), which is that using untrusted clicks can make your bot easier to detect and risk blocking. This is an interesting point, and I didn't have time to address it fully, so I thought I'd explore it in detail here for posterity. I'm open to further feedback and discussion on the issue.

For context, I don't generally try to bypass site blocks, other than user agent spoofing, or maybe running headfully. I'm mostly interested in optimizing automating sites that don't seem to mind being scraped (or are sites owned by me, or someone I work for, i.e. testing), which is most sites (albeit maybe not the high-value ones). If a site throws up a Cloudflare block or captcha, I usually don't bother with it. As such, I'm not an authority on bypassing blocks.

With that context in mind, having scraped many hundreds of sites, I don't recall having encountered any that have obviously blocked me on the grounds of untrusted events, or that have obviously been accepting of trusted event automation but not untrusted event automation. Most blocks were either up-front captchas or headless detection, or likely rate limits based on too many requests. I imagine blocks would be based mostly on the metrics from a site like https://bot.sannysoft.com/, rather than things like the usage of untrusted events or detecting direct DOM manipulation.

I've also userscripted a variety of popular sites and haven't experienced any throttling or blocking even though my userscripts use untrusted events extensively. If sites blocked users based on untrusted events alone, I'd likely have experienced that by now. An exception may be YouTube, which I have seen obvious throttling on, but they're a rare case of a company having maximal resources to block automation. I suspect my usage of ad blockers and Firefox are the cause of any throttling I've seen on YT, though, not necessarily my userscripts which mostly rip out recommendations and sidebars (but may be mistaken for ad bypassing--or YT doesn't want to allow _any_ automation, to be on the safe side).

In my talk, I suggested a number of other patterns for improving scraping reliability and speed beyond using untrusted events: blocking resources, disabling JS, etc. Many of these could just as much be used as bot detection measures. The same holds for any optimization that speeds the automation up unnaturally. So in the common case, I suggest doing the optimizations, but if blocking is likely, then go ahead and use trusted events and avoid optimizations to the extent necessary to avoid detection. If a site uses very aggressive detection and blocking, the automation could look something more like my recommendations for testing--acting like a user as much as possible.

In other words, the high-level antipattern is assuming the worst, and running super-slow, human-like scraping littered with random sleeps, mouse wiggling, and trusted events by default. I've rarely, if ever, seen this to be an effective way to bypass a block, and I don't know of any examples of sites that look at mouse behavior or trusted events to determine when to block. Happy to hear of one though!

In the talk, I mentioned that it'd be a bit difficult to detect untrusted events, that you'd probably use proxies, which isn't entirely true. Events have the [`isTrusted`](https://developer.mozilla.org/en-US/docs/Web/API/Event/isTrusted) property which can be easily read in a listener. I'm sure there are libraries to auto-detect this without having to instrument every event listener, but the company would still need an endpoint to collect the data from these untrusted events. All that takes effort that I doubt most companies will bother investing in--I don't know of any offhand.

Another consideration: if the site is making API calls from event listeners to record trusted events or other bot-like behavior, it seems easy enough to block these requests. Scanning input .js files for `isTrusted` also seems possible as a counter-measure. If a source file has `isTrusted`, switch to trusted events.

Also, all the usual residential proxy rotation still applies. If a scraper is blocked on the grounds of using trusted events, but the IP is different on every request, there should be no problem.

So for now I stand by my recommendation that trusted events are an antipattern in scraping, but I'd love to see some examples where it's not true, and perhaps I can be talked into dialing the recommendation back. The visibility problems of using trusted clicks for scraping are so rampant that in most cases, using them as the default for scraping seems like a sound principle, well worth the slight risk of an occasional block.

One can also take the recommendation on an as-needed basis: if a trusted click fails, switch to untrusted. This is actually what I do in practice when I use Puppeteer for scraping, because trusted clicks feel more idiomatic and the syntax is simpler, although I'm working on a library that uses untrusted by default.

There's also `{force: true}` in Playwright's click options, which bypasses actionability checks, but still issues a trusted event (see example below) and seems like a sensible default in web scraping. That said, I feel like I've seen this fail relative to a true untrusted browser JS click, but don't have evidence offhand.

On a deeper level, I make a number of recommendations to avoid interacting with buttons in the first place, like using `goto` to navigate between routes whenever possible. This seems unlikely to be detected, and adds a good deal of reliability in return.

Another way to trigger the same result as a trusted click, but without as much of a visibility problem, might be to use the keyboard to Tab over to the correct element (checking at each step by text/element properties), or focusing on it programmatically, and triggering an Enter keypress. There are probably other creative ways to avoid clicking entirely.

In any case, hopefully the recommendation (and talk in general) provided some food for thought and discussion.

---

Proof that `force: true` is trusted:

```js
import playwright from "playwright"; // ^1.42.1

const html = `<button></button><div></div><script>
document.querySelector("button").addEventListener("click", event => {
  document.querySelector("div").innerHTML = \`<p>\${event.isTrusted}</p>\`;
});
</script>`;

let browser;
(async () => {
  browser = await playwright.firefox.launch();
  const page = await browser.newPage();
  await page.setContent(html);
  await page.locator("button").click({force: true});
  console.log(await page.locator("p").textContent()); // => true
  await page.locator("button").evaluate(el => el.click());
  console.log(await page.locator("p").textContent()); // => false
})()
  .catch(err => console.error(err))
  .finally(() => browser?.close());
```
