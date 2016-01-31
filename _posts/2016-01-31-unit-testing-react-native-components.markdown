---
layout: post
title: Unit Testing React Native Components
date: 2016-01-31T09:36:05+00:00
---

I've recently jumped ship to React/React Native (and JS in general) and one of the first hurdles for me was figuring out how unit test RN component code. 

As many of us starting out, you'll have probably come across the [snowflake](https://github.com/bartonhammond/snowflake) for guidance and inspiration (which deserves credit for most of the content in this post). However since it's a big project with a large set of dependencies and files, I wanted to go through the minimal setup needed to go from `react-native init AwesomeProject` to being able to run `npm test` and get some reports back.

## The concept

The way we can easily test RN views is simply by mocking its built-in components/modules with a ***React*** equivalent, and then use existing JS/React testing frameworks to test our RN code (one of the popular ones is [jest](https://www.npmjs.com/package/jest-cli)). Then using a "shallow renderer" we're able to traverse our view as an object and verify that it matches our expectation. More importantly, the shallow renderer gives us an instance of our component which we can use to invoke and test JS functions within our components as explained below. 

I'll take you through an example based on a modified version of the movie listing app from the official RN docs.

## Setup

## package.json

First of all we need to include some dev dependencies: `babel-jest` (JS compiler), the `jest-cli` and `react-addons-test-utils` (shallow renderer). We've also defined our `npm test` command to execute our tests with the configurations defined in the `jest` block

`package.json`
```json
{
  .
  .
  "scripts": {
    .
    .
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
    .
    .
    "react-native": "^0.18.1"
  },
  "devDependencies": {
    "babel-jest": "^6.0.1",
    "jest-cli": "^0.8.2",
    "react-addons-test-utils": "^0.14.6"
  }
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
```

#### `___mocks___`

This contains the key file that jest will use to replace all the ReactNative components with dummy React views or simple objects.

You'll find yourself coming to this file quite often as you add more code to your views. Basically any time you get a 'cannot called x on undefined' when running tests, you've likely added some new component code in your view but it was not mocked here.


`__mocks__/react-native.js`
```
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

#### `___tests___`

For jest to pick up test files, they need to be in a `___tests___` folder. You can have many of these folders sitting by your components but I've decided to just have one high level one to separate the test code.

## Testing rendered output

Consider we have the following render code:

`js/view/MoviesView.js`
```
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

render() will return a different view based on the state boolean value of `loaded`, so we want to be able to write two different tests to cover both branches, here what it looks like:


js/__tests__/view/MovieView-test.js

```
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

First, we define a `renderScreen` helper function, which allows us to render a component using the shallow renderer and it returns us the output and a component instance. You'll notice the `props` and `states` parameters which are passed to the renderer, meaning we're able render the component in a pre-defined set of.. states and props! 
 

## Testing JS code in React Native components

Now the other part of our component deals with requesting data from `DataService.js`. Consider the following:

```

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

To test that `componentDidMount` makes a call to DataService, we can do the following:

```
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
```

Here, we provide a custom DataService prop on MovieView, render the screen and extract `instance`, manually invoke `componentDidMount` on it and assert that our mock implementation of `fetchData` was invoked with the correct parameters.


 
 




