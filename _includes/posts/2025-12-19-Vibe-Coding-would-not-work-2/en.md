On a whim, I decided to try using the command-line interface of an `AI` for file-related operations, which is more convenient than web chat. I started with OpenAI's `Codex CLI`, but it's not available in free version, so I switched to the `Google Gemini CLI`.

After installation, I opened the command line in the directory containing my articles and entered `Gemini CLI`. I pointed the `AI` to my previously written article [Vibe Coding would not work](https://eagleboost.com/2025/08/16/Vibe-Coding-would-not-work/) and asked if there were any logical fallacies in the article.

`Gemini` quickly concluded that the author of the article did not attack the core of `Karpathy`'s viewpoint but instead attacked a simplified and distorted straw man version that was easier to refute.

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Vibe%20Coding%20Straw%20Man%20Fallacy1.jpg)

I decided to argue with it, asking, "But if a person doesn't know how to program, how can they read code and guide the AI to modify it?" After a more detailed explanation, the `AI` finally understood and admitted that its analysis regarding the "straw man" was not comprehensive enough.

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Vibe%20Coding%20Straw%20Man%20Fallacy2.jpg)
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Vibe%20Coding%20Straw%20Man%20Fallacy3.jpg)

It's evident that the `AI`'s comprehension is still quite mechanical. The meaning I ultimately conveyed in my input to the `AI` was actually present in the original article, but judging from the earlier conversation, the `AI` had either overlooked or downplayed it. I suspect that since its initial task was "find any logical fallacies in the article," its focus shifted towards the "logical fallacies" aspect, causing it to neglect the article's original intent: to deconstruct the concept of `Vibe Coding` and argue that it is misaligned for its two potential user groups.

This also illustrates, from one perspective, the importance of prompts and why `AI` doesn't truly think. If a human were to read such a short essay, even with a specific task, they would be unlikely to ignore its central thesis.

So, are there any logical fallacies in the article? At least not the straw man fallacy. Let's demonstrate this. First, ask two questions:

+ Does the true `Vibe Coding` actually exist?
+ Does `Vibe Coding` have any meaning?

The answer is yes. A real-world example: a client with no technical knowledge tells a contractor what kind of software to develop. However, for this model to work, several preconditions are necessary:

+ The contractor has experience.
+ The contractor has the ability to identify and communicate ambiguous requirements with the client.
+ The contractor has the ability to fix errors (which is inherently the contractor's responsibility).

For current `AI`, the first point is arguably met, given the vast amount of data it's been pre-trained on—its theoretical experience surpasses that of all humans combined. The second point is questionable. Probability-based `AI` cannot understand the real world through interaction, nor can it truly comprehend human needs. Furthermore, for various reasons, it cannot repeatedly ask humans for clarification, as that would make the `AI` vendor seem inadequate.

The third point pertains to the coding process, which usually occurs after the second point is clear. If there are issues with the second point, the third becomes a castle in the air. `AI` does not genuinely understand the code it generates, so its inability to fix errors is common, especially for code generated without clear constraints.

In reality, programmers often complain that clients themselves don't know what they want. This requires the contractor to have an experienced project manager to lead the entire project, serving as a bridge between the client and the programmers. Thus, the actual subjects of `Vibe Coding` become the project manager, who deeply understands the client's needs, and the programmers. Although it's a bit of a stretch, one could argue that the client and the project manager together direct the programmers in `Vibe Coding`.

`Karpathy` likely derived the concept of `Vibe Coding` from the real world, but the chasm between its ideal and reality is quite evident: one subject in his envisioned `Vibe Coding` is a human who doesn't know programming but provides requirements—essentially a client that programmers cannot communicate with; the other subject is an `AI` with abundant theoretical experience but no understanding of the real world and prone to severe hallucinations. If even human programmers struggle to communicate with such clients, how can a non-human `AI`?

In the real world, an excellent project manager is key to a project's success, and their importance cannot be overstated. However, this role is absent in the `Vibe Coding` concept. This is why it's possible for someone completely non-technical to direct an `AI` to create "toy projects" with entirely defined requirements like Snake or Tetris, but it's utterly unfeasible for real projects.

For a considerable time to come, the only model likely to yield genuine productivity is probably this: an experienced project manager (or architect) who understands both technology and business communicates with the client to understand requirements, breaks them down into relatively independent small tasks, directs the `AI` to complete these tasks, and can intervene manually when necessary. The feasibility of this approach is unquestionable, so there's no need to invent a term like `Vibe Coding` to define it specifically.

If we must use the term `Vibe Coding`, this is probably its only accurate definition.

On a side note, the power of `AI` is one reason why recent graduates are finding it difficult to get jobs: their business communication skills can't match those of business analysts, their technical knowledge can't match that of architects, and their grunt work capability can't match that of `AI`. Thus, inexperienced newcomers unwittingly become a liability. Companies will favor mid-career professionals who understand both technology and business—precisely those over `35` whom internet memes say get laid off. The advantage of young people having good health and being able to work overtime is worthless compared to tireless `AI`. Who capitalists will choose is self-evident.

How young people can navigate this situation in the `AI` era is a question worth pondering deeply.