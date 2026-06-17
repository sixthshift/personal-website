---
title: "Building a Personal MCP Server"
description: "Built-in AI personalization is just a bag of facts about me. I wanted a layer I own and maintain, so I built my own MCP server. Here's how it went."
date: 2026-06-16
---

I wanted my AI to know more about me than the built-in personalization does, which is really just a collection of data points. So I built my own personal MCP server to be that layer, and I maintain it myself. Here's how it went.

## How I approached it

Strip away the protocol and an MCP server is a plain webserver: it exposes an endpoint that lists the tools it offers, and clients call them. So the real work was never the protocol, it was deciding what sits behind that endpoint.

```
  claude.ai      Claude Code      …any MCP client
      │               │                  │
      └───────────────┼──────────────────┘
                      │   one front door — OAuth
              ┌───────▼────────┐
              │   auth proxy   │
              └───────┬────────┘
                      │
            ┌─────────▼─────────┐
            │    personal-mcp   │   ← the layer
            │  service registry │
            └─────────┬─────────┘
                      │
   ┌──────────┬───────┴───────┬───────────┐
   │          │               │           │
┌──▼──────┐ ┌─▼───────┐ ┌─────▼───┐ ┌─────▼───┐
│ Mealie  │ │ Immich  │ │Paperless│ │  media  │
└─────────┘ └─────────┘ └─────────┘ └─────────┘
  recipes     photos     documents    shows…
```

Each piece is a service, a small contract: what it's called, what credentials it needs, how it registers its tools. One registry lists them all, and adding a new part of myself is a folder and a line.

The plan is for this to be the single front door for all my services, not just a one-off for recipes.

## My first tenant
Recipes were always a hassle for me. I'd find one online, want to adjust it, and ask Claude about the tradeoffs of each change — then I'd have to track all those changes and amend them in Mealie by hand. I wanted Claude to just do that part, since it already has the context for the change. That's why recipes became the first tenant.

When a client connects, the Mealie tenant hands it this set of tools:

```
recipes     mealie_search_recipes              find a recipe
            mealie_get_recipe                  read one in full
            mealie_create_recipe               write a new one, in my style
            mealie_update_recipe               change an existing one

meal plan   mealie_get_meal_plan               see what's scheduled
            mealie_create_mealplan_entry       put a meal on a day
            mealie_remove_mealplan_entry       take one off

shopping    mealie_get_shopping_lists          list my shopping lists
            mealie_get_shopping_list_items     read what's on one
            mealie_add_shopping_item           add a single item
            mealie_add_recipe_to_shopping_list add a whole recipe's ingredients
            mealie_check_shopping_item         tick something off

vocabulary  mealie_list_vocabulary             list known foods and units
            mealie_update_vocabulary           rename or fix one
            mealie_merge_vocabulary            fold duplicates together
```

So now I just chat to Claude in plain English. We work out the recipe together, I understand it as well as I want to, and once it's settled the tedious part — writing it all into Mealie — takes care of itself. I could have Claude handle my meal plans and shopping lists the same way since those tools are exposed too, I just haven't wired that into how I work yet.

## The hard part was letting it in

A personal layer is useless if no AI can reach it, so making it reachable and safe ended up being most of the work.

I hand-rolled the auth first, a few hundred lines doing the OAuth dance myself. I fought it for a while, deleted it, and handed the whole job to a dedicated auth proxy. Then a quieter problem: for a while the connection never actually worked, because the server was closing its response stream before it finished writing the reply. The AI got back an empty 200, so it looked connected when it wasn't.

The one that stuck with me came once it was live on the internet. I asked whether it was actually secure and went looking, and the auth was letting everyone through. It checked that a valid login happened, never that the login was mine. There was no list of who's allowed in.

Authentication versus authorization, textbook stuff, and I walked straight into it. It matters more here because the thing behind the door is my actual life, so anyone who gets in is reading it. The door has to only open for me.

## From here on out

I'm hoping to keep this going. The hard parts are done now, so from here it's mostly hooking in new services. Each one is another piece of me the AI gets to know, instead of starting over every time I switch to a new one.