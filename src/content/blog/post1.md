---
title: "What I learned by creating my first SaaS"
description: "A reflection on the key lessons and insights gained from building MonsterLabs, my first SaaS project"
pubDate: "Dec 21 2024"
heroImage: "/monsterlabs.webp"
tags: ["SaaS", "Product Development"]
---

One of things I wanted to do in 2024 was to build a software product that actually made money. I've been working on MonsterLabs for a few months now and I'm proud to say that MonsterLabs is now live and serving real customers!

Monsterlabs is an AI-powered platform for generating custom Dungeons & Dragons monsters and magic items. It's a project that allowed me to blend my love for gaming with my passion for development, and along the way, I picked up invaluable lessons and skills. Here's a look at what I learned while turning this idea into reality.

## OpenAI and structured output
The most important part of MonsterLabs is the AI generation. I wanted to make sure that the AI was able to generate the best possible results, so I spent a lot of time researching how to structure the data in a way that would be easy for the AI to understand. I've found that the best way to do this is to use either function calling or structured output. Strucuted outputs weren't out yet when I started this project, so I had to use function calling. Function calling is a bit of a misnomer, as the AI model isn't actually calling a function, but rather returning a JSON object which follows a specific schema, which you then could use to call a function. 

It's important to get your data structure correct, consistent and easy for the AI to understand. Zod is a great library for this. Here's a snippet of what I'm talking about:

``` typescript
import { z } from 'zod';

export const itemSchema = z.object({
  name: z.string(),
  type: z.union([z.literal('Weapon'), z.literal('Armor'), z.literal('Ammunition'), z.literal('Potion'), z.literal('Scroll'), z.literal('Ring'), z.literal('Wand'), z.literal('Rod'), z.literal('Staff'), z.literal('Wondrous item'), z.literal('Consumable'), z.literal('Tool'), z.literal('Trinket')]),
  subtype: z.string().nullable().describe("Subtype of the item, if applicable. Examples include 'longsword', 'dagger', or 'plate', 'chain' etc. Leave blank if not applicable."),
  rarity: z.union([z.literal('common'), z.literal('uncommon'), z.literal('rare'), z.literal('very rare'), z.literal('legendary')]),
  requiresAttunement: z.boolean().describe("Whether the item requires attunement, and if so, whether it requires attunement by a specific class. Only applies to magical items that are very special and powerful. It should make sense for lore reasons. Generally avoid requiring attunement unless it makes a lot of sense."),
  requiresAttunementSpecific: z.string()
  .describe("If the item requires attunement, specify what conditions someone must have in to attune to it. Do not include if the item does not require attunement. Always structure your sentence as 'requires attunement by ...'. For example, 'requires attunement by a wizard' or 'requires attunement by a creature of good alignment' or 'requires attunement by an elf, half-elf, or a ranger'.")
  .nullable()
  .optional(),
  cost: z.number().describe("Cost in gold pieces"),
  weight: z.number().describe("Weight in pounds"),
  description: z.string().describe("A detailed and inspired description of the item. This should include its visual description, lore, history, and any other relevant information."),
  paragraphs: z.array(z.object({
    title: z.string(),
    content: z.string()
  })).describe("Description of the item. If the item can do something, explain how it works here. For example if it needs an action or bonus action to activate, or if it has charges, when they recharge etc."),
});
```

As you can see, the schema is quite detailed so that the AI can generate the best possible results. Using unions and literals is a great technique to ensure that you can really narrow down the options for the AI. Also, using the `describe` method is a great way to help the AI understand the data structure.

## NextJS optimizations

I've also learned a lot about NextJS optimizations. NextJS has the ability cache API responses and use ISR (Incremental Static Regeneration) to speed up the page load times. An example of where this is applied is in any page where you're looking at a single monster or magic item. The first time that page is loaded, the API response is cached and on subsequent loads, the cached response is used. This means that we can skip the API call and just render the page almost instantly. Try it out for yourself by looking at the page for a single monster or magic item [here](https://monsterlabs.app/creature/view/18).

## Data fetching optimizations
Since all of data for the site is stored in postgres, we need to make sure that we're fetching the data in the most efficient way possible. One easy trick is to run multiple database queries concurrently using Promise.allSettled() rather than waiting for them to complete one after another. Here's an example of how this can be done:

``` typescript
// Both of these queries will run concurrently
const [monsters, items] = await Promise.allSettled([
  db.query('SELECT * FROM monsters'),
  db.query('SELECT * FROM items')
]);

// The items query will run after the monsters query has completed
const monsters = await db.query('SELECT * FROM monsters');
const items = await db.query('SELECT * FROM items');
```

## Marketing
I mainly used Reddit to get the word out about MonsterLabs. I created posts about them on various DnD subreddits (The ones that allowed it) and made a post on [Product Hunt](https://www.producthunt.com/products/monster-labs). I also got a few friends to test the site and give me feedback. This is one area that I still have to improve on, but for a first public release, I'm happy with the results. Especially considering how niche the product is, and how much time I've spent on it.

## First real customers
This may not sound like a big deal, but getting your first real customers is a huge milestone. It's a great feeling to know that people are actually using your product and finding value in it. It's also a great feeling to know that you're actually making money from your work. Granted as of the time of writing this, I have 2 customers on the monthly plan, but the fact that they're paying me money for something I've created is a great feeling.
![Net income from Stripe](../../../public/stripe-moneys.webp)

## Conclusion

I've learned a lot by creating MonsterLabs and I'm proud of what I've achieved. This post is a very small section of what I've learned, otherwise this blog post would be a book. I'm looking forward to continuing to work on MonsterLabs and see where it takes me.
