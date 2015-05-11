---
layout: default
title: "Carnac the Magnificent"
---

Programmers love to discuss interviewing programmers. And hate to discuss it. Interviewing touches the very heart of human social interaction: It's a process for picking "winners" and "losers," for determining who's "in" and who's "out."

Today I'd like to discuss an anti-pattern, **Carnac the Magnificent**.

> Carnac the Magnificent was a recurring comedic role played by Johnny Carson on The Tonight Show Starring Johnny Carson. One of Carson's most well known characters, Carnac was a "mystic from the East" who could psychically "divine" unknown answers to unseen questions.--[wikipedia](https://en.wikipedia.org/wiki/Carnac_the_Magnificent)

The "Carnac the Magnificent" anti-pattern is setting up a situation where the only way to pass is to guess what the interviewer is looking for. Here's an example from a blog post that is currently [causing tongues to wag][wag]:

[wag]: http://www.reddit.com/r/programming/comments/358tnp/five_programming_problems_every_software_engineer/

> Write a program that outputs all possibilities to put `+` or `-` or nothing between the numbers `1`, `2`, ..., `9` (in this order) such that the result is always `100`. For example: `1 + 2 + 34 – 5 + 67 – 8 + 9 = 100`.

According to [TFA], this question is a "filter," designed to separate those who have no hope of being a programmer from those who have the basic qualifications to write software for a living.

[TFA]: http://www.urbandictionary.com/define.php?term=TFA

### the problem with the problem

So what is the problem? Well, the problem is that **there are too many ways to solve the problem**.

For starters, you can generate all of the possible strings (e.g. `123456789`, `12345678-9`, `12345678+9`, `1234567-89`, `1234567-8-9`, `1234567-8+9`, `1234567+89`, `1234567+8-9`, `1234567+8+9`, ...), then use `eval` to compute the answer, and select those that evaluate to `100`.

Or you could do the same thing, but avoid `eval` and bake in a little of your own computation. Because `eval` is "bad."

And of course, this brute force executes fewer than 10,000 iterations, and runs faster than you can blink on contemporary hardware. But you're applying for a job where you're supposed to know about "scale" and "speed," so you could optimize things and not do obviously wasted computations. Nothing that starts with `12345` can ever add up to `100`, for example. Aren't programmers supposed to know this?

And should you solve this recursively or iteratively? One is fast, the other reveals the underlying mathematical symmetry of the problem.

### no hire!

There are a bunch of ways forward (many more than these four considerations, in fact).

And you can easily imagine a sadistic interviewer failing a candidate for getting the correct answer the wrong way. If you use `eval`, you're a bozo. And if you write your way around `eval`, you're a "theorist" who doesn't know when to use the right tool for the job. If you don't optimize, you don't value scale. And if you do optimize, you're wasting time that could be better used for another part of the interview.

And if you solve it without recursion, you don't grasp elegance. And if you do solve it with recursion, sorry, but we use JavaScript here, Lisp jobs are down the hall.

Here's the most naïve code I can think of:

{% highlight javascript %}
for (let o1 of ["", "+", "-"]) {
  for (let o2 of ["", "+", "-"]) {
    for (let o3 of ["", "+", "-"]) {
      for (let o4 of ["", "+", "-"]) {
        for (let o5 of ["", "+", "-"]) {
          for (let o6 of ["", "+", "-"]) {
            for (let o7 of ["", "+", "-"]) {
              for (let o8 of ["", "+", "-"]) {
                const expr = `1${o1}2${o2}3${o3}4${o4}5${o5}6${o6}7${o7}8${o8}9`;
                const value = eval(expr);
                if (value === 100) {
                  console.log(expr)
                }
              }
            }
          }
        }
      }
    }
  }
}
{% endhighlight %}

