# Use Zend Components Anywhere

This is a slide deck developed for ZendCon 2016. It was developed using
reveal.js.

## Running the slides

In order to run the slides, you will need to use npm/grunt, as the slides were
written using the markdown features of reveal.js, which require a node-based
server.

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

## Notes

There are slide notes available. Press "s" in the browser to spawn the presenter
view.

Additionally, the full narrative is available in the `markdown/outline.md` file.

## License

[CC BY-NC-SA](https://creativecommons.org/licenses/by-nc-sa/4.0/)
