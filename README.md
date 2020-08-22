### Features
- Fields can be of type:
  - simple (string, number, bool, date/time/datetime, list)
  - structs (will be displayed in selectbox as tree)
  - custom type (dev should add its own widget component in config for this)
- Comparison operators can be:
  - binary (== != < > ..)
  - unary (is empty, is null)
  - 'between' (for numbers, dates, times)
  - complex operators like 'proximity'
- Values of fields can be compared with:
  - values
  - another fields (of same type)
  - function (arguments also can be values/fields/funcs)
- Reordering (drag-n-drop) support for rules and groups of rules
## Getting started
Install: `npm i react-awesome-query-builder`  
See [basic usage](#usage) and [API](#api) below.  


## v2 Migration
From v2.0 of this lib AntDesign is now optional (peer) dependency, so you need to explicitly include `antd` (4.x) in `package.json` of your project if you want to use AntDesign UI.  
Please import `AntdConfig` from `react-awesome-query-builder/lib/config/antd` and use it as base for your config (see below in [usage](#usage)).  
Alternatively you can use `BasicConfig` for simple vanilla UI, which is by default.  
Support of other UI frameworks (like Bootstrap, Material UI) are planned for future, see [Other UI frameworks](#other-ui-frameworks).


## Usage
```javascript
import React, {Component} from 'react';
import {Query, Builder, BasicConfig, Utils as QbUtils} from 'react-awesome-query-builder';
import AntdConfig from 'react-awesome-query-builder/lib/config/antd';
import 'react-awesome-query-builder/css/antd.less'; // or import "antd/dist/antd.css";
import 'react-awesome-query-builder/lib/css/styles.css';
import 'react-awesome-query-builder/lib/css/compact_styles.css'; //optional, for more compact styles
const InitialConfig = AntdConfig; // or BasicConfig

// You need to provide your own config. See below 'Config format'
const config = {
  ...InitialConfig,
  fields: {
    qty: {
        label: 'Qty',
        type: 'number',
        fieldSettings: {
            min: 0,
        },
        valueSources: ['value'],
        preferWidgets: ['number'],
    },
    price: {
        label: 'Price',
        type: 'number',
        valueSources: ['value'],
        fieldSettings: {
            min: 10,
            max: 100,
        },
        preferWidgets: ['slider', 'rangeslider'],
    },
    color: {
        label: 'Color',
        type: 'select',
        valueSources: ['value'],
        fieldSettings: {
          listValues: [
            { value: 'yellow', title: 'Yellow' },
            { value: 'green', title: 'Green' },
            { value: 'orange', title: 'Orange' }
          ],
        }
    },
    is_promotion: {
        label: 'Promo?',
        type: 'boolean',
        operators: ['equal'],
        valueSources: ['value'],
    },
  }
};

// You can load query value from your backend storage (for saving see `Query.onChange()`)
const queryValue = {"id": QbUtils.uuid(), "type": "group"};


class DemoQueryBuilder extends Component {
    state = {
      tree: QbUtils.checkTree(QbUtils.loadTree(queryValue), config),
      config: config
    };
    
    render = () => (
      <div>
        <Query
            {...config} 
            value={this.state.tree}
            onChange={this.onChange}
            renderBuilder={this.renderBuilder}
        />
        {this.renderResult(this.state)}
      </div>
    )

    renderBuilder = (props) => (
      <div className="query-builder-container" style={{padding: '10px'}}>
        <div className="query-builder qb-lite">
            <Builder {...props} />
        </div>
      </div>
    )

    renderResult = ({tree: immutableTree, config}) => (
      <div className="query-builder-result">
          <div>Query string: <pre>{JSON.stringify(QbUtils.queryString(immutableTree, config))}</pre></div>
          <div>MongoDb query: <pre>{JSON.stringify(QbUtils.mongodbFormat(immutableTree, config))}</pre></div>
          <div>SQL where: <pre>{JSON.stringify(QbUtils.sqlFormat(immutableTree, config))}</pre></div>
          <div>JsonLogic: <pre>{JSON.stringify(QbUtils.jsonLogicFormat(immutableTree, config))}</pre></div>
      </div>
    )
    
    onChange = (immutableTree, config) => {
      // Tip: for better performance you can apply `throttle` - see `examples/demo`
      this.setState({tree: immutableTree, config: config});

      const jsonTree = QbUtils.getTree(immutableTree);
      console.log(jsonTree);
      // `jsonTree` can be saved to backend, and later loaded to `queryValue`
    }
}
```


## API

### `<Query />`
Props:
- `onChange` - callback when value changed. Params: `value` (in Immutable format), `config`.
- `renderBuilder` - function to render query builder itself. Takes 1 param `props` you need to pass into `<Builder {...props} />`.

*Notes*:
  - use prop `disableEnforceFocus={true}` for dialog or popver
  - set css `.MuiPopover-root, .MuiDialog-root { z-index: 900 !important; }` (or 1000 for AntDesign v3)

### `<Builder />`
Render this component only inside `Query.renderBuilder()` like in example above:
```js
  renderBuilder = (props) => (
    <div className="query-builder-container">
      <div className="query-builder qb-lite">
          <Builder {...props} />
      </div>
    </div>
  )
```
Wrapping `<Builder />` in `div.query-builder` is necessary.  
Optionally you can add class `.qb-lite` to it for showing action buttons (like delete rule/group, add, etc.) only on hover, which will look cleaner.  
Wrapping in `div.query-builder-container` is necessary if you put query builder inside scrollable block.  

### `Utils`
- Save, load:
  #### getTree (immutableValue) -> Object
  Convert query value from internal Immutable format to JS format. 
  You can use it to save value on backend in `onChange` callback of `<Query>`.
  #### loadTree (jsValue, config) -> Immutable
  Convert query value from JS format to internal Immutable format. 
  You can use it to load saved value from backend and pass as `value` prop to `<Query>` (don't forget to also apply `checkTree()`).
  #### checkTree (immutableValue, config) -> Immutable
  Validate query value corresponding to config. 
  Invalid parts of query (eg. if field was removed from config) will be always deleted. 
  Invalid values (values not passing `validateValue` in config, bad ranges) will be deleted if `showErrorMessage` is false OR marked with errors if `showErrorMessage` is true.
  #### isValidTree (immutableValue) -> Boolean
  If `showErrorMessage` in config.settings is true, use this method to check is query has bad values.
- Export:
  #### queryString (immutableValue, config, isForDisplay) -> String
  Convert query value to custom string representation. `isForDisplay` = true can be used to make string more "human readable".
  #### mongodbFormat (immutableValue, config) -> Object
  Convert query value to MongoDb query object.
  #### sqlFormat (immutableValue, config) -> String
  Convert query value to SQL where string.
  #### jsonLogicFormat (immutableValue, config) -> {logic, data, errors}

## Development
To build the component locally, clone this repo then run:  
`npm install`  
`npm start`  
Then open localhost:3001 in a browser.

Scripts:
- `npm test` - Run tests with Karma and update coverage. Recommended before commits.
- `npm run lint` - Run ESLint. Recommended before commits.
- `npm run lint-fix` - Run ESLint with `--fix` option. Recommended before commits.
- `npm run build-examples` - Build examples with webpack. Output path: `examples`
- `npm run build-npm` - Build npm module that can be published. Output path: `lib`

Feel free to open PR to add new reusable types/widgets/operators (eg., regex operator for string, IP type & widget).  
Pull Requests are always welcomed :)




