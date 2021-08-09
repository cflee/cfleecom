+++
title = "Two years on: the Three-Fund Portfolio costs calculator"
+++

I've been lurking on the Shiny Things fan club threads on the HardwareZone Forums for some time. 
It's been so long that I've forgotten when I started -- must have been late 2018 or early 2019, since it was in "part 2" of the megathread.

Josh Giersch aka Shiny Things has basically been explaining and re-explaining his localised-for-Singapore version of the Bogleheads [three-fund portfolio](https://www.bogleheads.org/wiki/Three-fund_portfolio) to everyone that engages with him.
And he's been doing it for years.
With more patience than I could possibly muster.

One very common question was -- given the recommendation (back then) to buy IWDA, ES3/G3B, and A35: for $x monthly amount, with this or that asset allocation, which broker should I use?
What if I batch it up to every y months?
While some rules of thumb were used to try and address this, it is never quite enough, for both the people that want to be told what to do, as well as the people who would calculate it down to the cent and then quarrel with the rule of thumb (it me!).

Sometime in Aug 2019 I put together a spreadsheet to make it more efficient for me to calculate these numbers and reply to people.
Not too long later, but definitely after using it for a while to see if [somebody on the internet lets me know that my calculations are wrong](https://xkcd.com/386/), I published it on Google Sheets as the [ShinyThings Three-Fund Portfolio cost simulation](https://docs.google.com/spreadsheets/d/1BqIhbYNJF7VJbtj6PS08Csbyx1s4vs_03Kvh0pPRUCI/edit?usp=sharing), and dropped it into the forum thread.

About a year later in Jul 2020, I branched a copy off to do some major expansion triggered by the launch of Interactive Brokers in Singapore and their expansion to allow Singapore residents to trade on the SGX, and published it as the **[v2 spreadsheet](https://docs.google.com/spreadsheets/d/1T0SnPLJxdzhi5kLIK2zxbWLGkX6ILfFLsIlhzG8o534/edit?usp=sharing)**.

Now, two years in, I occasionally see it popping up on the new-ish [r/singaporefi](https://www.reddit.com/r/singaporefi/) subreddit, though occasionally in the wrong contexts.
It's not at all suitable for picking a broker for buying U.S.-listed stonks and SPY.

## In retrospect

My hope was that it'd make things more efficient, whether for someone using it to quickly confirm a recommendation for someone asking in the thread, or someone to directly use it to help make their own decisions.

- Since 25 Aug 2019, the v1 spreadsheet has clocked 848 unique viewers. It peaked at 101 monthly unique viewers in Jan 2020, in particular around 5 Jan 2020. Perhaps a new year's resolution thing? It's been holding at around 30+ monthly unique viewers even after Jul 2020, until it finally started tailing off after May 2021.
- Since 24 Jul 2020, the v1 spreadsheet has clocked 500 unique viewers. There's been a recent surge to 114 monthly unique viewers in Jul 2021, probably after it was updated to reflect IBKR's removal of the monthly minimum commissions.

It's a bit difficult to try and quantify the impact this actually had.
These aren't large numbers at all.
But since I wanted this spreadsheet for myself anyway, and fortunately publishing this has not been a huge maintenance / support burden, so no real regrets.

That said, one thing that has been bugging me for some time was the early decision to not move ahead with wrapping this up into a more user-friendly web UI, whether as a step-by-step wizard like the [International Boglebot](https://www.boglebot.com/play) or as a calculator with charts like all those FIRE calculators.
Instead, I doubled down and stuck with the spreadsheet format.

The idea was that a spreadsheet is something that power users can both inspect and extend, since Excel formulas are more widely understandable than a file of JavaScript code.
However, in the single digits of feedback received about problems in the sheet, there's been exactly one report that came with a detailed analysis of exactly which bit went wrong. 
The others were all reporting based on the observed output vs their expected output, so can't really tell if having it be a spreadsheet had helped at all.

On the extensions front, no way to tell for sure whether anyone has extended this, but haven't seen anyone make derivative works from it so far.
It's possible that anyone who would do spreadsheet modeling would prefer to do their own from ground up, since then it'd make sense to them and do exactly what they need, without any extra complexity that is built into a one-size-fits-most thing.

With all that in mind, I don't really know if it has been worth the downsides: a poorer experience for users, who keep having to make a copy to pick up the latest changes, plus the "UI" is getting more cluttered and confusing as it gets more comprehensive; and on my end it has gradually become a bit precarious to maintain given the challenges with automatically testing Excel sheets.
Recently I had also thought about extending it to calculate total fees over time (e.g. starting lump-sum amount + periodic amounts), shuddered at the thought of ensuring and maintaining the correctness of that in Excel cells instead of in code, and dropped it.

Perhaps this conflict or confusion is because I've been unclear about the goal here.
Is this about helping the very financially literate (whether in the form of financial advisors, or people on forums that play the role of free advisors) weigh decisions more efficiently than they would otherwise, or about helping others in general make better decisions?
If it's the former, which is my own use case, this sort of works, once the correctness is sorted out.
If it's the latter, this might not be making much difference at all -- the individual engagement and re-explaining in different words and addressing their specific questions is what convinces people to make any decision at all, and which broker to use is really just a minor execution detail afterwards.

## Looking ahead

I think there are some other interesting things that could be tackled in this personal finance space.

Financial planning or projections in a Singaporean context, taking into account our rather special CPF system, is probably an interesting one, given all the questions that always crop up:

- Should I pay for housing with CPF OA or with cash?
- Should I do CPF top-ups?
- Should I invest my CPF OA?
- Should I transfer from CPF OA to CPF SA?
- Should I just assume that my CPF account doesn't exist when calculating whether I can FIRE?

There are some commercial attempts at this, like [GoalsMapper](https://www.goalsmapper.com/) that I'm told has been around for some time and is fairly comprehensive but only sells access to financial advisors, and [BetterTradeOff](https://www.bettertradeoff.com/) that seems to have a newer consumer-facing [UpPlan](https://www.upplan.sg/) tool.
But it would be nice to have a tool that's freely available, and freely extensible.
