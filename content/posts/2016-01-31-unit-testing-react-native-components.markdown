---
layout: post
title: Unit Testing React Native Components
date: 2016-01-31T09:36:05+00:00
---

tldr: [github.com/hosainnet/RNUnitTests](https://github.com/hosainnet/RNUnitTests)

I've recently jumped ship to React/React Native (and JS in general) and one of the first hurdles for me was figuring out how to unit test RN component code. 

As many of us starting out, you'll have probably come across the [snowflake](https://github.com/bartonhammond/snowflake) project for guidance and inspiration (which deserves credit for most of the content in this post). However since it's a big project with a large set of dependencies and files, I wanted to go through the minimal setup needed to go from `react-native init AwesomeProject` to being able to run `npm test` and get some results back.

## The concept

We can easily test RN views by mocking its built-in components/modules with a ***React*** equivalent, and then use existing JS/React testing frameworks to test our RN code (one of the popular ones is [jest](https://www.npmjs.com/package/jest-cli)). Then using a "shallow renderer" we're able to traverse our view as an object and verify that it matches our expectation. 

As well as testing output, the shallow renderer gives us an instance of our component which we can use to invoke and test JS functions as explained below. 

I'll take you through an example based on a modified version of the movie listing app from the official RN docs.

## Setup

## package.json

First of all we need to include some dev dependencies: `babel-jest` (JS compiler), the `jest-cli` and `react-addons-test-utils` (shallow renderer). We've also defined our `npm test` command to execute our tests with the configurations in the `jest` block

`package.json`
 
```json
{
  //...
  "scripts": {
    //...
    "test": "rm -rf ./node_modules/jest-cli/.haste_cache && jest"
  },
  "jest": {
    "scriptPreprocessor": "<rootDir>/node_modules/babel-jest",
    "testFileExtensions": [
      "es6",
      "js"
    ],
    "moduleFileExtensions": [
      "js",
      "json",
      "es6"
    ],
    "unmockedModulePathPatterns": [
      "<rootDir>/node_modules/react",
      "<rootDir>/node_modules/fbjs"
    ],
    "verbose": true,
    "collectCoverage": true
  },
  "dependencies": {
    //...
    "react-native": "^0.18.1"
  },
  "devDependencies": {
    "babel-jest": "^6.0.1",
    "jest-cli": "^0.8.2",
    "react-addons-test-utils": "^0.14.6"
  }
}
```

## `.babel.rc`

Add this file to the root directory for configuring Babel

```json
{
    "extends": "react-native/packager/react-packager/.babelrc"
}
```

### Folder structure

```
├── js
│   ├── __mocks__
│   │   └── react-native.js
│   ├── __tests__
│   │   └── view
│   │       └── MoviesView-test.js
│   ├── service
│   │   └── DataService.js
│   └── view
│       └── MoviesView.js
{% endcodeblock %}
```

#### `__mocks__/react-native.js`

This is the key file that jest will use to replace all the ReactNative components with dummy React views or simple objects.

You'll find yourself coming to this file quite often as you add more code to your views. Basically any time you get a 'cannot called x on undefined' when running tests, you've likely added some new component code in your view but it was not mocked here.


```js

const React = require('react');
const ReactNative = React;

ReactNative.StyleSheet = {
    create: function create(styles) {
        return styles;
    }
};

class View extends React.Component {
    render() { return false; }
}

class ListView extends React.Component {
    static DataSource() {
    }
}

class AppRegistry {
    static registerComponent () {
    }
}

ReactNative.View = View;
ReactNative.ScrollView = View;
ReactNative.ListView = ListView;
ReactNative.Text = View;
ReactNative.TouchableOpacity = View;
ReactNative.TouchableHighlight = View;
ReactNative.TouchableWithoutFeedback = View;
ReactNative.ToolbarAndroid = View;
ReactNative.Image = View;
ReactNative.AppRegistry = AppRegistry;

module.exports = ReactNative;
```


#### `__tests__`

For jest to pick up test files, they need to be in a `__tests__` folder. You can have many of these folders sitting by your components but I've decided to just have one high level one to separate the test code.

## Testing rendered output

Consider we have the following render code:

`js/view/MoviesView.js`

```js

render() {
    if (!this.state.loaded) {
        return this.renderLoadingView();
    }

    return (
        <ListView
            dataSource={this.state.dataSource}
            renderRow={this.renderMovie}
            style={styles.listView}
        />
    );
}

renderLoadingView() {
    return (
        <View style={styles.container}>
            <Text>
                Loading movies...
            </Text>
        </View>
    );
}
```

render() will return a different view based on the boolean state value of `loaded`, so we want to be able to write two different tests to cover both branches, here is what it looks like:


`js/__tests__/view/MovieView-test.js`

```js
const React = require('react-native');
const { View } = React;

const utils = require('react-addons-test-utils');

jest.dontMock('../../view/MoviesView');
var MoviesView = require('../../view/MoviesView');

describe('MovieView', () => {
    let moviesView;

    function renderScreen(props, states) {
        const renderer = utils.createRenderer();
        renderer.render(<MoviesView {...props || {}}/>);
        const instance = renderer._instance._instance;
        instance.setState(states || {});
        const output = renderer.getRenderOutput();

        return {
            output,
            instance
        };
    }
    
    it('should display the loading view if data was not loaded', () => {
        moviesView = renderScreen();
        const {output} = moviesView;
        expect(output.type).toEqual(View);
        expect(output.props.children.props.children).toBe("Loading movies...");
    });

    it('should display the list view if data was loaded', () => {
        const states = {loaded: true, dataSource: {test: "test"}};
        moviesView = renderScreen({}, states);
        const {output} = moviesView;
        expect(output.type.name).toBe("ListView");
        expect(output.props.dataSource).toEqual(states.dataSource);
    });

});
```

First, we define a `renderScreen` helper function, which allows us to render a component using the shallow renderer and it returns an output and a component instance. You'll notice the `props` and `states` parameters which are passed to the renderer, meaning we're able render the component in a pre-defined set of.. states and props! 
 

## Testing JS code in React Native components

Now the other part of our component deals with requesting data from `DataService.js`. Consider the following:


```js
//...
let DataService;

class MoviesView extends Component {
    constructor(props) {
        super(props);
        //...

        DataService = props.dataService ? props.dataService : require("../service/DataService");
    }

    componentDidMount() {
        DataService.fetchData(this.onDataResponse);
    }
    
    onDataResponse(responseData) {
        //...
    }

    //...
```

To test that `componentDidMount` makes a call to DataService::

```js

describe('MovieView', () => {
    let moviesView;

    const initialProps = {
        dataService: {fetchData: jest.genMockFn()}
    };

    //renderScreen...

    it('should fetch data on componentDidMount', () => {
        moviesView = renderScreen(initialProps);
        const {instance} = moviesView;
        instance.componentDidMount();
        expect(initialProps.dataService.fetchData.mock.calls.length).toBe(1);
        expect(initialProps.dataService.fetchData.mock.calls[0][0]).toBe(instance.onDataResponse);
    });
})
```

Here, we pass MovieView a custom DataService prop, render the screen and extract `instance`, manually invoke `componentDidMount` on it and assert that our mock implementation of `fetchData` was invoked with the correct parameters.

## Running the tests

`npm test` should now run and give you some test results!

![Android project structure](/images/blog/2016/rn-unit-tests.png)


Working example: [github.com/hosainnet/RNUnitTests](https://github.com/hosainnet/RNUnitTests)