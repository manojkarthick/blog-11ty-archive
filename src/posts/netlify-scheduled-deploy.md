---
id: "post/14"
title: "Deploy your blog on a schedule using Netlify Scheduled functions"
date: 2022-02-21 02:00:00
tags:
- netlify
- automation
desc: "Using Netlify scheduled functions (cron jobs) to rebuild your website periodically"
url: https://manojkarthick.netlify.app
img: https://manojkarthick.netlify.app/img/netlify-scheduled-function.png
img_alt: "Sony WH-1000XM4 Headphones"
type: summary_large_image
twitter: manojkarthick
name: Manoj Karthick
---

## Background

<!-- Excerpt Start -->

Earlier this month, [Netlify][1] announced their [acquisition][11] of [Quirrel][2] and subsequently announced the availability of [Netlify Scheduled functions][3] in Beta. Scheduled functions are serverless cron jobs that run periodically alongside your Netlify site. I just started using this feature for my blog to automatically deploy my site. Let's dive in to see how you can too!

![Netlify Scheduled Functions](/img/netlify-scheduled-function.png)

<!-- Excerpt End -->

As you might already [know][4], this blog is built using [11ty][5] and hosted on Netlify. One of the features I wanted while building the blog was a project [page][6] to list my [GitHub][7] projects. The page also has displays the number of GitHub stars for each of the project.

Instead of hardcoding this information, I used a [11ty filter][8] that [fetches][9] that information from the [GitHub API][12]. Any time the blog is deployed, the project description, language and stars are updated. The disadvantage of this approach is it gets out of date if I haven't published a blog post for a while. To fix this, we can use Netlify Scheduled functions to deploy the website periodically.

## Steps

Before we start, we need to enable Scheduled Functions in Netlify Labs and install the required dependencies. We also need to create a new [build hook][10] for the site to allow us to trigger builds from our function.

Navigate to Netlify > Your site > Site Settings > Build & Deploy > Build hooks. Use the "Add build hook" button to create a new build hook and choose the branch to deploy from. Once done, you should see a URL for your build hook.

![Netlify Build hooks](/img/netlify-build-hook.png)

This URL should be kept as a secret. Let's add this to our environment variables as `SCHEDULED_DEPLOY_HOOK_URL`, so it can be referenced within our function.

![Netlify Environment Variables](/img/netlify-env-vars.png)

The next step is to install your dependencies and add your function under the `./netlify/functions` folder in your project.

```shell
npm install @netlify/functions
npm install node-fetch@2.6
mkdir -pv ./netlify/functions
touch ./netlify/functions/scheduled-deploy.js
```

The function is pretty simple - we register a new handler that sends a `POST` request to the Webhook URL whenever the function is triggered.

```js
// scheduled-deploy.js
const fetch = require("node-fetch");
const WEBHOOK_URL = process.env.SCHEDULED_DEPLOY_HOOK_URL;

const handler = async function(event, context) {
  console.log("Rebuilding site. Received event:", event)
  await fetch(WEBHOOK_URL, {method: 'POST'});

  return {
    statusCode: 200,
  };
};

module.exports.handler = handler;
```

After adding the function, we can specify the cron schedule for the function in our `netlify.toml` file under the project's root directory. The following schedule specifies that the function should every three days at 00:00 UTC. This means that the GitHub project info would be out of date for upto a maximum of three days. We can update the schedule to run it more often depending on how often your projects are updated/starred.

```toml
# netlify.toml
[functions."scheduled-deploy"]
schedule = "0 0 */3 * *"
```

Run `netlify deploy` (or commit to the repo if you have git integration enabled) to deploy your site and your function. The next time your function runs, you should see the site update.

## Bonus

The projects page on my blog displays when the information was last updated. I find this to be a handy way to verify your site's build time and also verify your function ran successfully. As a bonus, it also subtly conveys to the reader that the information on the page might be slightly out of date.

![Project Last updated time](/img/projects-last-updated.png)


[1]: https://www.netlify.com/
[2]: https://dev.to/quirrel/quirrel-is-acquired-and-i-am-joining-netlify-dha
[3]: https://github.com/netlify/labs/blob/main/features/scheduled-functions/documentation/README.md
[4]: https://manojkarthick.netlify.app/about/#about-this-blog
[5]: https://www.11ty.dev/
[6]: https://manojkarthick.netlify.app/projects/
[7]: https://github.com/manojkarthick
[8]: https://www.11ty.dev/docs/filters/
[9]: https://github.com/manojkarthick/blog/blob/6a23a15a7eeeecd07aa6dc38d1a403f824c33fde/.eleventy.js#L60
[10]: https://docs.netlify.com/configure-builds/build-hooks/
[11]: https://www.netlify.com/press/netlify-announces-acquisition-of-quirrel-to-extend-serverless-functions-capabilities
[12]: https://docs.github.com/en/rest
