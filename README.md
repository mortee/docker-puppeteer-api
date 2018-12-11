# Puppeteer API

Headless chrome scrapper API based on [puppeteer](https://github.com/GoogleChrome/puppeteer).

This image contains:

1. Google Chrome working in headless mode.
1. ExpressJS application exposing single endpoint allowing to get contents of the website using puppeteer and headless chrome.
1. Alternative command line interface for the above (CLI).

Features:

1. Fetching content from URL generated by real browser on `load` event.
1. Fetching content from URL generated by real browser on element appearance in DOM. This is especially useful when page you want to get loads some dynamic data through Ajax and pushes it into DOM **after** `load` event.
1. Simple REST API endpoint with hash-based security.
1. CLI to get data with single command.

## How to use

### With REST API

Start docker container using following command:

```
docker run -d -p 8000:8000 --restart unless-stopped --name puppeteer-api -e "SALT=abcdef" l0coful/puppeteer-api
```

Where:

1. The container port `8000`  is bound to `8000` host port making API accessible at `http://localhost:8000` (also through the network).
1. The `SALT` environment variable provided should be set to some random string and will be used for requests security.

You can check if everything is OK tracing container logs:

```
$ docker logs puppeteer-api
[...]
Using 'abcdef' as salt
[local] Scrapper API is listening on port: 8000
```

#### Fetching URL content

##### TL;DR

POST following JSON to `http://localhost:8000/scrap`:

```js
{
	"url": "http://example.com",
	"selector": "h1",
	"hash": "129f2756eac7b62b5b7f428175e5a4e3"
}
```

Where:

1. `url` is the URL to fetch.
1. `selector` is an optional selector. If provided the content will be returned only after this selector returns non-empty elements array. Otherwise the content will be returned on page `onload` event.
1. `hash` is the request signature done by ``md5(`${url}:${SALT}`)``.

##### On page load

By default you can fetch DOM untouched by javascript using CURL:

```
$ curl -s http://example.com | grep "h1"
<h1>Example Domain</h1>
```

To do the same but using this API you first need to calculate you URL hash using previously used `SALT` and URL you want to fetch:

```
$ echo -n "http://example.com:abcdef" | md5sum
129f2756eac7b62b5b7f428175e5a4e3 -
```

Having this *signature* you can now ask API for the URL:

```
$ curl \
	-s \
	-X POST \
	-H "Content-Type: application/json" \
	-d '{"url": "http://example.com","hash":"129f2756eac7b62b5b7f428175e5a4e3"}' \
	http://localhost:8000/scrap \
| grep "h1"

<h1>Example Domain</h1>
```

In logs you can see that HTML has been returned after `load` event:

```
$ docker logs puppeteer-api
[...]
[68825600] requesting from: ::ffff:172.17.0.1 to fetch: http://example.com
[68825600] starting chrome browser
[68825600] going to: http://example.com
[68825600] page loaded, resolving content immediately
[68825600] closing chrome browser
[68825600] sending data with: 1262 bytes
```

##### On element appearance in DOM

In the second mode the API fetches URL content only after element with given [CSS selector](http://htmldog.com/references/css/selectors/) appears in the DOM. This can be done this way (for `h1` selector):

```
$ curl \
	-s \
	-X POST \
	-H "Content-Type: application/json" \
	-d '{"url": "http://example.com","selector":"h1","hash":"129f2756eac7b62b5b7f428175e5a4e3"}' \
	http://localhost:8000/scrap \
| grep "h1"

<h1>Example Domain</h1>
```

Related logs:

```
$ docker logs puppeteer-api
[51b45bd7] requesting from: ::ffff:172.17.0.1 to fetch: http://example.com
[51b45bd7] starting chrome browser
[51b45bd7] going to: http://example.com
[51b45bd7] page loaded, setting 1000 ms refresh interval
[51b45bd7] element with selector: 'h1' appeared, resolving content
[51b45bd7] clearing refresh interval
[51b45bd7] closing chrome browser
[51b45bd7] sending data with: 1262 bytes
```

### With command line

The same options as above are executable from the command line in the host OS using temporary docker container.

On page load:

```
$ docker run --rm -it --entrypoint "/bin/bash" l0coful/puppeteer-api puppeteer fetch http://example.com | grep "h1"
<h1>Example Domain</h1>
```

On `h1` element appearance in DOM:

```
$ docker run --rm -it --entrypoint "/bin/bash" l0coful/puppeteer-api puppeteer fetch -s "h1" http://example.com | grep "h1"
<h1>Example Domain</h1>
```


## How to build

```
docker build --tag l0coful/puppeteer-api .
```