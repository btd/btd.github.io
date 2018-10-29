---
title: Initials Avatar
key: 20181028
tags: image avatar initials
---

On time I had a task to generate avatars from initials of user names, to make default placeholder more fancy.
I think need to start from how this could look in different langs.

## Names and Initials around the world

Let first talk about initials. From Cambridge dictionary we can get that this is 'the first letter of a name, especially when used to represent a name'. 

Now we need to see how initials and names look in different countries. 
First of all not every name is written in english and do not need to assume that english grammar spread to other langs.

W3C has very good write up about names. [There it is](https://www.w3.org/International/questions/qa-personal-names).

Several examples:

- Denis Bardadym - DB
- Бардадым Денис - БД
- Björk Guðmundsdóttir - BG
- 毛泽东 - there is no initials! 

Do not need to think that names is always has first name and last name, and they are written in the order - first and next last.

For this problem we will need to define list of languages and/or locales we want to support. For simplicity i will assume for now english only names, but in real world application you need to take several more (at least common european locales like cyrillic, but this all depends from task) and also if it is applied ask user name in english.

## Getting avatar

I need to show this avatar not only in browser but also in html email. This restricts me to generating widely supported format like gif or png.

If I would need to show it in browser only i would use SVG for this.

Anyway lets start from SVG template.

```xml
<?xml version="1.0"?>
<svg viewBox="0 0 100 100" version="1.1" xmlns="http://www.w3.org/2000/svg">
  <style>
    text {
      font-family: Ubuntu,"Segoe UI",Roboto,"Helvetica Neue",Arial,sans-serif;
      font-size: 40;
      fill: ${color};
    }
  </style>
  <rect width="100%" height="100%" fill="${backgroundColor}"/>
  <text text-anchor="middle" x="50%" y="50" dy="15">${text}</text>
</svg>
```

I am using square picture there because i can always round it with CSS (even in email). If you need round avatar change `<rect>` to be `<circle>`. Also you can see template variables for `color`, `backgroundColor` and `text` (initials will be there). MDN has good articles about different SVG elements.

I want to generate different colors avatars depending intials. You can choose random colors, but i think it makes more
sense to show the same color for the same initials.
For colors i used in my implementation [this](https://evergreen.segment.com/components/colors) and [schemeCategory20*](https://bl.ocks.org/pstuffa/3393ff2711a53975040077b7453781a9).

Avatars like this we will have at the end:

![Example of avatar at the end](/assets/images/posts/initials-avatar/ss.png)

And round version with `border-radius: 50%`:

![Example of avatar at the end](/assets/images/posts/initials-avatar/ss.png){: .circle }

For colors in this text i will use (from d3):
```js
const COLORS = [
  { color: 'rgb(31, 119, 180)', backgroundColor: 'rgb(174, 199, 232)' },
  { color: 'rgb(255, 127, 14)', backgroundColor: 'rgb(255, 187, 120)' },
  { color: 'rgb(44, 160, 44)', backgroundColor: 'rgb(152, 223, 138)' },
  { color: 'rgb(214, 39, 40)', backgroundColor: 'rgb(255, 152, 150)' },
  { color: 'rgb(148, 103, 189)', backgroundColor: 'rgb(197, 176, 213)' },
  { color: 'rgb(140, 86, 75)', backgroundColor: 'rgb(196, 156, 148)' },
  { color: 'rgb(227, 119, 194)', backgroundColor: 'rgb(247, 182, 210)' },
  { color: 'rgb(127, 127, 127)', backgroundColor: 'rgb(199, 199, 199)' },
  { color: 'rgb(188, 189, 34)', backgroundColor: 'rgb(219, 219, 141)' },
  { color: 'rgb(23, 190, 207)', backgroundColor: 'rgb(158, 218, 229)' }
];
```

Now i need to get initials from full name and somehow choose color to use.
I assume my full name will look like `WORD[0] ... WORD[N]` and 
i want 1-2 letter initials from first letters of first and last words in full name.

```js
function getInitials(string) {
  const names = string.split(/s+/);
  let initials = names[0].substr(0, 1).toUpperCase();
    
  if (names.length > 1) {
    initials += names[names.length - 1].substr(0, 1).toUpperCase();
  }
  return initials;
};
```

As we have initials we need hash function to map this string to `0...COLORS.length` integer.

```js
function hashCode(s) {
  const str = String(s);
  let hash = 0;
  let char;
  if (str.trim().length === 0) return hash;
  for (let i = 0; i < str.length; i++) {
    char = str.charCodeAt(i);
    hash = (hash << 5) - hash + char;
    // Convert to 32bit integer
    hash &= hash;
  }
  return Math.abs(hash);
}
```

This is copied from [evergreen](https://github.com/segmentio/evergreen). 

Combining all peaces in this function:

```js
function getSVGAvatar(name) {
  const initials = getInitials(name);
  const hash = hashCode(initials);
  const color = COLORS[hash % COLORS.length];
  return svgTemplate({ ...color, text: initials });
}
```

We can pass optional hash code to this function also if e.g our name is some placeholder but we want different colors (for example name: anonymous and hash code text: id_134 ). Or we can get hash code from full name, this way we will have
different colors even for the same initials, but it is probably easier to just choose random.

This is the end if you need SVG only avatar. But i need png image also. For this i will use awesome package [sharp](https://www.npmjs.com/package/sharp). It is `libvips` bindings to node.js. I used this module when i was need to get
`phash` of the picture (see my pachage `sharp-phash`).

With `sharp` it is easy as:

```js
function getPNGBufferAvatar(name, size = 150) {
  const tplBuffer = Buffer.from(getSVGAvatar(name).trim());//we need to avoid spaces at start of svg
  return sharp(tplBuffer)
    .resize(size, size)
    .png()
    .toBuffer();
}
```

And that is all! As fallback picture (when i could not generate avatar for input name) i used icon from fontawesome coloring it with the same way using 2 colors.

Use this [issue](https://github.com/btd/btd.github.io/issues/1) for comments corrects and everything.
