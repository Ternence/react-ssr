Let’s have a closer look at the feature that allows you to build universal applications with React.

Server-Side Rendering — SSR from here on — is the ability of a front-end framework to render markup while running on a back-end system.

Applications that have the ability to render both on the server and on the client are called universal apps.

Why bother?
In order to understand why SSR is needed, we need to understand the evolution of web applications in the past 10 years.

This is tightly coupled with the rise of the Single Page Application — SPA from here on. SPAs offer great advantages in speed and UX over traditional server-rendered apps.

But there is a catch. The initial server request is generally returning an empty HTML file with a bunch of CSS and JavaScript (JS) links. Then the external files need to be fetched in order to render relevant markup.

This means that the user will have to wait longer for the initial render. This also means that crawlers may interpret your page as empty.

So the idea is to render your app on the server initially, then to leverage the capabilities of SPAs on the client.

SSR + SPA = Universal App*

*You will find the term isomorphic app in some articles — it’s the same thing.

Now the user does not have to wait for your JS to load and gets a fully rendered HTML as soon as the initial request returns a response.

Imagine the huge improvement for users navigating on slow 3G networks. Rather than waiting for over 20s for the website to load, you get content on their screen almost instantly.


And now, all the requests that are made to your server return fully rendered HTML. Great news for your SEO department!

Crawlers will now see your website as any other static site on the web and will index all the content you render on the server.

So to recap, the two main benefits we get from SSR are:

Faster times for the initial page render
Fully indexable HTML pages
Understanding SSR — one step at a time
Let’s take an iterative approach to build our complete SSR example. We start with React’s API for server rendering and we’ll add something to the mix at each step.

You can follow this repository and the tags defined there for each step.

Basic Setup
First things first. In order to use SSR, we need a server! We’ll use a simple Express app that will render our React app.

import express from "express";
import path from "path";

import React from "react";
import { renderToString } from "react-dom/server";
import Layout from "./components/Layout";

const app = express();

app.use( express.static( path.resolve( __dirname, "../dist" ) ) );

app.get( "/*", ( req, res ) => {
    const jsx = ( <Layout /> );
    const reactDom = renderToString( jsx );

    res.writeHead( 200, { "Content-Type": "text/html" } );
    res.end( htmlTemplate( reactDom ) );
} );

app.listen( 2048 );

function htmlTemplate( reactDom ) {
    return `
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="utf-8">
            <title>React SSR</title>
        </head>
        
        <body>
            <div id="app">${ reactDom }</div>
            <script src="./app.bundle.js"></script>
        </body>
        </html>
    `;
}
We need to tell Express to serve our static files from our output folder — line 10.

We create a route that handles all non-static incoming requests. This route will respond with the rendered HTML.

We use renderToString — lines 13–14 — to convert our starting JSX into a string that we insert in the HTML template.

As a note, we’re using the same Babel plugins for the client code and for the server code. So JSX and ES Modules work inside server.js.

The corresponding method on the client is now ReactDOM.hydrate . This function will use the server-rendered React app and will attach event handlers.

import ReactDOM from "react-dom";
import Layout from "./components/Layout";

const app = document.getElementById( "app" );
ReactDOM.hydrate( <Layout />, app );
To see the full example, check out the basic tag in the repository.

That’s it! You just created your first server-rendered React app!

React Router
We have to be honest here, the app doesn’t do much. So let’s add a few routes and see how we handle the server part.

import { Link, Switch, Route } from "react-router-dom";
import Home from "./Home";
import About from "./About";
import Contact from "./Contact";

export default class Layout extends React.Component {
    /* ... */

    render() {
        return (
            <div>
                <h1>{ this.state.title }</h1>
                <div>
                    <Link to="/">Home</Link>
                    <Link to="/about">About</Link>
                    <Link to="/contact">Contact</Link>
                </div>
                <Switch>
                    <Route path="/" exact component={ Home } />
                    <Route path="/about" exact component={ About } />
                    <Route path="/contact" exact component={ Contact } />
                </Switch>
            </div>
        );
    }
}
The Layout component now renders multiple routes on the client.

We need to mimic the Router setup on the server. Below you can see the main changes that should be done.

