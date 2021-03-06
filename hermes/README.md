# Hermes

Hermes offers several services not available in a Web Browser through one (or several) Node.js processes:

* File-system abstraction for the Ares IDE.
* Archive service (used by the PhoneGap build service).

## Filesystem service

### Protocol

#### Resources

* `/id/{id}` resources are accessible via avery verbs described below.  These resources are used to browse the folder tree & get idividual items
* `/file/*` resources are files that are known to exist.  These resources are used only by the Enyo Javascript parser.  They can only be accessed using `GET`.

#### Verbs

Hermes file-system providers use verbs that closely mimic the semantics defined by [WebDAV (RFC4918)](http://tools.ietf.org/html/rfc4918):  although Hermes reuses the same HTTP verbs (`GET`, `PUT`, `PROPFIND`, `MKCOL`, `DELETE` ...), it differs in terms of carried data.  Many (if not most) of the HTTP clients implement only the `GET` and `POST` HTTP verbs:  Hermes uses the same HTTP Method Overrides as WebDAV usually do (tunnel every requests but `GET` into `POST` requests that include a special `_method` query parameter)

* `PROPFIND` lists properties of a resource.  It recurses into the collections according to the `depth` parameter, which may be 0, 1, … etc plus `infinity`.  For example, the following directory structure:

		$ tree 1/
		1/
		├── 0
		└── 1

… corresponds to the following JSON object returned by `PROPFIND`.

		$ curl "http://127.0.0.1:9009/id/%2F?_method=PROPFIND&depth=10"
		{
		    "isDir": true, 
		    "path": "/", 
		    "name": "", 
		    "contents": [
		        {
		            "isDir": false, 
		            "path": "/0", 
		            "name": "0", 
		            "id": "%2F0"
		        }, 
		        {
		            "isDir": false, 
		            "path": "/1", 
		            "name": "1", 
		            "id": "%2F1"
		        }
		    ], 
		    "id": "%2F"
		}

* `MKCOL` create a collection (a folder) into the given collection, as `name` passed as a query parameter (and therefore URL-encoded):

		$ curl -d "" "http://127.0.0.1:9009/id/%2F?_method=MKCOL&name=tata"

* `PUT` creates or overwrite a file resource.

* `DELETE` delete a resource (a file), which might be a collection (a folder).  Status codes:
  * `200/Ok` success, resource successfully removed.  The method returns the new status (`PROPFIND`) of the parent of the deleted resource.

		$ curl -d "" "http://127.0.0.1:9009/id/%2Ftata?_method=DELETE"

* `COPY` reccursively copies a resource as a new `name` or `path` provided in the query string (one of them is required).  The optionnal query parameter `overwrite` defines whether the `COPY` should try to overwrite an existing resource or not.  The method returns the new status (`PROPFIND`) of the target resource.
  * `201/Created` success, a new resource is created
  * `200/Ok` success, an existing resource was successfully overwritten (query parameter `overwrite` was set to `true`)
  * `412/Precondition-Failed` failure, not allowed to copy onto an exising resource

* `MOVE` has the exact same parameters and return codes as `COPY`

#### Parameters

##### Path parameters

		http://127.0.0.1:{port}/{urlPrefix}/id/{id}

* **`port`** TCP port
* **`urlPrefix`** server path
* **`id`** resource IDs.  Even when readable, those resources need to be handled as if they were opaque values.

##### Query parameters

* **`name`** File or folder name, for creation methods (`MKCOL`, `PUT`, `COPY`, `MOVE`).  This is required, as the resource `id` is not (yet) known.
* **`folderId`** target folder `id` used by `COPY` and `MOVE` methods.
* **`overwrite`** when set to `true` with the `COPY`, `MOVE` and `PUT` methods, causes the target resource to be overwritten (not merged) in case it already exists.  When absent, `overwrite` defaults to `false`.
* **`depth`** is only valid with the `PROPFIND` method.  It defines the recursion level of the request in the folder tree.  When omitted, it defaults to `1` (list immediate child resources of a folder).  The special value `infinity` causes the methods to recurse to the deepest possible level in the folder tree.

	
### Run Test Suite

**New** the test suite does not cover the former verbs used by Hermes, and implemented into `filesystem/index.js`.

So far, only hermes local filesystem access comes with a (small) test suite, that relies on [Mocha](http://visionmedia.github.com/mocha/) and [Should.js](https://github.com/visionmedia/should.js).  Run it using:

	$ hermes/node_modules/mocha/bin/mocha hermes/fsLocal.spec.js

## Archive service

This is the `arZip.js` service.  It takes 2 arguments:

* `pathname`
* `port`

It can be started standlone using the following command-line (or a similar one):


…in which case it can be tested using `curl` by a command-line like the following one:

	$ curl \
		-F "file=@config.xml;filename=config.xml" \
		-F "file=@icon.png;filename=images/icon.png" \
		-F "archive=myapp.zip" \
		"http://127.0.0.1:9019/arZip" > /tmp/toto.zip
	  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
	                                 Dload  Upload   Total   Spent    Left  Speed
	100  8387    0  3860  100  4527   216k   253k --:--:-- --:--:-- --:--:--  276k

The generated file is expected to look like to below:

	$ unzip -l /tmp/toto.zip 
	Archive:  /tmp/toto.zip
	  Length     Date   Time    Name
	 --------    ----   ----    ----
	      942  10-17-12 17:26   config.xml
	     3126  10-17-12 17:26   images/icon.png
	 --------                   -------
	     4068                   2 files

## Debug

It is possible to debug an Hermes services by commenting-out the `command` property of the service to be debugged in the ide.json & start it manually along with `--debug` or `--debug-brk`  to later connect a `node-inspector`.

Start the file server:

	$ node ares-project/hermes/filesystem/index.js 9010 hermes/filesystem/root
	
**Debugging:** The following sequence (to be run in separated terminals) opens the ARES local file server in debug-mode using `node-inspector`.

	$ node --debug ares-project/hermes/filesystem/index.js 9010 hermes/filesystem/root
		
...then start `node-inspector` & the browser windows from a separated terminal:

	$ open -a Chromium http://localhost:9010/ide/ares/index.html
	$ node-inspector &
	$ open -a Chromium http://localhost:8080/debug?port=5858

References:


## References

1. [GitHub: node-inspector](https://github.com/dannycoates/node-inspector)
1. [Using node-inspector to debug node.js applications including on Windows (and using ryppi for modules)](http://codebetter.com/glennblock/2011/10/13/using-node-inspector-to-debug-node-js-applications-including-on-windows-and-using-ryppi-for-modules/)
1. Screencast [Debugging with `node-inspector`](http://howtonode.org/debugging-with-node-inspector)
1. [RFC4918 - HTTP Extensions for Web Distributed Authoring and Versioning (WebDAV)](http://tools.ietf.org/html/rfc4918)
1. [RFC5689 - Extended MKCOL for Web Distributed Authoring and Versioning (WebDAV)](http://tools.ietf.org/html/rfc5689)
