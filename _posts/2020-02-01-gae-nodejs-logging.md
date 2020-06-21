---
layout: post
title: Logging on Google App Engine Node.js Standard Environment (en)
---

{{ page.title }}
================

[Google App Engine _Python_ Standard Environment](https://cloud.google.com/appengine/docs/standard/python) provides nice logging where all log entries related to a HTTP request are grouped. 
For [Node.js](https://cloud.google.com/appengine/docs/standard/nodejs) stdout and stderr are collected in the log viewer but with out any grouping - on entry per line.

Google suggests:

> To write log entries, we recommend that you integrate the Bunyan or Winston plugins with Cloud Logging. ([source](https://cloud.google.com/appengine/docs/standard/nodejs/writing-application-logs#writing_app_logs))

But using an in-process cloud logging client is generally an problematic approach for the async / single process server approach offered by node.js / Express.

Logging to stdout / stderr and transporting logs from there to the log viewer in a separate process (provided by App Engine infrastructure) provides much better decoupeling. This shoud be possible, as stated by Google:

> you can send simple text strings to stdout and stderr. The strings will appear as messages in the Logs Viewer, the command line, and the Cloud Logging API, and will be associated with the App Engine service and version that emitted them.
> If you want to filter these strings in the Logs Viewer by severity level, you need to format them as structured data. For more information, see Structured logging.
> If you want to correlate entries in the app log with the request log, your structured app log entries need to contain the request's trace identifier. You can extract the trace identifier from the X-Cloud-Trace-Context request header. In your structured log entry, write the ID to a field named logging.googleapis.com/trace. For more information about the X-Cloud-Trace-Context header, see Forcing a request to be traced.
> See an example of writing structured log entries with a trace ID in the Cloud Run documentation. You can use the same technique in your App Engine apps. ([source](https://cloud.google.com/appengine/docs/standard/nodejs/writing-application-logs#writing_structured_logs))

So this should be easy. Documentation for `trace` explains:

> Optional. Resource name of the trace associated with the log entry, if any. If it contains a relative resource name, the name is assumed to be relative to //tracing.googleapis.com. Example: projects/my-projectid/traces/06796866738c859f2f19b7cfb3214824

Unfortunately on the first glance this does not work as advertised:

![Logs not aggregated](http://f.foxel.org/zQksw2cEFaiRgJQo.png)


Actually the Logs _are_ aggregated if you do a sane selection in the Google Logs Viewer:

![Logs aggregated](http://f.foxel.org/j18MXXqKcTeXuX9K.png)

To get structured logging for your server using something like [Pino](http://getpino.io/#/) for logging is the right aproach. You need some Setup to map Pino field names to the ones used by Stackdriver / Google Cloud Logging.

Also we can wire logging into the Express http machinery yo ensure that th whole grouping per request works nicely.

Interesting doumentation for field Names at Google is somewhat scattered:

* [severity](https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry#logseverity)
* [httpRequest](https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry#httprequest)
* [operation](https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry#logentryoperation)


Some mapping has to be done. Setting up Pino for Stackdriver looks like this:

```javascript
// https://github.com/pinojs/pino/blob/master/docs/api.md#level-string
// https://cloud.google.com/logging/docs/agent/configuration#special-fields
function _levelToSeverity (level) {
  if (level < 30) { return 'DEBUG' }
  if (level <= 30) { return 'INFO' }
  if (level <= 39) { return 'NOTICE' }
  if (level <= 49) { return 'WARNING' }
  if (level <= 59) { return 'ERROR' }
  if (level <= 69) { return 'CRITICAL' } // also fatal
  if (level <= 79) { return 'ALERT' }
  if (level <= 99) { return 'EMERGENCY' }
  return 'DEFAULT'
}
const logger = pino({
  // Default Level: Info or taken from the environment Variable LOG_LEVEL
  level: process.env.LOG_LEVEL || 'info',
  formatters: { // Here the Mapping is happening
    level (label, number) {
      return { level: number, severity: _levelToSeverity(number) }
    }
  },
  messageKey: 'message' // AppEngine / StackDriver want's this key
  // Enable Pretty Printing on the development Server
  prettyPrint: (process.env.NODE_ENV || 'development') === 'development',
})
```

This enables us to write out logs in a Format which can be digested by the Google Log Viewer. Next thig is to enable logging per HTTP request. [pino-http](https://github.com/pinojs/pino-http) uses the logger defined above and can be wired into the Google machinery to use Googles Request-IDs:

```javascript
var httpLogger = require('pino-http')({
  logger: logger,
  genReqId: function (req) {
    return (
      req.headers['x-appengine-request-log-id'] ||
      req.headers['x-cloud-trace-context'] ||
      req.id
    )
  },
  useLevel: 'debug'
})
```

This adds a `log` Instance to every Express-Request and can be used in your route hadlers as `req.log.info('bla')` etc.

We also want to log the `logging.googleapis.com/trace` key as explained above. This can be done by a Express-Middleware:

```javascript
const gaeTraceMiddleware = (req, res, next) => {
  // Add log correlation to nest all log messages beneath request log in Log Viewer.
  const traceHeader = req.header('X-Cloud-Trace-Context')
  if (traceHeader && process.env.GOOGLE_CLOUD_PROJECT) {
    const [trace] = traceHeader.split('/')
    req.log = res.log = req.log.child({
      'logging.googleapis.com/trace': `projects/${process.env.GOOGLE_CLOUD_PROJECT}/traces/${trace}`,
      'appengine.googleapis.com/request_id': req.header('x-appengine-request-log-id')
    })
  }
  next()
}
```

Now the wired up Express instance can use res.log to output stuff for Google Log Viewer. There it will appear nearly as neat as Pytohn logs:

```javascript
const app = express()
app.use(httpLogger)
app.use(gaeTraceMiddleware)

// App Engine Instance Start
app.get('/_ah/warmup', (req, res) => {
  res.log.info('Warmup')
  res.send('HOT')
})
```