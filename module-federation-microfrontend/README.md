# Microfrontend with module federation

This pattern allows you easily build microfrontend with module federation and share components in multiple projects.


## Project setup

We are building two applications: host and remote.

The host app is the “main” app and remote is a sub-app plugging into it.

In your root directory run below commands:

```sh
npx create-react-app host
npx create-react-app remote
```

Within each host/ and remote/ run:

```sh
npm install --save-dev webpack webpack-cli html-webpack-plugin webpack-dev-server babel-loader
```

> Webpack Module Federation is only available in version 5 and above of webpack.

## Host App

We are going to start with our webpack configuration

Create a new webpack.config.js file at the root of host/:

```js
// host/webpack.config.js
const HtmlWebpackPlugin = require("html-webpack-plugin");

module.exports = {
  entry: "./src/index",
  mode: "development",
  devServer: {
    port: 3000,
  },
  module: {
    rules: [
      {
        test: /\.(js|jsx)?$/,
        exclude: /node_modules/,
        use: [
          {
            loader: "babel-loader",
            options: {
              presets: ["@babel/preset-env", "@babel/preset-react"],
            },
          },
        ],
      },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: "./public/index.html",
    }),
  ],
  resolve: {
    extensions: [".js", ".jsx"],
  },
  target: "web",
};
```

**Update package.json scripts in both /host and /remote**

```json
"scripts":{
  "start": "webpack serve"
}
```

First, we need the index.js entry to our app. We are importing another file bootstrap.js that renders the React app.

```js
// host/src/index.js
import("./bootstrap");
```

Next, we define the bootstrap.js file that renders our React application.

```js
// host/src/bootstrap.js
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

Now we are ready to write our App.js file where the app’s main logic happens. Here we will load two components from remote which we will define later.

import("Remote/App") will dynamically fetch the Remote app’s App.js React component.

```js
// host/src/App.js
import React from "react";
import ErrorBoundary from "./ErrorBoundary";
const RemoteApp = React.lazy(() => import("Remote/App"));
const RemoteButton = React.lazy(() => import("Remote/Button"));

const RemoteWrapper = ({ children }) => (
  <div
    style={{
      border: "1px solid red",
      background: "white",
    }}
  >
    <ErrorBoundary>{children}</ErrorBoundary>
  </div>
);

export const App = () => (
  <div style={{ background: "rgba(43, 192, 219, 0.3)" }}>
    <h1>This is the Host!</h1>
    <h2>Remote App:</h2>
    <RemoteWrapper>
      <RemoteApp />
    </RemoteWrapper>
    <h2>Remote Button:</h2>
    <RemoteWrapper>
      <RemoteButton />
    </RemoteWrapper>
    <br />
    <a href="http://localhost:4000">Link to Remote App</a>
  </div>
);
export default App;
```

Create file `ErrorBoundary.jsx` and paste below code:

```jsx
// host/src/ErrorBoundary.jsx

import * as React from 'react';

class ErrorBoundary extends React.Component {
    constructor(props) {
      super(props);
      this.state = { hasError: false };
    }
  
    static getDerivedStateFromError(error) {
      // Update state so the next render will show the fallback UI.
      return { hasError: true };
    }
  
    componentDidCatch(error, errorInfo) {
      // You can also log the error to an error reporting service
      console.log(error, errorInfo);
    }
  
    render() {
      if (this.state.hasError) {
        // You can render any custom fallback UI
        return <h1>Something went wrong.</h1>;
      }
  
      return this.props.children; 
    }
  }

export default ErrorBoundary;
```

In our webpack.config.js we introduce the ModuleFederationPlugin:

```js
// host/webpack.config.js
const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");
const { dependencies } = require("./package.json");

