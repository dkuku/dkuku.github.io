---
title: "Pseudolocalization in Phoenix with gettext_pseudolocalize"
meta_title: ""
description: "Understanding Gettext in Elixir Applications. Gettext is a widely adopted internationalization (i18n) system that helps developers make their applications available..."
date: 2024-12-27T15:51:49Z
image: "https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jzvb0peoetdcksy9roxq.png"
categories: ["Development", "Tutorial"]
author: "Daniel Kukula"
tags: ["elixir", "phoenix", "i18n", "gettext"]
draft: false
---

# Understanding Gettext in Elixir Applications

Gettext is a widely adopted internationalization (i18n) system that helps developers make their applications available in multiple languages. Originally developed for GNU projects, it has become a standard solution across many programming languages and frameworks, including Elixir and Phoenix.

At its core, Gettext works by replacing hardcoded strings in your code with function calls that look up translations from a translation list. Instead of writing:

```elixir
def hello do
  "Hello, World!"
end
```

You write:

```elixir
def hello do
  gettext("Hello, World!")
end
```

This small change enables your application to display different text based on the user's locale. The translations themselves are stored in PO (Portable Object) files, typically organized by language:

```
priv/gettext/en/LC_MESSAGES/default.po
priv/gettext/es/LC_MESSAGES/default.po
priv/gettext/fr/LC_MESSAGES/default.po
```

Phoenix applications come with Gettext support out of the box, making it straightforward to build multilingual web applications. However, during development, it's common to face challenges like:

- Ensuring UI elements can accommodate longer translations
- Identifying hardcoded strings that should be translated
- Testing how the application behaves with different languages
- Verifying that all strings are properly marked for translation

This is where pseudolocalization comes in...

# Pseudolocalization: Making Translation Issues Visible

When developing international applications, it's crucial to identify potential issues with translations before sending text to actual translators. This is where pseudolocalization comes in - a technique that transforms your English text into a modified version that simulates characteristics of other languages while remaining somewhat readable.

The `gettext_pseudolocalize` task transforms your Gettext strings into a special format that helps identify several common internationalization issues:

```
"Account Settings" → ⟦Åċċøüñť Șêťťíñğš~~~~~~~⟧
```

Let's break down what this transformation does:

1. **Diacritical Marks**: Letters are replaced with accented versions (A → Å, c → ċ). This helps identify font and character encoding issues.

2. **Text Expansion**: The `~~~~~` padding simulates how translations might be longer than the original text. Many languages, like German or Finnish, can be 30-40% longer than English.

3. **Brackets**: The `⟦` and `⟧` markers help identify string truncation and concatenation issues. If you see a broken bracket, you know something's wrong with string handling.

For example, if you have a UI element that looks fine with `"Email"` but breaks with `⟦Ȅɱàíĺ~~~~~~~~~~⟧`, you know you need to adjust your layout to accommodate longer translations.

Common issues this helps identify:
- Hard-coded string width assumptions
- Improper text wrapping
- Font compatibility problems
- String concatenation issues
- Missing translations

The beauty of pseudolocalization is that the text remains somewhat readable for developers while still testing the internationalization infrastructure of your application.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jzvb0peoetdcksy9roxq.png)

# Implementing Pseudolocalization in Your Phoenix Application

Let's walk through setting up `gettext_pseudolocalize` in a new Phoenix project and see it in action.

First, create a new Phoenix project:

```bash
mix phx.new gtx
cd gtx
mix ecto.setup
```

## Adding the Dependency

Add `gettext_pseudolocalize` to your dependencies in `mix.exs`:

```elixir
def deps do
  [
    # ... other deps ...
    {:gettext_pseudolocalize, "~> 0.1"}
  ]
end
```

Then fetch the dependency:

```bash
mix deps.get
```

## Configuration

We need to configure two things:

1. Tell Gettext to use our pseudolocalization plural forms handler for `xx` language in `config/config.exs`:

```elixir
config :gettext, :plural_forms, GettextPseudolocalize.Plural
```

2. Set the default locale to "xx" (our pseudolocale) in development in `config/dev.exs`:

```elixir
config :gtx, GtxWeb.Gettext, default_locale: "xx"
```

