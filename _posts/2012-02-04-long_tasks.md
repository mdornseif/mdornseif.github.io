---
layout: post
title: Long running Tasks for Websites (en)
---

{{ page.title }}
================

Some reactions to user actions need more than a few seconds to compute. [Jakob Nielsen][1] for tasks over 10 seconds waiting time "users will want to perform other tasks while waiting for the computer to finish". He [as others][2] also states that an activity indication and an estimation of the total time to completion are important.

Google App Engine enforces the paradigm, that webpages have to respond within a few seconds with it's request deadline of 10 seconds (later lifted to 30 and then to 60 seconds).

For an Web-Application an low latancy design for long computation means something like this:

1. User initiates the computation.
2. Computation is started in the Background.
3. To the user a page "please wait for computation to finish" is displayed
4. Check if computation is finished, if not, go back to 3., else proceed.
5. Display results of computation to user.

<img src="/images/2012-02-04-long_tasks.png">

This involves a lot of moving parts including RPC between bachend task and frontend.  Besides the activity indicator the requirement to handle users performing other tasks while waiting. That basically means, that users must be able to bookmark the computation and navigate away and later come back to retrive the result.

Implementation on AppEngine
---------------------------

For storing state and simple minded RPC the Datastore is the best option. We can use an Model like this:

    class LongTask(db.Model):
        parameters_blob = db.BlobProperty()
        result_blob = db.BlobProperty()
        status = db.StringProperty(required=True, default='ready',
            choices=['ready',    # to be started
                     'started',  # task has been initiated 
                     'error',    # an error occured during execution
                     'done', 
                     'finished']
                     )

Starting a Task could look like this:

    # convert HTTP patameters to a dict
    # for all parameters not starting with `_`
    parameters = dict([(name, self.request.get(name)) 
                       for name in self.request.arguments()
                       if not name.startswith('_')])
    # Create task Entity and start Taskqueue
    task = gaetk_LongTask(parameters_blob=pickle.dumps(paramters),
                          status='ready')
    task.put()
    taskqueue.add(url=TASKHANDLER_URL, method='GET',
              params={'_longtaskjob': 'execute', 
                      '_longtaskid': task.key()})
    # redirect to status queue
    parameters = urlencode([('_longtaskjob', 'query'),
                            ('_longtaskid', task.key())
                           ])
    raise HTTP307_TemporaryRedirect(location=STATUS_PAGE_URL
                                             + '?' + parameters)


The Task handler istself has to do some status updates and then do the real work. In the usual setup it has 10 Minutes todo it's work:

    task = gaetk_LongTask.get(self.request.get('_longtaskid', ''))
    task.status = 'started'
    task.put()
    # Decode Parameters from Datastore and execute the actual task.
    parameters = pickle.loads(task.parameters_blob)
    try:
        # to the actual work:
        time.sleep(61)  # obviously this is only an example
        result = output ot your calculation
        task.result_blob = pickle.dumps(result)
        task.status = 'done'
        task.put()
    except Exception, msg:
        # If an exception occured, note that and re-raise an error.
        task.status = 'error'
        task.put()
        raise


Than we need the handler to display the status / wait page:

    task = gaetk_LongTask.get(self.request.get('_longtaskid', ''))
    if task.status == 'done':
        result = pickle.loads(task.result_blob)
        self.response.out.write(result)
    elif task.status == 'started':
        # Generate HTML output
        html = u"""<html><head><meta http-equiv="refresh" content="3">
        <title>Task Status</title></head>
        <body><p>Still working.</p></body></html>"""
        self.response.write(html)
    elif task.status == 'error':
        html = u"""<html><head><meta http-equiv="refresh" content="3">
        <title>Task Status</title></head>
        <body><p>Error, automated retry in progress.</p></body></html>"""
        self.response.write(html)


This implements the machinery to run tasks anyncronous to displaying pages and it also implements bookmarkable, stable URLs for tasks. What we are still missing is a progress indicator. This requires somewhat more sophisticated messaging between the backend task, the fontend server and the client.

For intra-Server communication we can use memcache which works "reliable enough" for our purposes. The long-running computation can log a text message the current step and the (expected) total number of steps. 

    def log_progress(self, message, step=0, total_steps=0):
        memcache.set("longtask_status_%s" % self.task.key(),
                     dict(message=message, step=step,
                          total_steps=total_steps))

The handler for the status / wait page can now be extended to display progress.

    statusinfo = memcache.get("longtask_status_%s" % task.key())
    info = ''
    if statusinfo and statusinfo.get('total_steps'):
        info = u"""<p>Progress:
           <progress value="%d" max="%d">%d %%</progress></p>
           """ % (statusinfo.get('step'),
                  statusinfo.get('total_steps'),
                  int(statusinfo.get('step') * 100.0 / statusinfo.get('total_steps')))
        else:
            # indetermine progress bar
            info = u"<p><progress></progress></p>"
    if statusinfo and statusinfo.get('message'):
        info += '<p>%s</p>' % statusinfo.get('message')
    html = u"""<html><head><meta http-equiv="refresh" content="3">
    <title>Task Status</title></head>
    <body>%s</body></html>""" % info
    self.response.write(html)


Packaging it nicely
-------------------

This can end up in an awful lot of code scattered arround. Fotunately it leands itself to a nice OOP implementation. [`gae_longtask`][3] all bundles it up. 

In the simpelest form you can use it by just overwriting a single function. See the `gaetk_longtask`. Here a somewhat more involved real-world example:

    class IntrastatRohdaten(longtask.LongRunningTaskHandler):
        def execute_task(self, parameters):
            monthlast = date.today() - timedelta(days=datetime.date.today().day + 1)
            monthfirst = monthlast - timedelta(days=vormonatsletzter.day - 1)
            self.log_progress("Calculating INTRASTAT Data %s to %s"
                              % (monthlast, monthfirst),
                              step=0, total_steps=33)
            count = 0
            days = []
            day = monthlast
            while day <= monthfirst:
                days.append(day)
                day += datetime.timedelta(days=1)

            for day in days:
                count += 1
                self.log_progress("Handling %s: so far %d Invoices" % (day, count),
                                  step=day.day, total_steps=len(days) + 2)
                for rechnung in Rechnung.all().filter('datum =', tag).run():
                    intrastat_fuer_rechung(rechnung)

            self.log_progress("Summing up", step=32, total_steps=33)
            vormonat = intrastat_monat_aufsummieren(monthfirst)
            self.log_progress("Formating", step=33, total_steps=33)
            csvdata = reformat(vormonat)
            return csvdata

    def display_result(self, paramters, result):
        self.response.headers['Content-Type'] = 'text/plain'
        self.response.write(result)


Get the `gae_longtask` [here at github][3].


[1]: http://www.useit.com/papers/responsetime.html
[2]: http://www.asktog.com/basics/firstPrinciples.html
[3]: https://github.com/mdornseif/gaetk_longtask