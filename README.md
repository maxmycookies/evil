 Puppeteer is a high level abstraction over the Chrome Devtools Protocol that gives you a user-friendly API to drive Chromium (or Blink) based environments. Developers create high level abstractions like Puppeteer with the intent of making common use cases trivial and, as you stray further and further from those common cases, it’s not unlikely that you’ll need to jump passed those abstractions. Thankfully, you can still access the Chrome Devtools Protocol directly within Puppeteer to get the best of both worlds.
 
 **Intercepting and Modifying Responses** The ability to modify the execution of a running application is an important part of reverse engineering, troubleshooting, and analysis. Browser developer tools and their plugins have come a long way over the last 10 years but browser makers optimize them for the user experience of a developer actively in the midst of the development process. Once you leave that environment wave goodbye to the hooks for development libraries, say adios to your logging, be prepared to interpret cryptic error codes instead of comprehensive messages, and get ready to turn on your mental debugger as you step through source code minified beyond comprehension. Sourcemaps are helpful but they can break as build tools change or even across browser updates. You might not even notice broken sourcemaps until troubleshooting in production and, at that point, it’s usually too late.
 
 You can find an introduction to intercepting and modifying resources via the Chrome Devtools Protocol (CDP) on the [modifying](https://blog.shapesecurity.com/2018/09/17/intercepting-and-modifying-responses-with-chrome-via-the-devtools-protocol/) or visit YouTube for a version that leverages the concepts within Puppeteer.
 
 **Base puppeteer script** This is the base puppeteer script we’re starting from. This gets us a browser and the default page that loads as soon as Chrome or Chromium boots up.
 
 ```
 const puppeteer = require('puppeteer');
 
 (async function main(){
   const browser = await puppeteer.launch({
     headless:false, 
     defaultViewport:null,
     devtools: true
   });
 
   const page = (await browser.pages())[0];
 
   
 })()
 ```
 
 **Using the Devtools Protocol with Puppeteer** The first step is to have Puppeteer start a CDP session with the target page. Interaction with CDP happens over Web Sockets and you will need to create that connection per tab, or “page” in Puppeteer lexicon. It’s important to note that the connection persists even after navigating to another website as long as the tab remains open.
 
 ```
 const client = await page.target().createCDPSession();
 ```
 
 **Initializing a CDP session across all tabs** I don’t know about you but I have at least 30 tabs open at all times — gmail and music or a calendar pinned to the front permanently, 15 in the middle for sites I’ve opened in the past that I’ll never look at again but might so the tabs remain open. The 18th tab is a recent Google search, 19–29 are new tabs opened from that Google search or its results and the 30th onward is where I open new sites.
 
 Regardless of how you’ve personally chosen to abuse your RAM, having functionality exist in one tab alone is of limited use. You can listen for the browser’s targetcreated event to grab onto new tabs as they open.
 
 ```
 const page = (await browser.pages())[0];
 browser.on('targetcreated', async (target) = {
   const page = await target.page();
 });
 ```
 
 By combining what we’ve learned so far we can put together a base that allows us to get a CDP session for every page we open.
 
 ```
 const puppeteer = require('puppeteer');
 
 async function initializeCDP(page) {
   const client = await page.target().createCDPSession();
 
 }
 
 (async function main(){
   const browser = await puppeteer.launch({
     headless:false, 
     defaultViewport:null,
     devtools: true,
   });
 
   const page = (await browser.pages())[0];
 
   await initializeCDP(page);
 
   browser.on('targetcreated', async (target) = {
     const page = await target.page();
     await initializeCDP(page);
   })
 
 })()
 ```
 
 **CDP usage example: Intercepting traffic** The [Chrome Devtools Protocol](https://chromedevtools.github.io/devtools-protocol/) is rich with features but, without fail, the one thing I am doing over and over again is manipulating resources on the fly. It might only be that I’m pretty-printing JavaScript so that I can debug without ad hoc formatting but the ability to intercept and modify responses is a well worn tool in my toolbox.
 
 To do this we need to tell Chrome what URLs we want to intercept by way of a url pattern, a resource type, and the stage at which we want to intercept. The url pattern is a glob-like pattern format for matching URLs and the resource type is how Chrome is intending to use this resource. If you open a JavaScript file directly in a new tab you might be surprised to find that it treated like a Document whereas if you link the resource via a <script element the Chrome executes the resource as a Script. The difference makes sense but is not a common nuance to deal with on a day-to-day basis.
 
 The interception stage is important in our case because we want to intercept the actual response from the server so we intercept at the ‘HeadersReceived’ stage. This allows us to inspect the headers, determine if we want the body, and then request and manipulate the body as necessary.
 
 ```
 await client.send('Network.enable');
 await client.send('Network.setRequestInterception', { patterns: [
   { 
     urlPattern: '*', 
     resourceType: 'Script', 
     interceptionStage: 'HeadersReceived' 
   }
 ]});
 ```
 
 After setting our interception we can listen for the [Network.requestIntercepted](https://chromedevtools.github.io/devtools-protocol/tot/Network#event-requestIntercepted)[ event](https://chromedevtools.github.io/devtools-protocol/tot/Network#event-requestIntercepted) to hook into traffic. One of the properties of this event object is the interceptionId which allows us to get the body of the interception and then continue or abort the intercepted request with Network.continueInterceptedRequest.
 
 ```
 client.on('Network.requestIntercepted', ({ interceptionId }) = {
   client.send('Network.continueInterceptedRequest', {
     interceptionId,
   });
 });
 ```
 
 **Retrieving the body of a response** Because we’ve intercepted the response at the HeadersReceived stage we don’t have the body content available and need to make a call to [Network.getResponseBodyForInterception](https://chromedevtools.github.io/devtools-protocol/tot/Network#method-getResponseBodyForInterception) with our interception ID to retrieve it.
 
 ```
 const response = await client.send('Network.getResponseBodyForInterception',{
   interceptionId 
 });
 ```
 
 This response object comes with two properties, body and base64Encoded. base64Encoded is a boolean denoting that whether the body is in raw or encoded form.
 
 ```
 const originalBody = response.base64Encoded ? atob(response.body) : response.body;
 ```
 
 **Delivering a modified response** Delivering a modified response requires that you create a complete, raw HTTP response and send it along with your interception ID. This means you need to have an HTTP response code, version, headers, carriage return + newline separators (\r\n), and a blank line between the headers and the body. This raw response needs to be re-encoded in base64.
 
 ```
 const httpResponse = [
   'HTTP/1.1 200 OK',
   'Date: ' + (new Date()).toUTCString(),
   'Connection: closed',
   'Content-Length: ' + newBody.length,
   'Content-Type: application/javascript',
   '', // Do not delete
   newBody
 ].join('\r\n');
 client.send('Network.continueInterceptedRequest', {
   interceptionId,
   rawResponse: btoa(httpResponse)
 });
 ```
 
 **Wrapping it all up** The following is a complete script that intercepts every script from every tab and runs the content through [prettier](https://www.npmjs.com/package/prettier), the source formatting tool.
 
 ```
 const puppeteer = require('puppeteer');
 const prettier = require('prettier');
 const atob = require('atob');
 const btoa = require('btoa');
 
 const requestCache = new Map();
 
 const urlPatterns = [
   '*'
 ]
 
 function transform(source) {
   return prettier.format(source, {parser:'babel'});
 }
 
 
 async function intercept(page, patterns, transform) {
   const client = await page.target().createCDPSession();
 
   await client.send('Network.enable');
 
   await client.send('Network.setRequestInterception', { 
     patterns: patterns.map(pattern = ({
       urlPattern: pattern, resourceType: 'Script', interceptionStage: 'HeadersReceived'
     }))
   });
 
   client.on('Network.requestIntercepted', async ({ interceptionId, request, responseHeaders, resourceType }) = {
     console.log(`Intercepted ${request.url} {interception id: ${interceptionId}}`);
 
     const response = await client.send('Network.getResponseBodyForInterception',{ interceptionId });
 
     const contentTypeHeader = Object.keys(responseHeaders).find(k = k.toLowerCase() === 'content-type');
     let newBody, contentType = responseHeaders[contentTypeHeader];
 
     if (requestCache.has(response.body)) {
       newBody = requestCache.get(response.body);
     } else {
       const bodyData = response.base64Encoded ? atob(response.body) : response.body;
       try {
         if (resourceType === 'Script') newBody = transform(bodyData, { parser: 'babel' });
         else newBody === bodyData
       } catch(e) {
         console.log(`Failed to process ${request.url} {interception id: ${interceptionId}}: ${e}`);
         newBody = bodyData
       }
   
       requestCache.set(response.body, newBody);
     }
 
     const newHeaders = [
       'Date: ' + (new Date()).toUTCString(),
       'Connection: closed',
       'Content-Length: ' + newBody.length,
       'Content-Type: ' + contentType
     ];
 
     console.log(`Continuing interception ${interceptionId}`)
     client.send('Network.continueInterceptedRequest', {
       interceptionId,
       rawResponse: btoa('HTTP/1.1 200 OK' + '\r\n' + newHeaders.join('\r\n') + '\r\n\r\n' + newBody)
     });
   });
 }
 
 (async function main(){
   const browser = await puppeteer.launch({
     headless:false, 
     defaultViewport:null,
     devtools: true,
     args: ['--window-size=1920,1170','--window-position=0,0']
   });
 
   const page = (await browser.pages())[0];
 
   intercept(page, urlPatterns, transform);
 
   browser.on('targetcreated', async (target) = {
     const page = await target.page();
     intercept(page, urlPatterns, transform);
   });
 
 })()
 ```
 
 [Check out the other domains and methods available](https://chromedevtools.github.io/devtools-protocol/) via the Chrome Devtools Protocol. Mastering puppeteer and CDP can turn common chores into automated scripts and allow you to automate the creation of development environments specific to certain scenarios.

