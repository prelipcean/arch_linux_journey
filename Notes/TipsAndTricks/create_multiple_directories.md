Let's take as example below project structure:

```
Websites/
  static/
      cs
      js
  templates/
      html and xhtml
```

You can do it in a single command like this:

```
mkdir -p Website/{static/{cs,js},templates/html\ and\ xhtml}
```

Be careful to escape the spaces in your directory names.