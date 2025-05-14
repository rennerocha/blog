---
title: "Things I learned from translating free software projects"
date: 2025-03-31
tags: ["free software", "translation", "i18n"]
slug: things-i-learned-from-translating-free-software-projects
params:
    mastodon_id: 114502010539217500
---

Everyone who wants to start contributing to free software projects struggles to know how to start. There are different paths you can follow to get involved in a community. Smaller projects are easier to start with, but some will require technical knowledge that you don't have yet. Other projects may be more welcoming to new contributors but will require you to learn how the community works and its culture.

As a non-native English speaker, I found that one way to contribute is by translating applications **I use** into Brazilian Portuguese.

The first project I got involved with in translation was [Gancio](https://gancio.org/), a shared agenda for local communities. It had several translations, but the Portuguese one wasn't good — too many untranslated strings, and the translated ones weren't consistent. As I was installing it to be used by the community of my [hackerspace](https://lhc.net.br/) in Brazil, having it in Portuguese was very important to help people who don't know English use it entirely.

Some months later, I installed [pretix](https://pretix.eu/about/en/), a ticketing software I was going to use for a conference I was organizing. This project was in very bad shape, with only 18% translated into Portuguese, and the existing translations were terrible (with some very serious mistakes). Now, it is 95% translated, with almost everything reviewed.

After working on these projects, I learned a few things that I believe are important to mention if you plan to start following this path to contribute to free software.

This is not a full list of recommendations, techniques, and processes for translating free software. The two projects I worked on didn't have a group of people dedicated to translating them into Portuguese, but for bigger projects (like Python documentation, for example), we will usually find a more organized group of people with more well-defined processes, communication channels, and discussions to decide which is the best translation to use.

##### You will learn the business of the software

When you start translating software, you probably know a bit about its business concepts. But during the process, you will come across sentences and words that will lead you to discover other features you probably didn't know existed or didn't realize were so complete.

After working on these projects, I felt much more confident to start contributing with code (and I did for both projects). So, the translation task allowed me to contribute code as well.

##### Be aware of regional differences

Brazilian Portuguese is different from Portugal (or European) Portuguese. Even though we can understand both quite easily, software translated into one variant will look 'weird' to a native speaker of the other. This is also true for other languages, so you need to consider this when translating.

When I was working on Gancio, there was only `Portuguese` as a language choice. One day, I noticed that another person started to contribute and re-translate lots of words and expressions from the Brazilian variant to the Portugal one. **Don't do that!** You will be throwing someone else's work into the trash.

I didn't want to start a language war, so I asked the project's maintainer to create two separate versions: `pt-br` (for Brazilian Portuguese) and `pt-pt` (for Portugal Portuguese), so we could continue translating using the terms that make more sense to us. This is probably true for other languages (Spanish came to mind).

##### Create a glossary

Keep your translation consistent. Even if there are many different words for a term, avoid using synonyms everywhere. When you find a term that is used in many places, add it to a glossary so other translators will know that they should use one form rather than another

For example, in Portuguese, the verb `to delete` can be translated as `apagar`, `excluir`, or `deletar`. All of them are valid translations, but it is preferable to stick with one. This will make it easier for new translators to decide which expression/word to use, and for final users, the UI will look much better and be easier to use.

##### Know your own language

It is acceptable to make some mistakes during your translation. Usually, we are not language experts and/or professional translators. But we need to be very careful not to commit gross mistakes. Correct spelling and verb conjugations are essential. If you are not sure about how to spell a word, check it in a dictionary beforehand. Even if you have 1% doubt about whether that is the correct spelling, validate it.

##### Short words in English may not be short in other languages

Some words in English that are short can be longer in other languages. For example, `Next` is `Próximo` in Portuguese (3 extra characters). Sometimes this is not relevant, but some UIs have strict layout limits (e.g., the width of a button may be fixed), and it will not look good with longer translations. When you find something like this, checking how the translation will look is important.

Sometimes you will be able to choose a synonym in your language that keeps the same meaning; other times, you may suggest changes in the project code so the UI is not broken.

This [issue in pretix](https://github.com/pretix/pretix/issues/4796) is a great example of this.

##### Inclusive language is important

Inclusive language is a way of speaking that aims to avoid expressions that may be sexist, racist, or biased against certain groups of people. I noticed during the translations that sometimes it is difficult to translate some neutral terms in English into Portuguese.

Many nouns in English are not gender-specific, so if you say `the reviewer`, you are not specifying the gender of the person reviewing a proposal. But in Portuguese, this noun can be translated as `o revisor` (male) or `a revisora` (female). `All` can be `todos` (male) or `todas` (female). So, you can see how hard this can be.

The common 'default' is to use the male variant, but I tried my best to avoid it. There are some constructions that people use (in Portuguese, but I know that every language has constructions like this) to remove that bias. For example, instead of `todos` and `todas`, some people use `todes` (the `e` is added instead of using `o` and `a`).

This is not a common way to write Portuguese, and I personally don't like it (but it is not wrong if you decide to use it). So, some possibilities can be:

- Remove the article before the noun. For example, when translating `the speaker`, instead of using `o palestrante` or `a palestrante`, use only `palestrante`.
- Choose a different word that has the same meaning.
- Write the sentence in a different way, keeping the meaning but avoiding gender markers.

It is not an easy task, but it is important to worry about that. Remember that the software you are translating will be used by people worldwide. **Be respectful to them!**

##### You may procrastinate a lot doing this

Some days, I was tired and didn't want to do anything else. But I felt guilty for not doing anything, so I spent a significant amount of time translating the projects. I started to consider this a form of procrastination (at least the result of my procrastination was something meaningful), but — at least for me — it was important to set some boundaries and limit my time spent doing translations. I limited myself to 30 minutes every business day. It worked well for me.