([es6fiddle](http://www.es6fiddle.net/i9fp5ur2/))

Every single thing you can say negatively about this solution represents an unstated requirement.

Likewise, here's a recursive solution:

{% highlight javascript %}
function solutions (accumulatedOutput, runningTotal, ...numbers) {
  if (numbers.length === 0) {
    if (runningTotal == 100) console.log(accumulatedOutput);
  }
  else {
    const [first, ...butFirst] = numbers;

    if (accumulatedOutput !== "") {

      // case one, addition
      solutions(`${accumulatedOutput}+${first}`, runningTotal + first, ...butFirst);

      // case two, subtraction
      solutions(`${accumulatedOutput}-${first}`, runningTotal - first, ...butFirst);
  
    }
    else solutions(`${first}`, first, ...butFirst);

    // case three, catenation
    if (butFirst.length > 0) {
      const [second, ...butSecond] = butFirst;
  
      solutions(accumulatedOutput, runningTotal, first * 10 + second, ...butSecond);
    }
  }
}

solutions("", 0, 1, 2, 3, 4, 5, 6, 7, 8, 9);
{% endhighlight %}

([es6fiddle](http://www.es6fiddle.net/i9h7385g/))

It's faster, and more beautiful mathematically, but it's actually harder to understand how it works than the iterative solution. And it took me a lot longer to write. As did this one, based on Generators and Iterators:

{% highlight javascript %}
// Utility function from https://leanpub.com/javascriptallongesix

const filterIterableWith = (fn, iterable) =>
  ({
    [Symbol.iterator]: function* () {
      for (let element of iterable) {
        if (!!fn(element)) yield element;
      }
    }
  });
  
// problem-specific functions
  
const catenate = (left, right) =>
  left * Math.pow(10, Math.ceil(Math.log(right) / Math.LN10)) + right;

function * expressions ([first, ...rest]) {
  if (rest.length === 0) {
    yield [first];
  }
  else {
    for (let restBlanks of expressions(rest)) {
      const [firstOfRest, ...restOfRest] = restBlanks;
      
      yield [first, '+', ...restBlanks];
      yield [first, '-', ...restBlanks];
      yield [catenate(first, firstOfRest), ...restOfRest];
    }
  }
}

function calculate([first, operation, ...rest]) {
  if (rest.length === 0) {
    return first
  }
  else if (operation === '+') {
    return first + calculate(rest);
  }
  else if (operation === '-') {
    return first - calculate(rest);
  }
}

const is100 = (expr) =>
  calculate(expr) === 100

const solutions = filterIterableWith(is100,
    expressions([1, 2, 3, 4, 5, 6, 7, 8, 9]));
{% endhighlight %}

([es6fiddle](http://www.es6fiddle.net/i9hhvyp0/))

Again, this highlights a certain way of thinking about the problem, that of viewing it as a pipeline of operations on iterable collections: Recursively generate all the expressions, and filter for those that sum to `100`.

Beyond proving that a candidate knows how to write things recursively, or with iterators, or both... Why are either of these better? When are they better? For which interviewers are these better?

We don't know from the problem as stated.

So maybe what you should do is ask the interviewer about the hidden requirements. Optimize for speed above all else? Write tests or not? Is shorter code better? Should the code be factored neatly and all repetition DRY'd out?

That seems reasonable, diving requirements is part of a developer's job. And some interviewers will rate you highly for that. But others will consider it wasting time when all they wanted as a working answer, any answer, you are obviously tedious and slow and can't GetShitDone™.

The bottom line is, there is no right thing to do given a problem where the interviewer does not make it very, very clear what they want. The only person who can get this right is Carnac the Magnificent, a mystic from the east who can read minds and the contents of sealed envelopes.

### stress

Now let's be honest: If the interviewer and the interviewee are on the same page, this doesn't seem bad. But the fact is, the interviewee will be stressed 100% of the time. And that is not good for the interviewee or for the interviewing process.

There is good stress and bad stress, and uncertainty about what the interviewer wants is bad stress. You aren't testing whether the candidate can solve hard problems, you're testing whether the candidate can write code just before an all-hands where the CEO will announce layoffs.

### the right way forward

If all you want is working code, say so, preferably in writing so that all candidates get the same information:

> Write a program that outputs all possibilities to put `+` or `-` or nothing between the numbers `1`, `2`, ..., `9` (in this order) such that the result is always `100`. For example: `1 + 2 + 34 – 5 + 67 – 8 + 9 = 100`. Use any technique you want, the only thing that matters is getting the correct answer.

Or be up front that you want production-ish code:

> Write a program that outputs all possibilities to put `+` or `-` or nothing between the numbers `1`, `2`, ..., `9` (in this order) such that the result is always `100`. For example: `1 + 2 + 34 – 5 + 67 – 8 + 9 = 100`. Although this is a toy problem, solve it using the kind of code you're use in a production code base.

Or encourage the candidate to ask questions:

> Write a program that outputs all possibilities to put `+` or `-` or nothing between the numbers `1`, `2`, ..., `9` (in this order) such that the result is always `100`. For example: `1 + 2 + 34 – 5 + 67 – 8 + 9 = 100`. Feel free to ask questions if you need clarification on what is required.

### simple, right?

That's simple, right? So why do people do this? My theory is *They don't know they are asking a Carnac the Magnificent problem*. Interviewers often have a huge blind spot about how it feels to be on the other side of the table.

Perhaps they were hired with this exact same question, they wrote out an answer without asking any questions, they were hired, what's the problem? Or they asked a few questions, they weren't failed for wasting time, why do some candidates charge ahead without asking questions?

You need to have experience with interviews and experience working with a variety of programmers to appreciate how a "simple" technical question might actually have hidden pitfalls. Sometimes that's a pitfall in itself! Imagine a 22 year-old extremely smart programmer interviewing a 52 year-old veteran. Why is the veteran thinking so hard about this problem? Their brain must have fossilized. NO HIRE.

<iframe width="420" height="315" src="https://www.youtube.com/embed/76wzA2A2T1Q" frameborder="0" allowfullscreen></iframe><br/>

Let me be brutally frank:

**Part of the job of being a software developer is to understand the ways in which things that appear simple—like code, user experiences, security protocols, and almost everything else we touch—are not actually simple**. And it is our job to either make them simple, or make it very clear that they aren't that simple and document the way in which they aren't simple.

If you're interviewing candidates, it's your responsibility to figure out that the interviewing process, including the questions you ask, isn't as simple as it appears, and then to **make it simple**. Ask hard questions, fine, but don't make the process hard.

### tl;dr

Hidden possible requirements add stress to the interviewing process, and it's bad stress, not good stress. And it causes the most stress for the candidates with the most experience. So you get bad results.

It's so easy to get good results from questions: Be clear about what you want from the candidate.

So if you are interviewing programmers, here's your homework: Go through all of your interview questions, whether technical or otherwise, and ask yourself if there are any hidden assumptions about what you expect. Then ask yourself if your process would be even better if you made those implicit requirements explicit.

I think you'll find that eliminating Carnac the Magnificent from your interviewing process will make it better.

(discuss on [reddit](http://www.reddit.com/r/programming/comments/35afm0/carnac_the_magnificent/) and [hacker news](https://news.ycombinator.com/item?id=9511700))

---

*post scriptum*:

Note: I'm not saying that the author if TFA poses the question exactly as worded to real candidates without any further discussion. TFA is a blog post explaining the problem to fellow interviewers, not going into all the details of how to set up the problem, whether to encourage conversation or give hints, and so forth.

However, I *am* saying that I have seen similar problems posed as filters, without any further exposition about what is wanted, and I have definitely encountered interviewers who have hidden expectations, interviewers who are impatient if asked too many questions, interviewers who are judging candidates by the speed of writing a solution and consider questions to take up valuable time, and so forth.

TFA was simply an excuse to discuss something I have observed many, many times over the past couple of decades.
