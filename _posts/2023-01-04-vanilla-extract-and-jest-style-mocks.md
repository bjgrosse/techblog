---
title: "Vanilla Extract CSS and Jest style mocks"
date: 2023-01-04
---

# Problem
We are integrating Vanilla-extract CSS in a React project which uses Jest for our tests. Like many Jest configurations, we have this line in our`jest.config.ts` `moduleNameMapper` section:

```
'\\.(css|less|scss)$': '<rootDir>/src/Mocks/styleMock.js'
```

Without this, we get Jest transform errors when it tries to process imports of css files. However, this causes a problem with Vanilla Extract, as this mechanism can't distinguish between plain css imports and our vanilla extract css.ts file imports. The result is that our vanilla extract css.ts files get mocked and our code breaks where vanilla extract styles are imported from those files. 

The VE documentation recommends removing the style mock completely, or trying moving all your plain CSS files into a folder so you can update the pattern to match only those and ignore the css.ts files. This solution isn't feasible for us because we import some css files from packages and those CSS files can't be moved to a special folder. 

# Solution
We solved this problem by moving the CSS mocking login from the import processing phase (via `moduleNameMapper`) to the transform phase like this:

1. Update the `moduleNameMapper` style mock to no longer mock css files:
```
'\\.(less|scss)$': '<rootDir>/src/Mocks/styleMock.js'
```

2. Add a custom transform for css files in our `jest.config.ts`:

```
transform: {
    '\\.css(.ts)?$': '<rootDir>/cssTransformer.js',
    '^.+\\.(js|jsx|ts|tsx)$': 'babel-jest',
},
```

3. Create the custom transformer that properly transforms css.ts files but returns a mock value for all the rest.

```
const path = require('path')
const transformer = require('@vanilla-extract/jest-transform')

module.exports = {
    process(sourceText, sourcePath, options) {
        // if this is a vanilla-extract css.ts file then pass it to the vanilla-extract transformer
        if (sourcePath.endsWith('.css.ts')) {
            return transformer.default.process(sourceText, sourcePath, { config: options })
        }
        // otherwise, we just return a simple string that exports the filename because
        // we don't want any CSS being transformed in Jest
        return {
            code: `module.exports = ${JSON.stringify(path.basename(sourcePath))};`,
        }
    },
}

```

4. Update the `jest.config.js` `transformIgnorePatterns` to not ignore css files coming from node_modules, so that these CSS files get processed via our transformer.

```transformIgnorePatterns: ['[/\\\\]node_modules[/\\\\].+\\.(?!css)$']```

That's it. That worked for us without requiring any code refactoring. 