module.exports = {
  //...
  plugins: [
    new ModuleFederationPlugin({
      name: "Host",
      remotes: {
        Remote: `Remote@http://localhost:4000/moduleEntry.js`,
      },
      shared: {
        ...dependencies,
        react: {
          singleton: true,
          requiredVersion: dependencies["react"],
        },
        "react-dom": {
          singleton: true,
          requiredVersion: dependencies["react-dom"],
        },
      },
    }),
    //...
  ],
  //...
};
```

Important things to note:

* **name:** is used to distinguish the modules. It is not as important here because we are not exposing anything, but it is vital in the Remote app.

* **remotes:** is where we define the federated modules we want to consume in this app. You’ll notice we specify Remote as the internal name so we can load the components using import("Remote/<component>"). But we also define the location where the remote’s module definition is hosted: Remote@http://localhost:4000/moduleEntry.js. This URL tells us three important things. The module’s name is Remote, it is hosted on localhost:4000, and its module definition is moduleEntry.js.

* **shared:** is how we share dependencies between modules. This is very important for React because it has a global state, meaning you should only ever run one instance of React and ReactDOM in any given app. To achieve this in our architecture, we are telling webpack to treat React and ReactDOM as singletons, so the first version loaded from any modules is used for the entire app. As long as it satisfies the requiredVersion we define. We are also importing all of our other dependencies from package.json and including them here, so we minimize the number of duplicate dependencies between our modules.

Now, if we run npm start in the host app we should see something like:

![image](https://user-images.githubusercontent.com/28493237/218248376-d81413a7-5d3b-44f3-8f8f-a3f333b654ff.png)

This means our host app is configured, but our remote app is not exposing anything yet. So we need to configure that next.


## Remote App

Let’s start with the webpack config. Since we now have some knowledge of Module Federation, let’s use it from the get-go:

```js
// remote/webpack.config.js
const HtmlWebpackPlugin = require("html-webpack-plugin");
const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");
const path = require("path");
const { dependencies } = require("./package.json");

module.exports = {
  entry: "./src/index",
  mode: "development",
  devServer: {
    static: {
      directory: path.join(__dirname, "public"),
    },
    port: 4000,
  },
  module: {
    rules: [
      {
        test: /\.(js|jsx)?$/,
        exclude: /node_modules/,
        use: [
          {
            loader: "babel-loader",
            options: {
              presets: ["@babel/preset-env", "@babel/preset-react"],
            },
          },
        ],
      },
    ],
  },
  plugins: [
    new ModuleFederationPlugin({
      name: "Remote",
      filename: "moduleEntry.js",
      exposes: {
        "./App": "./src/App",
        "./Button": "./src/Button",
      },
      shared: {
        ...dependencies,
        react: {
          singleton: true,
          requiredVersion: dependencies["react"],
        },
        "react-dom": {
          singleton: true,
          requiredVersion: dependencies["react-dom"],
        },
      },
    }),
    new HtmlWebpackPlugin({
      template: "./public/index.html",
    }),
  ],
  resolve: {
    extensions: [".js", ".jsx"],
  },
  target: "web",
};
```

The important things to note are:

* Our webpack dev server runs at `localhost:4000`
* The remote module’s name is `Remote`
* The filename is `moduleEntry.js`

Combining these will allow our host to find the remote code at `Remote@http://localhost:4000/moduleEntry.js`.

`exposes` is where we define the code we want to share in the moduleEntry.js file. Here we are exposing two: `<App />` and `<Button />`.

Similar to the host app, we need a dynamic import in our webpack entry.

```js
// /remote/src/index.js
import("./bootstrap");
```

```js
// remote/src/bootstrap.js
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

```js
// remote/src/App.js
import React from "react";

export const App = () => {
  return <div>Hello from the other side</div>;
};
export default App;
```

```js
// remote/src/Button.js
import React from "react";

export const Button = () => <button>Hello!</button>;

export default Button;
```

Now the Remote app is fully configured, and if you run `npm start` you should see a blank page with “Hello from the other side.”

Now if we run npm start in both the host/ and remote/ directories, we should see the Host app running on localhost:3000 and the remote app running on localhost:4000.

The host app would look something like this:

![image](https://user-images.githubusercontent.com/28493237/218248671-fa18c62a-26a4-4a87-956b-488c86eb7b25.png)
