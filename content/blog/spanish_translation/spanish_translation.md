---
title: "Writing an OpenAI powered app to practice my Spanish skills"
date: 2024-12-28
tags: [OpenAI]
---

I have been learning Spanish for a year now. Like many others, I find that while I am quite good at reading, my active writing and speaking skills remain fairly basic. To address this gap, I developed a small application that generates random sentences for me to translate into Spanish. With the help of OpenAI, I can translate these sentences and receive corrections, making the process both interactive and effective.

# Concept

An app isn't strictly necessary for this purpose—anyone can use ChatGPT to generate sentences and then ask it to correct their translations. However, I wanted to streamline this process for a few key reasons:

- It’s more useful to practice translations that we've already done, rather than always translating new sentences. With long-term use, this requirement becomes difficult to fulfill with solely relying on ChatGPT.
- Interaction with ChatGPT typically happens through typed requests. When it corrects a translation, it often provides explanations in its response. However, when everything is included in a single response, it can become overwhelming. With a custom UI, we can hide the explanations and reveal them by clicking a button.
- I’ve found that OpenAI’s random sentence generation isn’t entirely random. After a few iterations, it tends to recycle the same topics and phrases. I’ve found it more effective to randomize several parameters in code and feed them into the OpenAI prompt.

# Setup

I wrote the app using TypeScript and ReactJS, and it’s deployed on fly.io. All translations are stored in a SQLite database, along with the number of times I’ve provided a correct translation. This helps during practice by adjusting the probability of seeing a sentence again. This was my first time using fly.io, and I’m quite satisfied with it. I needed a simple, cost-effective platform to deploy my code, and it fits this need perfectly.

# Challenges

## 1. Flaky corrections

Initially, I asked OpenAI to correct the sentence and provide a list of explanations. This worked in most cases, but there was one annoying issue:

1. I translate a sentence.
2. It corrects my translation and provides an explanation of why it’s incorrect.
3. Later, when translating the same sentence, I provide the corrected translation it gave me earlier.
4. It corrects it again, usually switching some words with synonyms and claiming that they fit the context better.

After struggling with this issue for a while, I tried refining the prompt by adding the following additional commands:
- "Focus only on major errors that affect meaning, and ignore minor stylistic differences or words that only have a slightly different meaning."
- "Be relaxed and non-critical in evaluating the Spanish translation."

However, it seemed to ignore many of these adjustments and still had a tendency to make corrections, even when the issue was simply with synonyms.

It took me some time to realize that the process works much better if I split it into two tasks:

1. One prompt to provide the correction and a list of explanations.
2. Another prompt to ask a simple question: Is the translation correct at all?

If the answer to "Is the translation correct?" is yes, there’s no need to show the output from the other prompt with more detailed corrections.

Since then, I’ve read the prompt engineering guide from OpenAI and discovered that they offer similar advice [here](https://platform.openai.com/docs/guides/prompt-engineering#strategy-split-complex-tasks-into-simpler-subtasks).

## 2. Random sentences

I went through several iterations when it came to generating random sentences for translation practice. Initially, the prompt was quite simple: "Generate a random beginner-level sentence for translation." However, the responses were often very similar, mostly revolving around students and their homework, and lacking some verb tenses I wanted to practice. Eventually, I decided to parameterize the prompt and generate random values for the following parameters using my code:

- Mood: indicative, question, imperative, etc.
- Tense: past, present, future
- Word: a word that must be included in the sentence. I ended up using a list of "The 1000 most common English words" and randomly selecting one each time.
- Topic: science, politics, religion, etc.




