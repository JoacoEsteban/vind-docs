---
title: How does Vind work?
description: A technical dive into Vind's inner workings.
date: 13/08/2024
hero_img: https://i.imgur.com/IRieyNk.png
---

So you've installed Vind, you're having a blast and are wondering how it works? Well, that's one of my favorite topics to talk about.

The Vind engine has 2 parts:

1. [Element attribute recollection strategy](#element-attribute-recollection-strategy)
2. [Element resolution algorithm](#element-resolution-algorithm)

Let's dive deeper for each of them.

## Element attribute recollection strategy{:#element-attribute-recollection-strategy}

![Image of registration process using Vind](https://i.imgur.com/i3qy1si.png)

When you click the element you want to bind Vind grabs the HTML node and recollects all necessary attributes that will later be used to build a selector and find such element in the future.
What Vind collects is the following:

1. Element Tag name (eg: `button`, `a`, `div`)
2. All of these attributes (if present)

- `id`
- `name`
- `title`
- `aria-labelledby`
- `aria-label`
- `href`
- `class`
- `text`

Let's use an example:

```html
<a href="https://vind-works.io" name="link to Vind's homepage">Try Vind</a>
```

This element would translate to this

```json
{
  "tagName": "a",
  "attrs": [
    {
      "name": "href",
      "value": "https://vind-works.io"
    },
    {
      "name": "name",
      "value": "link to Vind's homepage"
    },
    {
      "name": "text",
      "value": "Try Vind"
    }
  ]
}
```

After that Vind will look at the node's parent and do the same recursively until it hits the root node.
It will also do the same with its children if the node has exactly one child.

It would end up like this:

```json
{
  "tagName": "a",
  "attrs": [
    {
      "name": "href",
      "value": "https://vind-works.io"
    },
    {
      "name": "name",
      "value": "link to Vind's homepage"
    },
    {
      "name": "text",
      "value": "Try Vind"
    }
  ],
  "parent": {
    "tagName": "div",
    "attrs": [],
    "parent": {
      "tagName": "header",
      "attrs": [
        {
          "name": "class",
          "value": "p-5"
        }
      ],
      "parent": null
    }
  },
  "children": [
    {
      "tagName": "svg",
      "attrs": [
        {
          "name": "title",
          "value": "homepage icon"
        }
      ],
      "children": null
    }
  ]
}
```

This tight package of attributes and tag names is enough for Vind to locate the element in successive sessions (I'll expand on this in the future. It comes with complications).

Once this information is compiled Vind saves the binding. It is now available to use as we have a direct reference to the bound element by the click event. The clever trick happens once you reload the page and try to use your freshly-baked binding...

## Element resolution algorithm{:#element-resolution-algorithm}

![screenshot of of highlighted element using Vind](https://i.imgur.com/CKDTA4S.png)

So when you reload the page and try to use your binding there will be some clever tricks going on where all that information we recollected before will come into play.

Vind is lazy when it comes to element resolution. I might change this in the future but for now Vind just tries to locate a binding's element once you execute it or when you hover over the button in order to highlight it.

So what happens in that case? How does Vind manage to find the bound element in a new DOM? That's the element resolution algorithm's job.

In short what it does is grab the **tag name**, grab **all attributes**, generate an **[XPath](https://www.w3schools.com/xml/xpath_intro.asp) expression**
and **query the document**. Let's see a selector based off the computed object from before:

```xml
//a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind']
```

If the result of this query is **exactly one element** then we can say we found it. If not, then these nuances kick in until we get a one-element result:

1. The algorithm will try all combinations of attributes amounting to $2^x$, being x the amount of attributes (see [Pascal's triangle](https://en.wikipedia.org/wiki/Pascal's_triangle) for a visual representation):

```xml
//a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind']
//a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage']
//a[starts-with(href, 'https://vind-works.io')][text()='Try Vind']
//a[text()='Try Vind'][@name='link to Vind's homepage']
//a[starts-with(href, 'https://vind-works.io')]
//a[@name='link to Vind's homepage']
//a[text()='Try Vind']
//a
```

In english this would be:

- An `a` element with an `href` equal to `https://vind-works.io`, a `name` equal to `link to Vind's homepage` and a `text` equal to `Try Vind`
- An `a` element with an `href` equal to `https://vind-works.io` and a `name` equal to `link to Vind's homepage`
- An `a` element with an `href` equal to `https://vind-works.io` and a `text` equal to `Try Vind`
- An `a` element with a `text` equal to `Try Vind` and a `name` equal to `link to Vind's homepage`
- An `a` element with an `href` equal to `https://vind-works.io`
- An `a` element with a `name` equal to `link to Vind's homepage`
- An `a` element with a `text` equal to `Try Vind`
- An `a` element

As you see, if he have 3 attributes for an element the amount of possible selectors is equal to $2^3 = 8$

2. It will recursively prepend it's parents and append it's child to the selector in order to augment specificity elevating it to $2^{x^{y^z}}$ being $y$ parents and $z$ children.

```xml
//a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind']
//div/a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind']
//header[contains(@class, 'p-5')]/div/a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind']

//a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind'][svg[title='homepage icon']]
//div/a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind'][svg[title='homepage icon']]
//header[contains(@class, 'p-5')]/div/a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind'][svg[title='homepage icon']]

...Keep going for all possible combinations...
```

In english this would be:

- An `a` element with an `href` equal to `https://vind-works.io`, a `name` equal to `link to Vind's homepage` and a `text` equal to `Try Vind`
- An `a` element with an `href` equal to `https://vind-works.io`, a `name` equal to `link to Vind's homepage` and a `text` equal to `Try Vind` inside a `div` element
- An `a` element with an `href` equal to `https://vind-works.io`, a `name` equal to `link to Vind's homepage` and a `text` equal to `Try Vind` inside a `div` element inside a `header` element with a class containing `p-5`

Then the child:

- An `a` element with an `href` equal to `https://vind-works.io`, a `name` equal to `link to Vind's homepage` and a `text` equal to `Try Vind` containing a `svg` child with a `title` equal to `homepage icon`
- An `a` element with an `href` equal to `https://vind-works.io`, a `name` equal to `link to Vind's homepage` and a `text` equal to `Try Vind` containing a `svg` child with a `title` equal to `homepage icon` all inside a `div` element
- An `a` element with an `href` equal to `https://vind-works.io`, a `name` equal to `link to Vind's homepage` and a `text` equal to `Try Vind` containing a `svg` child with a `title` equal to `homepage icon` all inside a `div` element and all inside a `header` element with a class containing `p-5`

Wow, so basically if the first try fails then we're getting into a recursive $2^{x^{y^z}}$ notation nightmare. Wouldn't that take ages to complete? What if the element isn't even in the DOM?
The answer is yeah, it would take ages. Heat death of the universe scale.

So in order to mitigate this absurd amount of iterations we can implement some clever proofs and end up with good results:

### If the result is _zero_ elements continue with the next attribute combination

If the result is **zero elements** then we can **avoid looking for parent/child combinations**. If we know that for any specific combination there's **zero matches**, then we can assure that **all super-sets of higher specificity will also give zero matches**.

In simpler terms:
If `//div[@class='flex']` has 0 matches, `//header/div[@class='flex']` will also have 0 matches.

### If the result is _more than one_ element and the attibute combination is the last of its size, abort

We are able to leverage this point by querying from most specific to least specific.
If we know that for any **most** specific combination there's **more than one match**, then we can assure that **all sub-sets of attribute combinations will also give more than one match**.

Or in simpler terms:
If `//div[@class='flex']` has 2 matches, `//div` will have at least 2 matches, being the ones inside the most specific selector.

This causes a premature throw of `NoUniqueXPathExpressionErrorForElement`

```typescript
class NoUniqueXPathExpressionErrorForElement extends VindError {
  constructor() {
    super(
      'That element cannot be bound because Vind cannot uniquely identify it.',
      'NO_UNIQUE_XPATH_EXPRESSION',
    )
  }
}
```

So yeah, this is the sad path.

### And of course, If the result is **one** element then return it

If the result is one element then we can return it and stop the iteration. This is the happy path and the one we're looking for.

## TLDR?

So in the end Vind is a lazy engine that tries to find the element you bound by using clever tricks and assumptions in order to reduce the amount of iterations and time it takes to find the element. It's a balance between being fast and being accurate, and I think I've found a good middle ground.

I hope you've enjoyed this deep dive into Vind's internals and I hope you're having a blast using it. If you have any questions or want to know more about Vind feel free to ask me anything. You can open an issue in the [Vind repository](https://vind-works.io/issues) or contact me directly at [dev@joaco.io](mailto:dev@joaco.io).
