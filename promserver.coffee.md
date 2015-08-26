Promserver
==========

Promserver is a promise based webserver. It aims to be functional and
composable, providing you with a few simple tools that allow you to build
complex features.

    http = require 'http'
    Promise = require 'bluebird'
    stream = require 'stream'

Our entry point is super simple, we take an (optional) host, a port, and a
function, and we call that function on every request. That function's
(promised) result is then returned to the client.

    serve = (host, port, fn) ->
      [host, port, fn] = [undefined, host, port] if typeof fn is 'undefined'


This is currently too much logic. We should eventually split this up into much
smaller composable functions.

      server = http.createServer (request, response) ->
        Promise.resolve fn(request)
          .then (res) ->
            if typeof res is 'object'
              response.statusCode = res.statusCode or res.status or response.statusCode
              response.statusMessage = res.statusMessage or res.statusText or response.statusMessage
              headers = res.headers?._headers or res.headers
              response.setHeader k, v for k, v of headers if headers
              body = res.body
            else
              body = res

            if body instanceof stream.Readable
              body.pipe response
            else if typeof body is 'object'
              response.setHeader 'Content-Type', 'application/json'
              response.end JSON.stringify body
            else if typeof body is 'number'
              response.writeHead body
              response.end()
            else if typeof body is 'string'
              response.end(body)
            else
              response.end()

          .catch (err) ->
            throw err
            response.writeHead 500
            response.end(err.toString())

Return the node http server object in case we want to do anything important
with it.

      server.listen = Promise.promisify(server.listen)
      server.close = Promise.promisify(server.close)

      server.ready = server.listen(port, host)
      server

    module.exports = serve

