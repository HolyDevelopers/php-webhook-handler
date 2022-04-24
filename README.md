# PHP handler for GitHub webhooks

If you want to deploy from GitHub to some web server, this script might be
exactly what you need.

## Installation

### 1. Install webhook handler on your server

Put ``webhook-handler.php`` somewhere on your PHP-enabled web server, and make it
accessible for the outside world. For a bit more security add some random to the URL, e.g.
http://example.com/webhook-handler-2r0fZiEddBNx24kuSztq/webhook-handler.php

With shell access to your server you can just type:

``git clone https://github.com/HolyDevelopers/php-webhook-handler.git ~/public_html/webhook-handler-2r0fZiEddBNx24kuSztq``

### 2. Prepare your deploy script(s)
Put your deploy script somewhere on your server **outside of web root**. The simplest version
of such a script would just call ``cd myrepo; git pull``.

You find a more robust version of this in ``deploy_scripts/pull_from_git_repository.sh``

**Make sure your script is executable.** (``chmod u+x path/to/script.sh``)

### 3. Configure the webhook handler
Look at ``.ht.config.example.json`` and prepare a ``.ht.config.json`` with your configuration.
Make sure it's not accessible from the outside! In most apache configurations no files starting
with ``.ht`` will be served at all so you can leave it here. Or move it to a secure location
on your server (outside of web root). If you move it, make sure the PHP script knows where to find it.

If you want email notification (yes, you want!), enter your email
address to **email.to**. Set **email.from** to something meaningful
like deployment-notifications@example.com.

You can use it for several repositories or branches at the
same time by adding more entries to the **endpoints** list. For each endpoint
you need to set **endpoint.repo** to *"username/reponame"* and **endpoint.branch**
to your branch. Write a secret random string to **endpoint.secret**.
You can configure endpoints for different branches, for instance if you
use different branches for development/production etc.

Set **endpoint.run** to the path of your update script like ``/path/to/update/script.sh``.
If you want no action to be taken (because you have more than one branch and want automatic
deployment only for one of them), leave it empty.

For clarity, describe what happened under **endpoint.description**.
It will be used as subject in notification emails. This is especially
helpful if you have multiple endpoints.

The email will contain all the messages of the pushed commits and the output of your deployment script.

If you want the person who pushed the changes to receive the deployment notification as well,
set **endpoint.cc-pusher** to *"true"*. This involves only people with write permissions to your repository,
but still think about it as the script output may contain sensitive details.

### 4. Set up Webhooks in your GitHub repository
On the settings page of your GitHub repository, go to **Webhooks** and
enter the public url of your ``github.php``. Set the content type to "application/json" and enter
the same secret you stored in the config in **endpoint.secret**.

If things don't work as expected, GitHub has nice debugging features for Webhooks:
You can go to **Recent Deliveries** and check what exactly was sent and how our script responded.
You can also re-send any payload with the **Redeliver** button.

## Limitations
We use the PHP [exec](https://www.php.net/manual/en/function.exec.php) function that synchronously calls the deploy script,
captures the output and sends it via email after the deployment finishes.
If deploy scripts take longer than just some seconds, the following problems arise:
- [GitHub expects webhook handlers to respond within 10 seconds](https://docs.github.com/en/rest/guides/best-practices-for-integrators#favor-asynchronous-work-over-synchronous). If your handler takes longer, you'll see a warning
  in the "Recent Deliveries" tab of your webhook. That's not a real issue though because your script still runs
  everything on your server and sends an email to you with the result.
- If the deployment takes even longer you may run into real problems with your web server: If the script runs longer
  than what is configured in ``max_execution_time`` in ``php.ini`` (see [set_time_limit()](https://www.php.net/manual/de/function.set-time-limit.php)), Apache will silently kill it. A good web hoster will let you increase ``max_execution_time``
  but don't do that, you should rather look for a better, asynchronous deployment solution.

### TODO
Support GitLab as well. Should be [fairly similar](https://docs.gitlab.com/ee/user/project/integrations/webhooks.html).
The HTTP header for checking the secret is called ``X-Gitlab-Token``. Is the payload structure the same?
Caution: [GitLab resends the webhook if it times out](https://docs.gitlab.com/ee/user/project/integrations/webhooks.html#http-responses-for-your-endpoint) which could lead to indefinite loops of triggering the deployment if the deploy script always takes some time.

## Credits
It is based on https://gist.github.com/gka/4627519.

One important change is that we're not checking anymore whether the sending IP address belongs to GitHub
but instead we're using a secret
[as recommended](https://docs.github.com/en/developers/webhooks-and-events/webhooks/securing-your-webhooks).
Besides, code quality was improved in many places.