## Generating Pseudolocalized Translations

Now we can extract all translatable strings and create pseudolocalized versions:

```bash
mix gettext.extract
mix gettext.pseudolocalize
```

This will:
1. Extract all `gettext` calls into PO template files
2. Generate a pseudolocalized version in the "xx" locale

# Seeing It in Action

When you first run your Phoenix application, you might notice something interesting - not all strings are pseudolocalized. This actually helps us identify strings that aren't properly set up for internationalization!

For example, in the default Phoenix homepage, you'll see this string in its original form:
```
Peace of mind from prototype to production.
```

This tells us that this string isn't using Gettext and therefore won't be translatable in production. Let's fix that!

Edit `lib/gtx_web/controllers/page_html/home.html.heex` and change the hardcoded string to use gettext:

```heex
- Peace of mind from prototype to production.
+ {gettext("Peace of mind from prototype to production.")}
```

Now run the extraction and pseudolocalization commands:
```bash
mix gettext.extract
mix gettext.pseudolocalize
```

Looking at `priv/gettext/xx/LC_MESSAGES/default.po`, you'll see our string has been added and pseudolocalized:

```po
#: lib/gtx_web/controllers/page_html/home.html.heex:57
#, elixir-autogen, elixir-format
msgid "Peace of mind from prototype to production."
msgstr "⟦Ƥêàċê øƒ ɱíñđ ƒȓøɱ ƥȓøťøťÿƥê ťø ƥȓøđüċťíøñ.~~~~~~~~~~~~~~~~~~⟧"
```

When you refresh your page, you'll now see the pseudolocalized version of the string. This visual difference makes it easy to spot which strings in your application are properly internationalized (pseudolocalized) and which ones are still hardcoded (original English).

This is one of the key benefits of using pseudolocalization during development - it helps you identify missed internationalization opportunities before they become problems in production.

# Best Practices and Tips

## When to Use Pseudolocalization

1. **Early Development**
   - Enable pseudolocalization from the start of development
   - Makes internationalization issues visible before they become deeply embedded
   - Helps establish good i18n habits in your team

2. **UI Reviews**
   - Use pseudolocalization during UI reviews to spot potential layout issues
   - Pay special attention to:
     - Buttons and navigation elements
     - Forms and labels
     - Table headers
     - Alert messages

## Common Issues to Watch For

1. **Text Truncation**
   - If you see broken brackets `⟦` or `⟧`, your UI is cutting off text
   - Look for CSS `text-overflow`, fixed widths, or other constraints

2. **Layout Breaking**
   - Watch for elements that jump to new lines or overlap
   - The `~~~~~` padding helps simulate longer translations
   - If it breaks with pseudo-text, it will likely break with real translations

3. **Hardcoded Strings**
   - Any text that appears in regular English is likely missing gettext
   - Common places to check:
     - Error messages
     - Flash messages
     - Email templates
     - Dynamic content

# Conclusion

Internationalization is often treated as an afterthought, leading to rushed implementations, broken layouts, and frustrated users. The `gettext_pseudolocalize` library helps tackle these challenges early in the development cycle by making translation-related issues immediately visible.

## Key Benefits

- **Early Detection**: Spot internationalization issues before they reach production
- **Visual Feedback**: Instantly see which strings are properly internationalized
- **Layout Testing**: Identify UI components that won't scale with different language lengths
- **Developer Awareness**: Builds good i18n habits in your development team

## Real-world Impact

Instead of waiting for translations or testing with actual languages, pseudolocalization provides immediate feedback during development. This means:
- Fewer emergency fixes when adding new languages
- More consistent user experience across all locales
- Reduced cost of internationalization
- Better preparation for global markets

## Next Steps

The library is open source and available on [GitHub](https://github.com/dkuku/gettext_pseudolocalize). You can:
- Star the repository
- Report issues or suggest improvements
- Contribute to the codebase
- Share your experience with the community

Remember, good internationalization isn't about just translating strings—it's about building applications that gracefully adapt to different languages and cultures. `gettext_pseudolocalize` is one tool to help achieve that goal.

Happy internationalizing! 🌍
