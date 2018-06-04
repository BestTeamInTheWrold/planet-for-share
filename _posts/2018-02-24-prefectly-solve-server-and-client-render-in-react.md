---
layout: post
title: "Perfectly solve server and client render in React"
description: "React, a javascript library for building user interfaces."
image: assets/images/pic09.jpg
---
ReactJS stands out from a heap of front-end JavaScript frameworks, which dues to its one-way data binding and reactive data flow.
In order to reuse processing logic in both server and client side, we use express for backend, which is a minimal and flexible Node.js web application framework.
So the puzzle comes in: `How to handle render in both sides for multiple request`
#### Brief Glance
Choose [React Router](https://github.com/reactjs/react-router) for dynamic route matching, separate react component into two types:
container component and view component, introduce a handler layer mechanism.
Further explanation: Container component is a smart component which knows how to fetch data it needs and operate the puppet ones (view component).
#### Going Deep
* Route matching
After look into [React Router docs](https://github.com/reactjs/react-router/tree/1.0.x/docs), we can build Nesting logic in Routes.js.
{% highlight bash %}
export default (
  <Route path="/" name="app" component={App}>
    <Route path="landing-page" name="landingPage" component={LandingPage}/>
    <Route path="other-page" name="otherPage" component={OtherPage}/>
  </Route>
);
{% endhighlight%}
* Handler layer definition
**Firstly** consider every container component(e.g LandingPage) will render one page content, so define static method `fetchApiData`.
{% highlight bash %}
  static fetchApiData(host) {
    try {
      const bffApi = new BffApi(host);
      return bffApi.getUser().then(d => {
          return {user: d};
        }
      );
    }
    catch (e) {
      return when.reject(e);
    }
  }
{% endhighlight%}
**Secondly** build the matching logic for both sides, here we introduce `fetchApiData util` to help us,
filter the component and return its own `fetchApiData` result after calling.   
{% highlight bash%}
import when from 'when';
export default function (renderProps, host) {
  let data = {};
  const filteredRoutes = renderProps.routes.filter(route => route.component)
    .filter(route => route.component.fetchApiData);
  return when.Promise.all(
    filteredRoutes.map(route => {
      return route.component.fetchApiData(renderProps, host).then(d => {
        data = d;
      });
    })
  ).timeout(10000).then(() => data);
}
{% endhighlight%}
After having this beautiful util, magic time shows.
In server side render:
{% highlight bash%}
import routes from './Routes';
import fetchApiData from "./fetchApiData";
app.all('/*', (req, res) => {
  match({routes: routes, location: req.url}, async function (error, redirectLocation, renderProps) {
    logger.info("Request headers hostname: " + req.headers.host);
    fetchApiData(renderProps, bffHost)
      .then(apiData => {
          if (error) {
            logger.error(error);
            res.status(500).send(error);
          } else if (redirectLocation) {
            res.redirect(302, redirectLocation.pathname + redirectLocation.search);
          } else if (renderProps) {
            apiData.status = 200;
            renderProps.params.apiData = apiData;
            const context = <RoutingContext {...renderProps}/>;
            const renderResult = '<!DOCTYPE html>' + renderToString(context);
            res.status(200).send(html);
          } else {
            res.status(404).send('Not found');
          }
        }
      );
  });
});
{% endhighlight%}
In client side render:
{% highlight bash%}
import {createHistory, useBasename} from 'history';
import routes from './Routes';
import fetchApiData from "./fetchApiData";
const history = useBasename(createHistory)({basename: ''});
const pageData = window.pageData;
function matchLocation(location) {
  match({routes, location}, (error, redirectLocation, renderProps) => {
    if (pageData && !_.isEmpty(pageData)) {
      const data = pageData;
      let component;
      if (data.status == 200) {
        renderProps.params.apiData = pageData;
        component = <RoutingContext {...renderProps} history={history}/>;
        render(component, document);
      }
    } else {
      fetchApiData(renderProps, bffHost)
        .then(apiData => {
            apiData.status = 200;
            renderProps.params.apiData = apiData;
            let component = <RoutingContext {...renderProps} history={history}/>;
            render(component, document);
          }
        );
    }

  });
}
history.listen(matchLocation);
{% endhighlight %}
Client side render, in order to prevent fetchApiData method call again if already got in server side,
our window has the pageData attribute. If pageData exists, just use the data to render whole page. If not,
client side will use the same logic to fetch api data.
**Thirdly** extract shared DOM structure, put them into Root Component(e.g App). Also in server sides render code
(`'<!DOCTYPE html>' + renderToString(context);`), directly return whole html instead of define index.html.  
{% highlight bash%}
import React from 'react';
import Header from './shared/Header';
import Footer from './shared/Footer';
export default class App extends React.Component {
  render() {
    return (
      <html lang="en">
      <head>
        <link rel="stylesheet" href="/assets/css/main.css"/>
      </head>
      <body>
      <Header/>
      {React.cloneElement(this.props.children, {...this.props})}
      <Footer/>
      <script type="text/javascript" src="/assets/js/vendor.js"}></script>
      <script type="text/javascript" src="/assets/js/main.js"}></script>
      </body>
      </html>
    );
  }
}
{% endhighlight%}
* Seo requirement
Different page has different meta data in head tag, which meets seo requirement.
In order to do that, additional static method is needed for container component.
{% highlight bash%}
static prepareMetadata() {
    return (
       <meta name="description" content="Page description"/>
       <meta rel="canonical" href="http://www.example.com/"/>
    );
  }
{% endhighlight%}
Also `prepareMetadata util` is needed.
{% highlight bash%}
export default function (renderProps) {
  let data = {};
  const filteredRoutes = renderProps.routes.filter(route => route.component)
    .filter(route => route.component.prepareMetadata);
  filteredRoutes.map(route =>
    data = route.component.prepareMetadata(renderProps)
  );
  return data;
}
{% endhighlight%}  
If more complex seo requirement comes, for example, need to generate meta data according to different api result,
current implementation can easily expand, just change the render logic like the following, use server side as example:
{% highlight bash%}
app.all('/*', (req, res) => {
  match({routes: routes, location: req.url}, async function (error, redirectLocation, renderProps) {
    logger.info("Request headers hostname: " + req.headers.host);
    fetchApiData(renderProps, bffHost)
      .then(apiData => {
            ...
            apiData.status = 200;
            renderProps.params.apiData = apiData;
            renderProps.params.metaData = prepareMetaData(renderProps, apiData);
            const context = <RoutingContext {...renderProps}/>;
            ...
          } else {
            res.status(404).send('Not found');
          }
        }
      );
  });
});
{% endhighlight%}
Call the prepareMetaData and pass api data as parameters, stuff metaData DOM as renderProps.params attribute, then you can use the metaData in App component.
* Configuration management
For configuration, make sure both sides can use, and the difficult part is: `How client side get configuration`.
Two ways:  
1) For server side, define global variables. For client side, replace special words in index.html, add extra attribute for window.
{% highlight bash%}
<script>
  var PAGE = PAGE ? PAGE : {};
  PAGE.api_endpoint = '¡APIENDPOINT!';
  PAGE.api_key = '¡APIKEY!';
</script>
{% endhighlight%}
2) For server side read configuration file. For client side, let server side pass configuration as renderProps.params attribute into App component, then insert script.
{% highlight bash%}
import config from './util/configManager';
const configuration = config.generateConfigurationForBrowser();
renderProps.params.configuration = configuration;
......
function insertAttribute() {
  return {
  __html: `window.configurationFromNode = ${JSON.stringify(this.props.params.configuration)};
           window.pageData = ${JSON.stringify(this.props.params.apiData)};`
  };
};
<script type="text/javascript" dangerouslySetInnerHTML={insertAttribute()} ></script>
{% endhighlight %}
