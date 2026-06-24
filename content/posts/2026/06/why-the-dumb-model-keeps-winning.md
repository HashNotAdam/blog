---
title: "When have you earned the right to be clever?"
date: 2026-06-23T09:00:00+10:00
slug: "when-have-you-earned-the-right-to-be-clever"
description: "Why simple heuristics out-predict complex models, and what overfitting has to say about it"
keywords: []
draft: false
tags: [ideas]
math: false
toc: false
---

You're in a city you don't know, you're hungry, and there are two restaurants on the same street. One is empty. The other is packed, with a small queue starting to form. You pick the packed one... and so does everyone else, without a second thought.

Stop and notice how little you used to decide that. You ignored the menus. You ignored the prices, the décor, the star ratings, the four-paragraph review someone left about the calamari. You used exactly one cue and threw the rest away. And you were almost certainly right.

That instinct looks like laziness. It isn't. A full restaurant is a few hundred strangers' independent verdicts compressed into a single signal you can read in a second, and it quietly beats the careful approach far more often than it has any right to. Which is an uncomfortable thought for anyone who builds models for a living: if throwing away almost all the information keeps winning at dinner, where else is it winning?

That's this whole post in one meal: that a rule simple enough to scribble on a napkin will, more often than is comfortable, out-predict the model you spent a whole sprint tuning.

Let me start by respecting the opposing view, because it's a sensible one. The intuition suggests that a decision is only as good as the information behind it, so more inputs, more parameters, and more computation should give you a better answer. Throw more cues at the problem, fit a richer curve, and you'll track reality more closely. That instinct is correct often enough that it's become the default. Reach for the sophisticated thing, and treat the simple thing as the version you'll replace once you have time.

I want to complicate that. Not overturn it — complicate it. Because there's a large body of evidence saying the simple thing frequently _is_ the better answer, and the reason why turns out to be the same reason your model overfits.

## The shortcut story we all learned