/* ... */
import { StaticRouter } from "react-router-dom";
/* ... */

app.get( "/*", ( req, res ) => {
    const context = { };
    const jsx = (
        <StaticRouter context={ context } location={ req.url }>
            <Layout />
        </StaticRouter>
    );
    const reactDom = renderToString( jsx );

    res.writeHead( 200, { "Content-Type": "text/html" } );
    res.end( htmlTemplate( reactDom ) );
} );

/* ... */
On the server, we need to wrap our React application in the StaticRouter component and provide the location.

As a side note, the context is used for tracking potential redirects while rendering the React DOM. This needs to be handled with a 3XX response from the server.

The full example can be seen on the router tag in the same repository.

Redux
Now that we have routing capabilities, let’s integrate Redux.

In the simple scenario, we need Redux to handle state management on the client. But what if we need to render parts of the DOM based on that state? It makes sense to initialize Redux on the server.

If your app is dispatching actions on the server, it needs to capture the state and send it over the wire together with the HTML. On the client, we feed that initial state into Redux.

Let’s have a look at the server first:

/* ... */
import { Provider as ReduxProvider } from "react-redux";
/* ... */

app.get( "/*", ( req, res ) => {
    const context = { };
    const store = createStore( );

    store.dispatch( initializeSession( ) );

    const jsx = (
        <ReduxProvider store={ store }>
            <StaticRouter context={ context } location={ req.url }>
                <Layout />
            </StaticRouter>
        </ReduxProvider>
    );
    const reactDom = renderToString( jsx );

    const reduxState = store.getState( );

    res.writeHead( 200, { "Content-Type": "text/html" } );
    res.end( htmlTemplate( reactDom, reduxState ) );
} );

app.listen( 2048 );

function htmlTemplate( reactDom, reduxState ) {
    return `
        /* ... */
        
        <div id="app">${ reactDom }</div>
        <script>
            window.REDUX_DATA = ${ JSON.stringify( reduxState ) }
        </script>
        <script src="./app.bundle.js"></script>
        
        /* ... */
    `;
}
It looks ugly, but we need to send the full JSON state together with our HTML.

Then we look at the client:

import React from "react";
import ReactDOM from "react-dom";
import { BrowserRouter as Router } from "react-router-dom";
import { Provider as ReduxProvider } from "react-redux";

import Layout from "./components/Layout";
import createStore from "./store";

const store = createStore( window.REDUX_DATA );

const jsx = (
    <ReduxProvider store={ store }>
        <Router>
            <Layout />
        </Router>
    </ReduxProvider>
);

const app = document.getElementById( "app" );
ReactDOM.hydrate( jsx, app );
Notice that we call createStore twice, first on the server, then on the client. However, on the client, we initialize the state with whatever state was saved on the server. This process is similar to the DOM hydration.

The full example can be seen on the redux tag in the same repository.

Fetch Data
The final piece of the puzzle is loading data. This is where it gets a bit trickier. Let’s say we have an API serving JSON data.

In our codebase, I fetch all the events from the 2018 Formula 1 season from a public API. Let’s say we want to display all the events on the Home page.

We can call our API only from the client after the React app is mounted and everything is rendered. But this will have a bad impact on UX, potentially showing a spinner or a loader before the user sees relevant content.

We already have Redux, as a way of storing data on the server and sending it over to the client.

What if we make our API calls on the server, store the results in Redux, and then render the full HTML with the relevant data for the client?

But how can we know which calls need to be made?

First, we need a different way of declaring routes. So we switch to the so-called routes config file.

export default [
    {
        path: "/",
        component: Home,
        exact: true,
    },
    {
        path: "/about",
        component: About,
        exact: true,
    },
    {
        path: "/contact",
        component: Contact,
        exact: true,
    },
    {
        path: "/secret",
        component: Secret,
        exact: true,
    },
];
And we statically declare the data requirements on each component.

/* ... */
import { fetchData } from "../store";

class Home extends React.Component {
    /* ... */

    render( ) {
        const { circuits } = this.props;

        return (
            /* ... */
        );
    }
}
Home.serverFetch = fetchData; // static declaration of data requirements

