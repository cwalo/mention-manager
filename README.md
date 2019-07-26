# MentionManager

### A study in mobile feature design
#### This is broken into two parts
* Designing the system (below)
* Building the system

---

The '@ mention' is a pretty ubiqitous feature these days and for any modern communication application it's effectively a prerequisite. 

On the surface, it seems like a relatively simple problem:

1) The user types an `@` symbol followed by another user's name (or part of a name)
2) A list of user suggestions appears
3) The user taps one of these suggestions
4) The text is autocompleted, the name is highlighted and the user carries on
...
Voila!

That's the user story for this feature, but we have to build this thing, so let's _really_ pick this system apart.

Given the following system design considerations:
- A mention is represented by the following syntax: `[@somename](user:<id>)` (similar to markdown link formatting).
- A user can be searched by their username or their real name (this means we need to consider spaces).
- If an autocompleted mention is broken by backspacing or deleting, we should remove any applied attributes.
- A mention needs to be serialized when sent and deserialized when received.
- Tapping a deserialized mention should display the respective user.

1) The user types an `@` symbol (the delimiter) followed by another user's name (or part of a name).
2) We need to enter a searching state when our delimiter is typed.
3) While in this search state, extract the search term.
4) Query a list of users for any that contain the search term. In some cases, we can query a local list, but in most cases we will be provided with a search API that will return search results asynchronously.
5) Display a list view of the returned results.
6) The user taps a result.
7) The range of text from the delimiter to the length of the search term is replaced with the result. 
    - Simultaneously, apply attributes to the text that signify that it's a mention.
8) On send, serialize the message tokenizing any mentions.

Let's cover some things that come to mind when I look at this rough design. There are lots of cases that we'll cover as we start to build this thing, but first...

- I'm not necessarily looking for a one-size fits all solution, but I want to build a system that I can use in multiple contexts, so keeping things decoupled or loosely coupled is pretty important. 
- I need a system that can handle asynchronous queries.
- There's an implied type-ahead feature here, so we need to be careful about spamming the API. We'll probably want to do some request throttling and caching to mitigate this.
- If spaces in names weren't allowed, it would be really easy to know when our search state ends, but because that's a requirement we need to think carefully about that. We might need to get a bit clever...
- We'll be manipulating text in place so it will be important to be strict when dealing with string ranges and character encoding (this one comes from experience 😬)

Platform-specific considerations (iOS):
- For long-form text entry, we'll want to use `UITextView`.
- I'm wary of making the `MentionManager` become the textview's delegate because other objects need to know about those events, too. So, lets investigate the tools we have to deal with that.
- We're talking about mentioning 'users,' but the `MentionManager` shouldn't know anything about our objects. Let's create a protocol that any object can conform to in order to become mention-able.

Okay, I've think we've thought through a lot, so lets start building this. Like any good problem, there will be unknown unknowns and we'll tackle those as they come.