If you've read any pop psychology in the last fifteen years, you've absorbed the standard account of heuristics — the mental shortcuts we use to make snap judgements. It mostly comes from Daniel Kahneman and Amos Tversky, and the popular distillation is [_Thinking, Fast and Slow_](https://en.wikipedia.org/wiki/Thinking,_Fast_and_Slow). The framing there is a fast, intuitive "System 1" that fires off cheap answers, and a slow, deliberate "System 2" that does the careful work. Heuristics belong to System 1, and the headline finding is that they leak. They produce _biases_, the predictable errors where our gut quietly misleads us.

This is genuinely useful work, and I'm not here to sledge it. Anchoring, availability, and base-rate neglect are real and they bite. But notice the flavour it leaves behind: the heuristic is the cut-rate option. It's what your brain does when it can't be bothered with the proper calculation, and the price you pay is bias. Simple is framed as second-best.

## The plot twist

There's a counter-tradition, led mostly by [Gerd Gigerenzer](https://en.wikipedia.org/wiki/Gerd_Gigerenzer), that looks at the same shortcuts and draws the opposite conclusion. While the debate started in 1996, Gigerenzer and Brighton poured fuel on the fire with [_Homo Heuristicus_](https://onlinelibrary.wiley.com/doi/10.1111/j.1756-8765.2008.01006.x)<sup>[1](#fn1)</sup>, reaffirming the contrarian claim: heuristics aren't just cheap fallbacks that _cost_ you accuracy. Under the right conditions, a heuristic that deliberately ignores information will be _more_ accurate than a strategy that uses all of it. They call it the _less-is-more_ effect, and it's not a turn of phrase, it's a measured result.

Start with a daft one. Suppose you want to predict the outcome of a Wimbledon match, and your method is this: of the two players, pick the one whose name you recognise. That's it. No rankings, no form, no surface preference — one bit of information, and you throw everything else away. It should be useless. Yet when researchers tested recognition like this against the official seedings and the ATP rankings, [it predicted winners about as well](https://en.wikipedia.org/wiki/Recognition_heuristic)<sup>[2](#fn2)</sup>. A rule a child could run kept pace with the machinery the sport uses to rank professionals.

That's a fun party trick. The example that should make an engineer sit up is about money.

In finance, the textbook way to build a portfolio is mean-variance optimisation. You estimate each asset's expected return and how the assets move together, then solve for the mathematically optimal weights. It's elegant and it won a Nobel Prize. The napkin alternative is the _1/N rule_: split your money evenly across everything and go home. In a much-cited study, DeMiguel, Garlappi and Uppal put fourteen flavours of the clever optimiser up against naive 1/N across a stack of real datasets — and [none of them reliably beat the even split out of sample](https://academic.oup.com/rfs/article-abstract/22/5/1915/1592901)<sup>[3](#fn3)</sup>. The optimiser had to _estimate_ all those returns and covariances from a finite, noisy history, and the estimation error swamped whatever edge the optimisation was supposed to buy. The even split estimates nothing, so it has nothing to get wrong. The story goes that Harry Markowitz, who won the Nobel for the optimisation, used 1/N for his own retirement savings.

Hold that phrase — _nothing to get wrong_ — because it's the whole argument.

## So why does the simple thing win?

If you write software for a living, you already have a word for what's happening to the clever model, even if you've only met it in a machine-learning context. It's overfitting.

Picture a student who prepares for an exam by memorising last year's paper. Word for word, they're flawless on it. Then they sit this year's paper, the questions are different, and they fall apart. They never learned the subject, they learned _that specific paper_, including the bits that were just quirks of how it happened to be written. They fit the practice data perfectly and the real test terribly.

A complex model does exactly this. Give it enough knobs and it will bend itself to pass through every point in the data you trained it on, including the noise, the flukes, the one-off wobbles that won't repeat. It looks brilliant on the data it has already seen and stumbles on the data that actually matters: tomorrow's.

The formal way to talk about this is the _bias–variance trade-off_, and it's worth knowing the shape of it even if you never write down the maths. Your expected error on new, unseen data splits into roughly three parts:

- **Bias:** Systematically wrong. Tight, but aimed at the wrong spot. More data won't save it.
- **Variance:** Over-sensitive to the sample you saw. Centred, but the aim is wild.
- **Irreducible noise:** The genuine randomness in the world that no model can ever explain.

<div style="display:flex; gap:1rem">
  <!-- Bias -->
  <svg data-dc-tpl="16" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 220 220" role="img" aria-label="Bias: tight cluster of shots landing away from the centre">
    <circle data-dc-tpl="17" cx="110" cy="110" r="102" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.15" data-om-id="bad56d67:22"></circle>
    <circle data-dc-tpl="18" cx="110" cy="110" r="68" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.15" data-om-id="bad56d67:23"></circle>
    <circle data-dc-tpl="19" cx="110" cy="110" r="34" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.15" data-om-id="bad56d67:24"></circle>
    <circle data-dc-tpl="20" cx="110" cy="110" r="3.5" fill="currentColor" opacity="0.45" data-om-id="bad56d67:25"></circle>
    <g data-dc-tpl="21" fill="#d6442b" data-om-id="bad56d67:26">
      <circle data-dc-tpl="22" cx="74" cy="84" r="8.5" data-om-id="bad56d67:27"></circle>
      <circle data-dc-tpl="23" cx="88" cy="80" r="8.5" data-om-id="bad56d67:28"></circle>
      <circle data-dc-tpl="24" cx="80" cy="94" r="8.5" data-om-id="bad56d67:29"></circle>
      <circle data-dc-tpl="25" cx="66" cy="92" r="8.5" data-om-id="bad56d67:30"></circle>
      <circle data-dc-tpl="26" cx="83" cy="92" r="8.5" data-om-id="bad56d67:31"></circle>
    </g>
  </svg>

  <!-- Variance -->
  <svg data-dc-tpl="34" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 220 220" role="img" aria-label="Variance: shots scattered widely around the centre" data-om-id="bad56d67:39">
    <circle data-dc-tpl="35" cx="110" cy="110" r="102" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.15" data-om-id="bad56d67:40"></circle>
    <circle data-dc-tpl="36" cx="110" cy="110" r="68" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.15" data-om-id="bad56d67:41"></circle>
    <circle data-dc-tpl="37" cx="110" cy="110" r="34" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.15" data-om-id="bad56d67:42"></circle>
    <circle data-dc-tpl="38" cx="110" cy="110" r="3.5" fill="currentColor" opacity="0.45" data-om-id="bad56d67:43"></circle>
    <g data-dc-tpl="39" fill="#e8a23c" data-om-id="bad56d67:44">
      <circle data-dc-tpl="40" cx="96" cy="56" r="8.5" data-om-id="bad56d67:45"></circle>
      <circle data-dc-tpl="41" cx="152" cy="86" r="8.5" data-om-id="bad56d67:46"></circle>
      <circle data-dc-tpl="42" cx="140" cy="152" r="8.5" data-om-id="bad56d67:47"></circle>
      <circle data-dc-tpl="43" cx="72" cy="142" r="8.5" data-om-id="bad56d67:48"></circle>
      <circle data-dc-tpl="44" cx="58" cy="100" r="8.5" data-om-id="bad56d67:49"></circle>
      <circle data-dc-tpl="45" cx="120" cy="118" r="8.5" data-om-id="bad56d67:50"></circle>
    </g>
  </svg>

  <!-- Noise -->
  <svg data-dc-tpl="53" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 220 220" role="img" aria-label="Irreducible noise: a grainy cloud of scatter that cannot be tightened">
    <circle data-dc-tpl="54" cx="110" cy="110" r="102" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.15" data-om-id="bad56d67:59"></circle>
    <circle data-dc-tpl="55" cx="110" cy="110" r="68" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.15" data-om-id="bad56d67:60"></circle>
    <circle data-dc-tpl="56" cx="110" cy="110" r="34" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.15" data-om-id="bad56d67:61"></circle>
    <circle data-dc-tpl="57" cx="110" cy="110" r="3.5" fill="currentColor" opacity="0.45" data-om-id="bad56d67:62"></circle>
    <g data-dc-tpl="58" fill="#2fa6ad" opacity="0.55" data-om-id="bad56d67:63">
      <circle data-dc-tpl="59" cx="96" cy="98" r="2.2" data-om-id="bad56d67:64"></circle>
      <circle data-dc-tpl="60" cx="124" cy="100" r="2.2" data-om-id="bad56d67:65"></circle>
      <circle data-dc-tpl="61" cx="100" cy="124" r="2.2" data-om-id="bad56d67:66"></circle>
      <circle data-dc-tpl="62" cx="122" cy="122" r="2.4" data-om-id="bad56d67:67"></circle>
      <circle data-dc-tpl="63" cx="88" cy="112" r="1.8" data-om-id="bad56d67:68"></circle>
      <circle data-dc-tpl="64" cx="132" cy="114" r="1.8" data-om-id="bad56d67:69"></circle>
      <circle data-dc-tpl="65" cx="112" cy="88" r="1.8" data-om-id="bad56d67:70"></circle>
      <circle data-dc-tpl="66" cx="110" cy="134" r="1.8" data-om-id="bad56d67:71"></circle>
      <circle data-dc-tpl="67" cx="94" cy="90" r="1.6" data-om-id="bad56d67:72"></circle>
      <circle data-dc-tpl="68" cx="128" cy="92" r="1.6" data-om-id="bad56d67:73"></circle>
      <circle data-dc-tpl="69" cx="92" cy="130" r="1.6" data-om-id="bad56d67:74"></circle>
      <circle data-dc-tpl="70" cx="130" cy="128" r="1.6" data-om-id="bad56d67:75"></circle>
      <circle data-dc-tpl="71" cx="80" cy="106" r="1.5" data-om-id="bad56d67:76"></circle>
      <circle data-dc-tpl="72" cx="140" cy="108" r="1.5" data-om-id="bad56d67:77"></circle>
      <circle data-dc-tpl="73" cx="110" cy="78" r="1.5" data-om-id="bad56d67:78"></circle>
      <circle data-dc-tpl="74" cx="108" cy="142" r="1.5" data-om-id="bad56d67:79"></circle>
      <circle data-dc-tpl="75" cx="84" cy="124" r="1.3" data-om-id="bad56d67:80"></circle>
      <circle data-dc-tpl="76" cx="136" cy="94" r="1.3" data-om-id="bad56d67:81"></circle>
    </g>
    <g data-dc-tpl="77" fill="#2fa6ad">
      <circle data-dc-tpl="78" cx="104" cy="106" r="6.5" data-om-id="bad56d67:83"></circle>
      <circle data-dc-tpl="79" cx="118" cy="108" r="6.5" data-om-id="bad56d67:84"></circle>
      <circle data-dc-tpl="80" cx="108" cy="120" r="6.5" data-om-id="bad56d67:85"></circle>
      <circle data-dc-tpl="81" cx="120" cy="119" r="6.5" data-om-id="bad56d67:86"></circle>
    </g>
  </svg>
</div>

Here's the part that matters. A complex model has low bias but high variance. A simple heuristic has higher bias but very low variance. And in a noisy, shifting, small-data world — most of our world — the _variance_ term is the one that kills you. The heuristic wins not because it captures the pattern better, but because it has almost no parameters to fit, so there's almost nothing in it for the noise to corrupt. Nothing to estimate, nothing to get wrong.

The cognitive scientists arguing about heuristics and the ML people arguing about overfitting are describing the _same decomposition_. It's [the bias–variance dilemma from statistical learning](https://en.wikipedia.org/wiki/Bias%E2%80%93variance_tradeoff)<sup>[4](#fn4)</sup>, and it says over-thinking a judgement and over-fitting a model are the same failure wearing different clothes. A fast-and-frugal heuristic is robust _because_ it's dumb.

## The honest caveat

Now I have to be honest with you, partly because there are people who'd catch me if I weren't. The strong version of _less-is-more_ is contested, and pretending otherwise would be exactly the kind of overconfidence this post is meant to warn against.

The sharpest pushback is Hilbig and Richter's [_Homo Heuristicus Outnumbered_](https://onlinelibrary.wiley.com/doi/10.1111/j.1756-8765.2010.01123.x)<sup>[5](#fn5)</sup>. Their case is that the effect is real but narrower than Gigerenzer sells it. Across many datasets, strategies that _weigh up_ several cues tend to do at least as well, and people's actual behaviour looks more like sensible cue-weighting than dramatic one-good-reason stopping. They also point out that the fight was sometimes rigged. Pit a frugal heuristic against plain multiple regression — which overfits badly on small samples — and of course the heuristic shines; you've handed it a soft opponent. Line it up against a regularised or Bayesian competitor, one that already controls its own variance, and the gap narrows.

This is the right conclusion to take away, and it's the opposite of a retreat. The honest version isn't "simple always wins." It's: **simplicity is the correct default when data is scarce and noisy, and that advantage narrows as the world hands you more signal.** Kahneman's camp and Gigerenzer's camp aren't really contradicting each other so much as describing different regimes of the same trade-off. In a stable world, with strong signal, and mountains of clean data, the complex model's low bias earns its keep, which is precisely why deep learning runs the table on images and language. Noisy, non-stationary, small sample — robustness wins. The mistake was never reaching for complexity. The mistake is assuming you're always in the first world when you're usually in the second.

## What this means on a Tuesday

Here's the payoff, and it's the part that should land for anyone who ships software for a living. You already make this bet constantly — you've just been calling it taste.

Take YAGNI (_You Aren't Gonna Need It_). When you refuse the premature abstraction, that isn't laziness and it isn't even only good manners, it's variance reduction with a proof behind it. The flexible, future-proof, handles-every-imagined-case design is a high-variance model fit to a future you are _guessing_ at. Your clever abstraction is overfitting to the three cases you've actually seen. Fewer moving parts is fewer things estimated from too little data.

The same trade-off shows up everywhere once you can name it. Architecture: fewer moving parts beats a configurable everything-engine fit to a guessed future. Hiring: one strong, legible signal often generalises better than an elaborate twelve-point rubric scored on a handful of interviews. Estimation: a simple reference class beats a detailed bottom-up model built from numbers you mostly invented. Every time, it's the same bet, and now you can make it on purpose:

- **Judge on out-of-sample performance, not training fit.** The student who aced the practice paper tells you nothing. Hold data back, or watch the real production numbers, and trust those.
- **Treat every feature, parameter, and abstraction as a cost.** Each one buys a sliver of bias reduction and a dose of variance. There's no free lunch — you're trading, not adding.
- **Ask how much signal there actually is.** High signal-to-noise and abundant data justify complexity. Thin, noisy, drifting data punishes it. Be honest about which one you're holding.
- **Lean on your observability.** You absolutely should be watching the live metric, because the moment a clever model starts overfitting to a passing fad, that's where you'll see it before your users do.

## Conclusion

The napkin rule isn't a law of nature, and I'm not asking you to junk your models and equal-weight everything in sight. The point is narrower and, I think, more useful: simplicity is not the consolation prize. A heuristic that ignores information can beat a model that uses all of it, for the same hard mathematical reason a regularised model generalises better than an overfit one — it has less to get wrong.

So the question I'd leave you with isn't "should this be simple, or complex?" It's "how much has the world actually given me?" Sparse, noisy, a handful of examples — bet simple. Abundant, stable, well-understood — then, and only then, you've earned the right to be clever.

Build the dumb baseline first and make the sophisticated model out-argue it on data it has never seen. Often enough, it can't; and that's not a failure of ambition, it's the easy win, and I'd take it.

---

<sup id="fn1">1. Gigerenzer, G. & Brighton, H. (2009). _Homo Heuristicus: Why Biased Minds Make Better Inferences._ Topics in Cognitive Science, 1(1), 107–143.</sup>

<sup id="fn2">2. On the tennis result, see Serwe, S. & Frings, C. (2006), _Who will win Wimbledon?_; on the recognition heuristic generally, Goldstein, D. G. & Gigerenzer, G. (2002), _Models of ecological rationality: The recognition heuristic._ Psychological Review, 109(1).</sup>

<sup id="fn3">3. DeMiguel, V., Garlappi, L. & Uppal, R. (2009). _Optimal Versus Naive Diversification: How Inefficient Is the 1/N Portfolio Strategy?_ Review of Financial Studies, 22(5), 1915–1953.</sup>

<sup id="fn4">4. Geman, S., Bienenstock, E. & Doursat, R. (1992). _Neural Networks and the Bias/Variance Dilemma._ Neural Computation, 4(1), 1–58.</sup>

<sup id="fn5">5. Hilbig, B. E. & Richter, T. (2011). _Homo Heuristicus Outnumbered: Comment on Gigerenzer and Brighton._ Topics in Cognitive Science, 3(1), 187–196.</sup>