/* ... */
Keep in mind that serverFetch is made up, you can use whatever sounds better for you.

As a note here, fetchData is a Redux thunk action, returning a Promise when dispatched.

On the server, we can use a special function from react-router, called matchRoute.

/* ... */
import { StaticRouter, matchPath } from "react-router-dom";
import routes from "./routes";

/* ... */

app.get( "/*", ( req, res ) => {
    /* ... */

    const dataRequirements =
        routes
            .filter( route => matchPath( req.url, route ) ) // filter matching paths
            .map( route => route.component ) // map to components
            .filter( comp => comp.serverFetch ) // check if components have data requirement
            .map( comp => store.dispatch( comp.serverFetch( ) ) ); // dispatch data requirement

    Promise.all( dataRequirements ).then( ( ) => {
        const jsx = (
            <ReduxProvider store={ store }>
                <StaticRouter context={ context } location={ req.url }>
                    <Layout />
                </StaticRouter>
            </ReduxProvider>
        );
        const reactDom = renderToString( jsx );

        const reduxState = store.getState( );

        res.writeHead( 200, { "Content-Type": "text/html" } );
        res.end( htmlTemplate( reactDom, reduxState ) );
    } );
} );

/* ... */
With this, we get a list of components that will be mounted when React is rendered to string on the current URL.

We gather the data requirements and we wait for all the API calls to return. Finally, we resume the server render, but with data already available in Redux.

The full example can be seen on the fetch-data tag in the same repository.

You probably notice that this comes with a performance penalty, because we’re delaying the render until the data is fetched.

This is where you start comparing metrics and do your best to understand which calls are essential and which aren’t. For example, fetching products for an e-commerce app might be crucial, but prices and sidebar filters can be lazy loaded.

Helmet
As a bonus, let’s look at SEO. While working with React, you may want to set different values in your <head> tag. For example, you may want to set the title, meta tags, keywords, and so on.

Keep in mind that the <head> tag is normally not part of your React app!

react-helmet has you covered in this scenario. And it has great support for SSR.

import React from "react";
import Helmet from "react-helmet";

const Contact = () => (
    <div>
        <h2>This is the contact page</h2>
        <Helmet>
            <title>Contact Page</title>
            <meta name="description" content="This is a proof of concept for React SSR" />
        </Helmet>
    </div>
);

export default Contact;
You just add your head data anywhere in your component tree. This gives you support for changing values outside the mounted React app on the client.

And now we add the support for SSR:

/* ... */
import Helmet from "react-helmet";
/* ... */

app.get( "/*", ( req, res ) => {
    /* ... */
        const jsx = (
            <ReduxProvider store={ store }>
                <StaticRouter context={ context } location={ req.url }>
                    <Layout />
                </StaticRouter>
            </ReduxProvider>
        );
        const reactDom = renderToString( jsx );
        const reduxState = store.getState( );
        const helmetData = Helmet.renderStatic( );

        res.writeHead( 200, { "Content-Type": "text/html" } );
        res.end( htmlTemplate( reactDom, reduxState, helmetData ) );
    } );
} );

app.listen( 2048 );

function htmlTemplate( reactDom, reduxState, helmetData ) {
    return `
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="utf-8">
            ${ helmetData.title.toString( ) }
            ${ helmetData.meta.toString( ) }
            <title>React SSR</title>
        </head>
        
        /* ... */
    `;
}
And now we have a fully functional React SSR example!

We started from a simple render of HTML in the context of an Express app. We gradually added routing, state management, and data fetching. Finally, we handled changes outside the scope of the React application.

The final codebase is on master on the same repository that was mentioned before.

Conclusion
As you’ve seen, SSR is not a big deal, but it can get complex. And it’s much easier to grasp if you build your needs step by step.

Is it worth adding SSR to your application? As always, it depends. It’s a must if your website is public and accessible to hundreds of thousands of users. But if you’re building a tool/dashboard-like application it might not be worth the effort.

However, leveraging the power of universal apps is a step forward for the front-end community.

Do you use a similar approach for SSR? Or you think I missed something? Drop me a message below or on Twitter.

If you found this article useful, help me share it with the community!
