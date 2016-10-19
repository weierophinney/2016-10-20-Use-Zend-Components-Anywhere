# Use Zend Components Anywhere

This is a slide deck developed for ZendCon 2016. It was developed using
reveal.js.

## Running the slides

You can run the slide deck in a couple ways:

- Via npm or yarn
- Via php

### npm/yarn

First, install dependencies.

```bash
$ sudo npm install -g grunt # Necessary to fire up the server
$ yarn install              # Preferred, as there's a lockfile
$ npm install               # Also works
```

Once installed:

```bash
$ npm start
```

This will fire up the server, and open the slides in your preferred browser.

### php

```bash
$ php -S 0.0.0.0:8000 -t .
```

Once you've dont that, open the slides at http://localhost:8000

## Notes

There are slide notes available. Press "s" in the browser to spawn the presenter
view.

Additionally, the full narrative is available in the `markdown/outline.md` file.

## License

[CC BY-NC-SA](https://creativecommons.org/licenses/by-nc-sa/4.0/)
