---
title: Tips for `supervisord` configuration
date: 2022-03-22T13:39:38Z
tags:
  - technical
  - quick-tips
  - containers
  - supervisor
---

[`supervisord`](http://supervisord.org/) is a really useful tool for managing processes in containers, but I find it's configuration a bit lacking.
There are some sensible defaults missing, no way to dynamically configure which processes should be running on a container, and I'm often left with duplicated configuration.
Having spent some time working with it I'd like to share my approach and solution to these headaches.

<!--more-->

One of the first hurdles I ran into when working with supervisor was trying to dynamically configure which processes should run.
Imagine this - we have a web application which offloads tasks to a background queue for later processing.
This background queue is consumed by a worker container which shares the same code and environment as the web application, but runs a different command to work on jobs from the queue.

Initially when trying to implement this architecture I tried running a second instance of the container with an overridden Docker `COMMAND`.
This was better than maintaining two images, but had the immediate drawback of losing all of the advantages of supervisor for the queue worker.
This also moved the responsibility of the command to run for a queue worker outside of the application boundary - going from being defined within the image to defined by some program running the container.

If is of course possible to set up multiple supervisor config files and choose between them dynamically, but this leads to a lot of duplication which just doesn't sit well with me.
My dream was to be able to define "roles" - a group of processes to run which can be supplied as config files and chosen between at runtime.
For example, a Python application might define a "web" role running a uWSGI web server and a "worker" role running celery to process jobs in the background.
By setting the `ROLES` environment variable to `web` or `worker` (or even `web,worker`), the relevant processes would be added to the configuration when the container starts up.

As with most of my problems, my answer was to write a bash script.
I split out my configuration file into a suitable directory structure:

```bash
- /etc/supervisor/
  - supervisor.conf    # Main supervisord configuration file
  - roles/             # A directory of process groups to choose from
    - web.conf         # Supervisord configuration for "web" containers
    - worker.conf      # Supervisord configuration for "worker" containers
```

I then wrote a script which reads the `ROLES` environment variable, and adds matching configuration file(s) from the `roles/` directory to the supervisor configuration:

```bash
#!/bin/sh

# split the comma seperated list into a newline seperated list which we can iterate over
ROLES=$(echo "$ROLES" | tr "," "\n")

# for each role specified, append the config to the main config file
for ROLE in $ROLES; do
    cat /etc/supervisor/roles/$ROLE.conf >> /etc/supervisor/supervisor.conf
done

# start supervisor and the processes
supervisord -c /etc/supervisor/supervisor.conf
```

This feels much tidier and gives a developer a lot of flexibity to run the same image in multiple configurations.
In development both the web and worker roles can run on a single container, whilst in a production environment the roles can run in seperate services which can be managed and scaled independently - with all of the benefits of supervisor.

Having proven the concept I quickly found ways to iterate and improve the script.
I was duplicating the same 4 lines of configuration in every process to send log output to `stdout`, the preferred destination for container logs.
I moved these lines into a `common.conf` file, and updated the script to append this file to the configuration after adding each role.

I also noticed whilst testing this setup that if I had configured supervisor to give up when a process couldn't start, the container wouldn't exit.
The remaining processes would continue to run, or supervisor would sit idle if everything had failed.

Supervisor can be configured with [event listeners](http://supervisord.org/events.html) - programs which run according to certain supervisor events.
One such event is the `PROCESS_STATE_FATAL` event, which is triggered in this scenario when a process cannot start and is in the `FATAL` state.

Event listeners are usually written to respond to specific events, but in this case I found it sufficient use a bash one-liner which waits for this event and asks supervisor to quit if it gets it:

```conf
[eventlistener:exit-on-fatal]
events=PROCESS_STATE_FATAL
command=sh -c "printf 'READY\n' && while read line; do kill -SIGQUIT $PPID; done < /dev/stdin"
```

I've posted an expanded version of this configuration [on my GitHub](github.com/leigholiver/supervisord-config), which can be easily customised and used in your images if you find this helpful.
