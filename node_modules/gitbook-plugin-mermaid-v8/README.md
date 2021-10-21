## Mermaid plugin for GitBook

Plugin for [GitBook](https://github.com/GitbookIO/gitbook) 3 which renders [Mermaid](https://github.com/knsv/mermaid) diagrams and flow charts detected in the book markdown.

### How to install it?

You can use install via **NPM**:

```
$ npm install gitbook-plugin-mermaid-v8
```

And use it for your book with in the book.json:

```
{
  "plugins": ["mermaid-v8"]
}
```

### How to use it?

Just put the code into fenced code block and tag it **mermaid** key word like this:

    ```mermaid
    graph TD;
      A-->B;
      A-->C;
      B-->D;
      C-->D;
    ```
