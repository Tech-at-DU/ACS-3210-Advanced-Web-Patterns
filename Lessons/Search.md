# 📄🔍 Day 2: Search

<!-- > -->

<!-- omit in toc -->
## ⏱ Agenda {docsify-ignore}

1. [🏆 Learning Outcomes](#%F0%9F%8F%86-learning-outcomes)
1. [[**20m**] Warm Up: Implement Pagination](#%5B%2a%2a20m%2a%2a%5D-warm-up%3A-implement-pagination)
1. [[**20m**] TT: Search](#%5B%2a%2a20m%2a%2a%5D-tt%3A-search)
   1. [Mongoose](#mongoose)
1. [[**10m**] 🌴 BREAK {docsify-ignore}](#%5B%2a%2a10m%2a%2a%5D-%F0%9F%8C%B4-break-%7Bdocsify-ignore%7D)
1. [[**30m**] Activity: RegEx Challenges](#%5B%2a%2a30m%2a%2a%5D-activity%3A-regex-challenges)

<!-- > -->

## 🏆 Learning Outcomes

By the end of this lesson, you should be able to...

- Understand how search works, and how to implement it


## [**20m**] Warm Up: Implement Pagination

Follow along with [this tutorial](https://medium.com/javascript-in-plain-english/simple-pagination-with-node-js-mongoose-and-express-4942af479ab2) and write a simple pagination implementation in a sample project.

<!-- > -->

## [**20m**] TT: Search

![autocomplete](assets/autocomplete.gif)

Think of how many websites you visit where a **Search Form** or an **Autocomplete** dropdown exists. This is a common web pattern or recipe called Simple Search.

A **Simple Search** is a search based on the text of one or a few attributes, e.g. on words in a title or body of articles or comments.

We're going to look at an implementation of Simple Search for Mongoose using Regex's. (The SQL implementation uses the SQL operator `LIKE`.)

Once we can search, we can then *paginate* the responses!

<!-- > -->

### Mongoose

![mongoose](assets/mongoose.png)

In mongoose, we can search by passing a Regex (regular expression) for the term we want to search for. Prefer **`async`/`await`** over callbacks — that is the hire-bar pattern for Express handlers.

```js
// Literal pattern when you know the term at write time:
const docs = await User.find({ name: /john/i });
```

For user input, turn the string into a `RegExp`. Pass the **pattern** as the first argument and the **flags** as the second — do **not** wrap the term in `/.../` inside the string (that would search for literal slashes).

**Escape first.** User-controlled strings often contain `.`, `*`, `(`, etc. Escape special characters **before** you call `new RegExp(...)`.

```js
function escapeRegex(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

const term = String(req.query.term || '').trim();
const regex = new RegExp(escapeRegex(term), 'i');
const docs = await User.find({ name: regex });
```

**Tip:** User-controlled regex can blow up your CPU (**ReDoS**). Always escape (or use a vetted library), and prefer bounded / indexed search when the dataset grows.

### Express route (await + errors)

Ship it as a real handler — coerce the term, escape, `await` the query, and return JSON errors (same hire-bar shape as Day 1 pagination). Aligns with Pete’s Pets P01 after the RegExp/await fix.

```js
app.get('/users/search', async (req, res) => {
  try {
    const term = String(req.query.term || '').trim();
    if (!term) {
      return res.status(400).json({ error: 'term query param is required' });
    }

    const regex = new RegExp(escapeRegex(term), 'i');
    const docs = await User.find({ name: regex }).limit(50).lean();

    return res.json({ data: docs, term });
  } catch (err) {
    console.error(err);
    return res.status(500).json({ error: 'Search failed' });
  }
});
```

Empty term → `400`. DB failure → `500` with a clear message (no stack leak). Cap with `.limit(...)` so one request cannot dump the collection.

> Same escape + await pattern as [Proud Pete's Pet Emporium — Simple Search](https://github.com/Tech-at-DU/Proud-Petes-Pet-Emporium) (Wave 0.2 fix-before-wave1).

<!--  -->

## [**10m**] 🌴 BREAK {docsify-ignore}

<!--  -->

## [**30m**] Activity: RegEx Challenges

![regex](assets/regex.jpeg)

Need a refresher? Read up on [this guide to RegEx](https://www.freecodecamp.org/news/a-quick-and-simple-guide-to-javascript-regular-expressions-48b46a68df29/), and then write RegExs for the following:

1. Return anything that has the letter "x"
1. Return anything that contains "ar"

**Stretch Challenges**

1. Return anything that starts with "the"
1. Return anything that ends with "ed"


Make sure to run it against test inputs!

<!-- > -->
