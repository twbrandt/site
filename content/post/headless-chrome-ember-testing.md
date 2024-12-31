---
title:  "Using headless Chrome for Ember testing"
date: 2017-06-26T15:41:21-04:00
expiryDate: 2024-03-26
draft: false
categories: ["ember", "software engineering"]
---

As of Chrome v59, you can run it in a headless environment, which means you don't need PhantomJS for running tests anymore. Using it is the easiest thing in the world:

{{< highlight javascript>}}

// testem.js

/*jshint node:true*/
module.exports = {
  "framework": "qunit",
  "test_page": "tests/index.html?hidepassed",
  "disable_watching": true,
  "launch_in_ci": [
    "Chrome"
  ],
  "browser_args": {
    'Chrome': [ '--headless', '--disable-gpu', '--remote-debugging-port=9222' ],
  },
  "launch_in_dev": [
    "Chrome"
  ]
};
{{< /highlight >}}

If you have a `testem.json` file, delete it.

Then:
```
ember test --server
```

That's it!
