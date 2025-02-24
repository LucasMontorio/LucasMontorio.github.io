Here’s a small discovery I made regarding Ruby regex rules and whitespace characters, that made me scratch my head for a moment:


Let’s have a look the following string, extracted from an email body:

```
From : 	John DOE <test@test.com>
```
Please note that the space character between `'From'` and the column (`‘:’`) is a Non-Breaking Space Character (`U+00A0` in Unicode), while the other spaces in this string are regular whitespaces (`U+0020`).

Let’s now consider the following regex rule, defined in a Ruby constant:

```
REGEX = /(?:From)\s*:\s*(?:.*?<)?([^<>\s]+@[^>\s]+)(?:>)?/i
```

When testing the mentioned string against this regex in a Ruby console, **we don’t get any match**:
```
REGEX.match(‘From : 	John DOE <test@test.com>’)
=> nil
```

**Why, you may wonder?**
The reason for this is that the `\s` matcher does not look for Non-Breaking Space Characters. In order to make it work, we need to update the regex to explicitly expect NBSC characters, as follows:
```
REGEX = /(?:From)[\s\u00A0]*:\s*(?:.*?<)?([^<>\s]+@[^>\s]+)(?:>)?/i
```
**Everything looks fine up to that point.**

However - here it becomes weird - when testing the original regex rule (the first one, without the `\u00A0` part) on the same string in an interactive visualiser (https://regexr.com/ for instance), **there is a match**:

![Screenshot of the test made on Regexr](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1c11o1iv7hygfzsgrvtf.png)



My understanding of the situation is that the interactive Regex visualiser actually converts the NBSC to regular whitespace when copy-pasting the string into its text input, simply because the browser interprets it as a regular whitespace in its HTML rendering.

This little experiment highlights the importance of testing regex patterns in the exact environment where they will be used. While online tools can be helpful for quick tests, they don't always accurately represent how the regex will behave in your production environment.

_PS: It is worth mentioning that the string under scrutiny was copy-pasted from the original email at every stage of this experiment, meaning that the string itself wasn’t transformed by the copy-pasting operation._
