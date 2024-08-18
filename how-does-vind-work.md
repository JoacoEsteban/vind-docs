So you've installed Vind, you're having a blast and are wondering how it works? Well, that's one of my favorite topics to talk about.

The Vind engine has 2 parts:

1. Element attribute recollection strategy
2. Element resolution algorithm

Let's dive deeper for each of them.

## Element attribute recollection strategy

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

This tight package of attributes and tag names is enough for Vind to locate the element in successive sessions (we'll talk about this hassle in the future).

Once this information is compiled Vind saves the binding and it's available to use as the element you clicked has been cached before. The clever trick happens once you reload the page and try to use your just-baked binding...

## Element resolution algorithm

![screenshot of of highlighted element using Vind](https://i.imgur.com/CKDTA4S.png)

So when you reload the page and try to use the binding there's some clever tricks going on and all that information we recollected before comes into play.

Vind is lazy when it comes to element resolution. I might change this in the future but for now Vind just tries to locate a binding's element once you execute it or when you hover over the button in order to highlight it.

So what happens in that case? How does Vind manage to find the bound element in a new DOM? That's the element resolution algorithm's job.

In short what it does is grab the tag name, grab all attributes, generate an XPath expression
and query the current document. Let's see a selector based off the computed object from before:

```xml
//a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind']
```

If the result is exactly one element then that element is treated as the result and gets returned. If not, then these nuances kick in until we get a one-element result:

1. The algorithm will try all combinations of attributes amounting to 2^x, being x the amount of attributes (see Pascal's triangle for a visual representation):

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

As you see, if he have 3 attributes for an element the amount of possible selectors is equal to 2^3 = 8

2. It will recursively prepend it's parents and append it's child to the selector in order to augment specificity elevating it to 2^x^y^z being y parents and z children.

```xml
//a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind']
//div/a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind']
//header[contains(@class, 'p-5')]/div/a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind']

//a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind'][svg[title='homepage icon']]
//div/a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind'][svg[title='homepage icon']]
//header[contains(@class, 'p-5')]/div/a[starts-with(href, 'https://vind-works.io')][@name='link to Vind's homepage'][text()='Try Vind'][svg[title='homepage icon']]

...Keep going for all possible combinations...
```

Wow, so basically if the first try fails then we're getting into a recursive 2^x^y^z notation nightmare. Wouldn't that take ages to complete? What if the element isn't even in the DOM?
The answer is yeah, it would take ages. You'd be 100% certain that the element isn't in the DOM and also 100% certain that the user's machine would cease to exist before you go around trying to go through all the combinations.

So I've implemented some tricks in order to mitigate this absurd amount of iterations and end up with some good results:

### If the result is **more than one** element and the combination is the last of its size (being 4, 3, 2, 1) drop it

By querying from most specific to least specific we can leverage this point. If I know that for any most specific combination there's more than one match, then I can assure that all sub-sets of attribute combinations will also give more than one match.

Or in simpler terms:
If `//div[@class='flex']` has 2 matches, `//div` will have at least 2 matches, being the ones inside the most specific selector.

This assumption greatly reduces the amount of iterations, as we can know when to break early.

### If the result is **zero** elements continue

If the result is zero elements then we can continue with the next combination of less specificity and avoid looking for parent/child combinations. This also leverages the fact that if the most specific combination doesn't have any matches then all sub-sets of attribute combinations will also give zero matches. It's the same reasoning as the previous point but in the opposite direction.

In simpler terms:
If `//div[@class='flex']` has 0 matches, `//header/div[@class='flex']` will also have 0 matches, but `//div` might match as it's less specific.

### And of course, If the result is **one** element then return it

If the result is one element then we can return it and stop the iteration. This is the happy path and the one we're looking for.

## Conclusion

So in the end Vind is a lazy engine that tries to find the element you bound by using clever tricks and assumptions in order to reduce the amount of iterations and time it takes to find the element. It's a balance between being fast and being accurate, and I think I've found a good middle ground.

I hope you've enjoyed this deep dive into Vind's internals and I hope you're having a blast using it. If you have any questions or want to know more about Vind feel free to ask me anything. You can open an issue in the [Vind repository](https://vind-works.io/issues) or contact me directly at [dev@joaco.io](mailto:dev@joaco.io).
