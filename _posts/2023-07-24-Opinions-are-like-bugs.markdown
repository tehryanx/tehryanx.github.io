---
layout: post
title:  "Opinions are like Bugs - Every Spec has one."
date:   2023-07-24
categories: bugbounty
---

*When two specifications have differing opinions on how something should be parsed: here be dragons.*

This writeup is about a bug I first discovered over a year ago and have found a number of times since. There's nothing particularly novel about the vulnerability itself, but I think there's an interesting lesson in here about why it exists, and about where to look for other bugs that exist for the same reason. 

---

The first time I spotted this issue it was in a sanitizer library for ruby called [loofah](https://github.com/flavorjones/loofah). This is a tool commonly used in Ruby on Rails projects, and has shipped as a part of Rails core since v4.2. That finding let me escalate a strictly limited html injection to RCE. Below is what the html injection payload looked like, you may be able to guess how it worked.

```
<!--><iframe src=http://169.254.169.254/latest/meta-data></iframe>-->
```

The application I was testing allowed me to generate reports and export them as PDFs. After generating a sample and examining the metadata I knew what pdf generator was being used to accomplish this. A quick test of that tool proved that an SSRF should be possible if I could inject an iframe, as the generator would attempt to populate that iframe by loading it from the server. Injecting an image tag pointing to a burp collab resulted in a callback from an aws EC2 ip, so my plan was to try pulling IAM creds from the aws metadata instance. At first this failed. 

All of the inputs I could control were validated against a strict allowlist of safe html tags and attributes. Things like text formatting, images, and links were allowed but iframes were not.

I spent some time mapping out allowed tags and attributes, but the results didn't look very promising. After a lot of testing I had exhausted all obvious options, but I was so close I couldn't stop yet. One final possibility was still tickling my brain. 

I wasn't totally sure what was being used to scrub my input, and I wanted to thoroughly test the sanitizer before calling it complete. I knew it was a ruby on rails application, and that rails made Loofah available for sanitizing HTML input. I pulled open rails console and starting pushing on Loofah in every direction trying to find interesting behaviour. Of all the tags that were allowed by the application I was testing, the one that stood out the most was the HTML comment tag `<!-- -->`. I wondered if it might be possible to leverage a parsing differential such that loofah would see the iframe as being inside a comment, but the pdf generator would not. 

The test was fairly simple to implement. I generated a wordlist using the following oneliner, and generated a pdf report containing that wordlist. I expected most of them to be considered comments by the PDF generator, but if any of the iframes were rendered I should get a nudge on my burp collaborator. 

```
ruby -e "0.upto(65535) {|i| puts '<\!--' << i << '<iframe src=burpcollap/?id=' << i << '></iframe>-->'}"
...
<!--[<iframe src=burpcollab/?id=91></iframe>-->
<!--\<iframe src=burpcollab/?id=92></iframe>-->
<!--]<iframe src=burpcollab/?id=93></iframe>-->
<!--^<iframe src=burpcollab/?id=94></iframe>-->
<!--_<iframe src=burpcollab/?id=95></iframe>-->
...
```

To my surprise, I got a hit on `burpcollab/?id=62`, the codepoint for the greater-than character `>`. This allowed me to bypass the sanitizer, which meant I could inject an iframe, steal the AWS creds from the metadata instance, and escalate to RCE. I still wanted to understand what was happening to cause the two HTML parsers to behave differently. 

![image](https://media4.giphy.com/media/zrmTqopWm4W5cPg8Ah/giphy.gif?cid=ecf05e47y5ni6z9jd9m44cw1nvjr16cx3cpbw5zi8tmblrgl&ep=v1_gifs_search&rid=giphy.gif&ct=g)

# But why?

To investigate what was going on I pulled open my ruby interpreter to see what this input looked like from Loofah's perspective. I started by defining a loofah scrubber that would take an html fragment and remove any instances of the <iframe> tag.

```
irb(main):001:0> require 'loofah'
=> true
irb(main):002:1* iframe_scrubber = Loofah::Scrubber.new do |node|
irb(main):003:1*   node.remove if node.name == "iframe"
irb(main):004:0> end
```
I tested this on a normal iframe, and it behaved as expected:
```
irb(main):005:0> html = "<h1>iframe test</h1><iframe></iframe>"
irb(main):006:0> Loofah.fragment(html).scrub!(iframe_scrubber).to_s
=> "<h1>iframe test</h1>"
```
Then I tested it with a normal comment:
```
irb(main):007:0> html = "<h1>iframe test</h1><!--<iframe></iframe>-->"
irb(main):008:0> Loofah.fragment(html).scrub!(iframe_scrubber).to_s
=> "<h1>iframe test</h1><!--<iframe></iframe>-->"
```
And finally, with the modified comment discovered by our fuzzer:
```
irb(main):009:0> html = "<h1>iframe test</h1><!--><iframe></iframe>-->"
irb(main):010:0> Loofah.fragment(html).scrub!(iframe_scrubber).to_s
=> "<h1>iframe test</h1><!--><iframe></iframe>-->"
```
As expected, Loofah see's the iframe tag in both of the the last two examples as being safely tucked inside a comment. However, when this exact same code is passed to the pdf generator, the iframe is rendered. This means that whatever is parsing the HTML in the pdf generator parses this in such a way that the iframe tag is not inside the comment. 

Even better, the html parsers in modern browsers agree with the pdf generator. Open a page with this html in chrome, firefox, edge, etc, and you'll see a rendered iframe:
```
<!--><iframe src=about:blank></iframe>-->
```

At first glance, this seemed like a bug in the html parser that Loofah is using, but after some investigation it turned out to be something much worse: Competing specs with different opinions. 

![image](https://github.com/tehryanx/tehryanx.github.io/assets/8878295/c57b60f0-9e28-496b-b6e5-00062a7daba1)

---

# Competing Specs

Loofah uses an html parsing framework called Nokogiri, which is built on libxml2. Libxml2 is a very popular xml (and html) parsing library that tries to be compliant with the [W3C html 5 specification](https://www.w3.org/TR/2011/WD-html5-20110405/tokenization.html). The problem is that this library is very old, and pieces of it's parsing functionality are much older than the most recent version of the standard. 

The library's comment parsing functionality, for example, was [implemented in 2000](https://github.com/GNOME/libxml2/blame/75693281389aab047b424d46df944b35ab4a3263/HTMLparser.c#L3455). At that time, there wasn't any clear guidance in the standard around how to parse comments. The way the libxml2 authors interpreted the specification resulted in the following steps:

- the tokenizer encounters the opening comment sequence `<!--` 
- Move the tokenizer forward until et encounters the closing comment sequence `-->` 
- treat everything inside as a comment. 

This means that, given our sequence `<!-->`, the `>` would be ignored and everything up to `-->` would be a comment. This explains why Loofah saw our iframe as being inside a comment. 

Eventually, the rules changed and as of the first completed html5 spec published in 2008, the rules for parsing comments were as follows:

- We start in "tag open state," and we encounter an `!`, so we swtich to "markup declaration state"
![image](https://github.com/tehryanx/tehryanx.github.io/assets/8878295/910ae211-55c1-418c-bc4b-e451f3f550c1)


- If the next two characters are `--` we switch to `comment start state`
![image](https://github.com/tehryanx/tehryanx.github.io/assets/8878295/cf9df270-cfb4-4e7b-a16f-372f54112f52)


- And finally, if the very next character is `>`, we throw a parse error and switch back to "data state" instead of "comment state".
![image](https://github.com/tehryanx/tehryanx.github.io/assets/8878295/3772de46-3a2c-4d2f-97ea-c5abd2a36258)


Now, given the sequence `<!-->`, the comment is opened and then immediately closed. Everything following the `>` is outside the comments. This explains why the PDF generator, and modern browsers, render the iframe. 

---

This issue has been patched in the most recent version of libxml2, but I still find it frequently. Any HTML parsing framework that depends on an old version of libxml2, or was developed against an older version of the HTML5 spec is likely vulnerable to this bug. For example, beautiful soup is another parsing framework that follows the old rules for parsing comments: 

```
>>> import bs5
>>> soup = bs4.BeautifulSoup("<iframe></iframe>", "html.parser")
>>> soup.iframe
<iframe></iframe>
>>> soup = bs4.BeautifulSoup("<!-- <iframe></iframe> -->", "html.parser")
>>> soup.iframe
>>> soup = bs4.BeautifulSoup("<!--> <iframe></iframe> -->", "html.parser")
>>> soup.iframe
>>>
```
Note that the last two examples, the normal comment and the abruptly closed comment, are treated the same. This has been reported but no fix has been implemented as yet. 

---

# Lesson

The lesson I learned from this research is that when you're dealing with complex functionality that depends on a specification like an RFC, you should read it. These documents can be a slog but the more you read the better you'll become at parsing and understanding them. There are so many subtle details hidden away in standards that are just waiting to be stumbled on by the right hacker with the right mindset. Keep your eyes peeled for ambiguous language, major changes across versions, and situations where more than one standard exists and your input is being consumed by both of them at different times.
