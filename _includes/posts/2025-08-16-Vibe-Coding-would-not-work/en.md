I watched the interview of Sam Altman on *Huge Conversations* following the release of **GPT-5**.

Shortly after beginning of the interview, he mentioned:  

> The thing that I am most excited about is this is a model **for the first time where I feel like I can ask kind of any hard scientific or technical question and get a pretty good answer**.
> And I’ll give a fun example: actually when I was in junior high, or maybe it was ninth grade, I got a TI-83, this old graphing calculator. And I spent so long making this game called **Snake**. It was a very popular game with kids in my school. And I was proud when it was done. But programming on a TI-83 was extremely painful. It took a long time. It was really hard to debug and whatever.


As the most powerful AI model ever created, **GPT-5**, it’s surprising that Sam Altman used something **GPT-3.5** could already do years ago to demonstrate its capabilities—Snake is neither a complex scientific nor technical problem. Even for the sake of making it understandable to the general public, there must be better examples to cite.  

One of OpenAI’s co-founders, **Andrej Karpathy**, earlier proposed the concept of **"Vibe Coding"**:  
> *"You fully give in to the vibes... forget the code even exists."*  

Undoubtedly, Sam Altman didn’t write a single line of code—he just gave some instruction, and **GPT-5** produced a Snake game. Is this **Vibe Coding**? I don’t think so. Models like GPT have already been trained on codes like Snake game. Generating it in one go, without even needing to specify requirements, hardly counts as *coding*.  

Whether **Vibe Coding** is useful is another question, but by definition, it should at least follow this workflow:  
> Human describes requirements → AI generates code → Human tests → Human provides feedback → AI iterates and optimizes

So, is **Vibe Coding** useful? Or is it even feasible? My answer is:  
> **For non-professionals creating simple prototypes, it’s somewhat feasible—and that’s it.**  

A simple test would be enough to verify my opinion.  

I used a interview question I designed to assess candidates’ knowledge—**implementing a basic user management window**. The AI assistant I used was **Claude 4 Sonnet**, widely regarded as the strongest model for coding.  

### **Round 1: High-Level Requirements**  
I provided a very general description of the task. Since this is an extremely common scenario, the AI quickly generated a **mediocre implementation**—better than most average interviewees, but the code quality was nothing impressive. Due to the vague requirements, corners were cut (e.g., input validation was ignored).  

The reason is simple: **If humans don’t ask the right questions, the AI won’t give thoughtful answers.** In software development, users rarely mention details like input validation or real-time feedback when describing requirements. Experienced business analysts can translate user needs into actionable documentation, but they often also skip such details—those only come into developers mind during actual coding.

So, **what role does the human play in Vibe Coding?** Non-experts can’t foresee details, so their requirements are vague, leading to mediocre outputs. **Higher-quality results demand higher-quality input, which requires expertise.**  

### **Round 2: Detailed Requirements (500+ words)**  
This time, I give AI details about 500 word long that covers below:  
- UI styling  
- Data formats  
- Input validation  
- Real-time feedback  
- ...and more  

The AI’s output improved significantly—it used **polymorphism for different modes** and **data templates for the view layer**.  

But when running it:  
1. Input validation had issues. After feedback, it improved.  
2. Error messages didn’t auto-dismiss. Multiple feedback attempts failed—Claude even started **hallucinating**. I had to take over and point out the exact problem so it could fix it.  

The conclusion is clear:  
1. **Non-experts (and even junior devs) struggle to articulate requirements from a development perspective.**  
2. **The AI doesn’t truly *understand* the code it generates (note: *generates*, not *writes*).** Sometimes it seems to fix issues, but it’s just swapping solutions based on other training data. In my case, this didn’t work—if I weren’t a programmer, I and the AI would be stuck in a headless situation.  

From a professional standpoint, while Round 2’s code was better, it was still **far from production-ready**. This touches on another issue: **training quality**.  

AI coding models are trained on **publicly available internet code**, but let’s be honest—most online code is **low-quality**. Some excellent code exists but is niche, carrying little weight in the AI’s "brain." Unless explicitly prompted, the AI won’t generate it. And **many high-quality private codebases are entirely absent from AI's training.**  

Thus:  
- **Non-experts** can only guide AI to produce mediocre results.  
- **Programmers** can get better code from AI, but when problems arise, they can’t rely solely on verbal instructions to fix them.  
- **For experts, Vibe Coding is redundant**—AI does save some repetitive work, but these professionals already automate such tasks themselves. AI is smarter, but its absence wouldn’t hinder those people's efficiency. Besides, **AI-generated code often requires extra polishing to meet high standards, which can be a waste of time.**  

To me, **Vibe Coding is a paradox**:  
- **Non-experts can’t do much with it.**  
- **Experts don’t need it.**  

It feels more like a marketing concept to promote AI. As for those hyping it up, I doubt many have done any real testing.  

**Serious programmers shouldn’t fear job loss** (if a role can be fully replaced by AI, it probably wasn’t essential to begin with). Instead:  
1. **Keep improving your skills.**  
2. **Understand AI’s strengths and limits.**  
3. **Use it as the best tool ever created—but don’t overestimate it.**