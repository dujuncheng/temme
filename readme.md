[![Build Status](https://img.shields.io/travis/shinima/temme/master.svg?style=flat-square)](https://travis-ci.org/shinima/temme) [![Coverage Status](https://img.shields.io/coveralls/shinima/temme/master.svg?style=flat-square)](https://coveralls.io/github/shinima/temme?branch=master) [![NPM Package](https://img.shields.io/npm/v/temme.svg?style=flat-square)](https://www.npmjs.org/package/temme)


**CAUTION** Temme is in an early stage, so the syntax may change in the future. Some parts of this doc is **outdated**.

# Temme

[![Greenkeeper badge](https://badges.greenkeeper.io/shinima/temme.svg)](https://greenkeeper.io/)

Temme is a concise and convenient jQuery-like selector for node crawlers. Temme is built on top of [cheerio](https://github.com/cheeriojs/cheerio), so the syntax is very similar to cheerio/jQuery. Temme add some extra grammar to enable selecting multiple items at one time. Try temme in [playground](http://shinima.pw/temme-playground/).

# Install

Install from npm:

`npm install temme` or `yarn add temme`

# Usage

```typescript
// es-module
import temme, { defineFilter } from 'temme'
// or use require
// const temme = require('temme').default

const html = '<div id="answers"> <a name="tab-top"></a> <div id="answers-header"> <div class="subhe......'

const temmeSelector = `
  .answer@answers (
    .votecell .vote-count-post{$upvote},
    .post-text{$postText},
    .user-info .user-details>a{$userName},
    .comments{$comments},
  )
`
const result = temme(html, temmeSelector)
```

# Concepts

## 匹配(MATCH)

匹配的含义是: 给定一个根节点和一个选择器, 找到符合该选择器的节点. Temme使用普通的CSS选择器作为匹配语法, 具体由cheerio来执行匹配.

## 捕获(CAPTURE)

捕获的含义是: 给定一个节点和一个选择器, 将该节点的内容放到结果的对应字段中, 并返回结果. 节点的内容可以是节点文本, 节点HTML, 或者节点的某个特性值(例如\<a>标签的`href`特性); 一般情况下, 返回的捕获结果是一个JavaScript对象, 包含了多个字段, 存放了节点不同的内容, 字段由选择器指定.

Temme定义了一个新的选择器语法(TemmeSelector), 该语法同时包含匹配信息和捕获信息. 其中匹配部分用来表示"我们想从什么样的节点中获取数据", 捕获部分用来表示"从节点中获取什么样的数据".

# Grammar Syntax and Semantics

## Value-Capture `$`

Value-capture is a basic form of capture. It uses `$foo` syntax. Value-capture can be placed at attribute part(in brackets) to capture attribute value, or at content part(in curly braces) to capture text/html.

In the following simple example, we want to extract the title and url of this [stackoverflow question][]. The CSS selector that matches the target \<a> tag is `#question-header .question-hyperlink`. Our temme-selector add `[href=$url]{$title}` after the normal selector. `[href=$url]` means that we want to *capture the href to `.url`* (put the `href` attribute in `url` field of the result); And `{$title}` means that we want to *capture text content of the node to `.title`*. The output is an object with `.url` and `.title`.

### Simple example: Extract title and url of stackoverflow problem page

```HTML
<!-- DOM structure -->
<div id="question-header">
  <h1 itemprop="name">
    <a href="/questions/291978/short-description-of-the-scoping-rules" class="question-hyperlink">
      Short Description of the Scoping Rules?
    </a>
  </h1>
</div>
```

```javascript
const temmeSelecotr = `#question-header .question-hyperlink[href=$url]{$title}`

// output of temme(html, temmeSelector):
const output = {
  "url": "/questions/291978/short-description-of-the-scoping-rules",
  "title": "Short Description of the Scoping Rules?"
}
```

The output of value-capture will be an object containing the data at specific fields that we define. We can use a single `$` to make a default value-capture, and the result will be a single value. It is useful when we just need capture one field. In the above example, temme selector `#question-header .question-hyperlink{$}` will lead to the output being string `'Short Description of the Scoping Rules?'`.

## Array-capture `@`

Array-capture is another form of capture. It is useful when we want to capture an array of similar items. We need place `@foo` after normal CSS selector (called parent-selector), and define several children selectors within trailing parenthesis. This means: for every node (called parent-node) that matches parent-selector, execute the children selectors one-by-one; every parent-node corresponds an object as result, and the result of array-capture is the array composed of parent-node result. The array itself will be the `foo` field of the upper-level result. Like default value-capture, we could just use a signle dollar sign to make a default array-capture, and the array will be the result of upper-level result.

In the following example, we want to capture all answers in this [stackoverflow question][]. We know that every answer is in an `<div class="answer">...</div>` and we use `.answer@answers` as the parent selector. There are 4 value-capture selectors in the parenthesis and these children selectors will capture four different parts of an answer. The running procedure is straightforward: `.answer` selector will match an array of answer-div; for each answer-div, run the children selectors, and get an answer-object `{ upvote: ..., postText: "...", userName: ..., comments: "..." }`; put all the answer-objects into an array, and the array is the `answers` field of the final result.

### Example: Retrieve all answers

* **HTML preview** Below is what an answer looks like. In the page there are several answers of the same structure.

![array-capture-preview](/assets/array-capture-preview.jpg)

```HTML
<!-- DOM struture of one answer div -->
<div id="answer-192502" class="answer" ...>
  <table>
    <tr>
      <td class="votecell"> BLUE PART </td>
      <td class="answercell">
        <div class="post-text" itemprop="text"> GREEN PART </div>
        ...
          <div class="user-info"> RED PART </div>
        ...
      </td>
    </tr> <!-- other trs -->
    <tr>
      ...
        <div id="comments-292502" class="comments"> PURPLE PART </div>
      ...
    </tr>
  </table>
</div>
```

```JavaScript
const cssSelectorForAnswerDiv      = '.answer'
const cssSelectorForAnswerVotecell = '.answer .votecell .vote-count-pos'
const cssSelectorForAnswerPostText = '.answer .post-text'
const cssSelectorForAnswerUserInfo = '.answer .user-info .user-info>a'
const cssSelectorForAnswerComments = '.answer .comments'

const temmeSelector = `
  .answer@answers (
    .votecell .vote-count-post{$upvote};
    .post-text{$postText};
    .user-info .user-info >a{$userName};
    .comments{$comments};
  )
`
// output of temme(html, temmeSelector):
const output = {
  answers: [
    {
      upvote: "253",
      postText: "Actually, a concise ...",
      userName: "CommunityRizwan Kassim",
      comments: "comments texts...",
    },
    {
      upvote: "108",
      postText: "Essentially, the onl ...",
      userName: "Brian",
      comments: "comments texts...",
    },
    // more answers ......
  ],
}
```

If all we want is the answers array, we can make a default array-capture. Change the selector to `'.answer@ ( /* children selectors */ )'` , and the output will just be an array of answer objects.

## Nested Array-Captures

Array-capture can be nested. Just place a array-capture within another array-capture. For example, in [stackoverflow question][], one question has several answers and each answer has several comments. We could use the following temme selector to capture an array of arrays of comments.

![array-of-arrays](/assets/array-of-arrays.jpg)

## Parent Reference `&`

`&` gives us the ability to capture data in the parent node. It has the same semantic meaning as in sass, less or stylus. Parent-reference is useful in array-capture when the data is stored in the parent node. In the following example, `&` refers to answer-div and `&[data-answerid=$answerId]` captures the answer ID into `.answerId`.

```javascript
const temmeSelector = `
  .answer@answers (
    &[data-answerid=$answerId];
    .votecell .vote-count-post{$upvote};
    .post-text{$postText};
    .user-info .user-info >a{$userName};
    .comments{$comments};
  )
`
```

## Multiple Selectors at Top-Level

Like we can use multiple selectors as children of parent selector, we can use multiple selectors at top level. Remember to use comma or semicolon as the separator. For example, the first basic example could also be written as:

```javascript
// one-selector-version: `#question-header .question-hyperlink[href=$url]{$title}`
const multipleSelectorsVersion = `
  #question-header .question-hyperlink[href=$url];
  #question-header .question-hyperlink{$title};
`
```

## Assignment Syntax

```javascript
const temmeSelector = `
  $foo = 'bar'; // in top-level
  div.foo{$a = null}; // in content capture
  li@list (
    $x = 123; // in children selector
  );
`
```

Assignment is a statement like `$foo = bar` where `foo` is an identifier and `bar` is a JavaScript literal (string/number/null/boolean, except RegExp). Assignment could appears in three places:

1. In top-level, `$foo = 'bar'` means that string `'bar'` will be in `.foo` of the final result;
2. In content-capture, `div.foo{ $a = null }` means that if we have such a div, then we put `null` in field `a`.
3. In children selector, `li@list ( $x = 123 )` means that every object in `list` will have `123` as the `x` field.

## JavaScript Style Comments

Temme selector supports both single line comments `// ......` and multi-line comments `/* ...... */`.

## Capture Filters (post-processors) `|`

When a value is captured, it is always a string. A filter is a simple function that receive input as *this context* with several arguments, and returns a single value. You could use filters to process the captured value.

### Filter Syntax

* No arguments: `li.good{$x|foo}` or `li.good{$x|foo()}`

  In the above, `foo` is the filter function. Every time variable `x` is captured, it will be processed as `x = foo.apply(x)`.

* With arguments: `div.bad{$x|foo(1,false)}`

  In the above, `foo` is the filter function. Every time variable `x` is captured, it will be processed as `x=foo.apply(x, [1, false])`;

* Chained filters: `div.hello{$x|foo|bar(0, 20)}`

  Filters are easy to be chained. In the above example, the value will first be processed by filter `foo` and then by filter `bar`. The value is processed like `x = foo.apply(x); x = bar.apply(x, [0, 20]);`.

Note that filter arguments could only be JavaScript literals (string/number/null/boolean, except RegExp).

### Built-in filters (TODO)

Temme provide some common filters. In file `src/filters.ts`, `defaultFilterMap` defines the built-in filters.

Built-in filters could be divided into three categories:

1. Structure Manipulation Filters: this category includes `pack`, `flatten`, `compact`, `words`, `lines`. // TODO a short description for  pack, compact, flatten, lines, words
2. Type Coercion Filters: this category includes `String`, `Number`, `Date`, `Boolean`. These filters converts the captured value to specific type;
3. Prototype Filters: we can use methods on value's prototype chain as filters. For example, if we can ensure that `x` is always a string, then we can safely use `$x|substring(0, 20)` or `$x|toUpperCase`.

### Defining customized filters

Use `defineFilter` to add customized global filters. Or provide a customized filter map as the third parameter of function `temme`.

## Content functions(outdated)

By default, `li{$name}` will capture <u>the text content</u> of element into field name. We could use other content functions to capture <u>html</u> or <u>cheerio-node</u>.

Four supported content functions

1. `text($var)` captures the text content of element (default behavior).
2. `html($var)` captures the inner-html of element
3. `node($var)` captures the cheerio-node.
4. `contains('xxx')` checks whether text content contains 'xxx' and captures nothing. If the check fails, the element does not match the selector(Just like the element does not match normal CSS selector).

### Multiple content functions(outdated)

We could use multiple content functions in a single curly brace. For example,

`p{contains('hello'), html($h), node($n), text($t)}`

This selectors means: check the text content of element contains string 'hello'. If  it does, capture the inner-html of element into field `h`, capture the cheerio-node into field `n`, and capture the text content of element into field `t`.

### Text matching(outdated)

`text` content function supports string matching. `text` accepts multiple parameters. Each parameter is either a value-capture form or a string literal. It will try to match the text content against multiple parameters. If match succeed, the result will be an object in which field `foo` is the corresponding text  content in the whole text. (TODO text matching description should be more specific.)

For example:

`text($name, ':', $value)`

The above selector will match `<p>foo: hello world!</p>`. And the result is:

```json
{
  "name": "foo",
  "value": " hello world!"
}
```

[stackoverflow question]: https://stackoverflow.com/questions/291978/short-description-of-the-scoping-rules
