# That SQL looks pretty complicated. Where are the tests?

My standard development technique with PostgreSQL is to hack away in
Emacs, try things out on live data, and eventually build up working
queries and functions. This "test as you go" approach works well for
small projects, but for really hard problems the result is usually
hundreds of lines of SQL that looks a lot like an object lesson in
unmaintainable code.

Old hackers can learn new tricks, and I've been trying to be more
disciplined about writing tests. While I have a good workflow in Perl,
R, and JavaScript, my PostgreSQL code has remained stuck in
hacker-mode.

This talk is about testing PostgreSQL functions and queries. I was
going to talk about how I create modular functions, write tests of
those functions in node.js, and then build up larger functions from
those parts. But then, as I prepared this talk proposal, I discovered
pgTAP (http://pgtap.org/). So my talk is about how I adopted pgTAP and
use that to test my SQL functions. The talk is heavy on examples,
taking real code I've written in the course of geocoding highway and
street facilities in California.
