---
layout: post
title: "Send Emails from WordPress with Google GMail (OAuth)"
date: 2019-10-12 12:00
categories: Software
tags: [wordpress, gmail]
---

Part 1: [Setting up LEMP and Wordpress with SELinux on CentOS 7]({% post_url 2019-10-08-setting-up-lemp-and-wordpress-with-selinux-on-centos7 %})

Part 2: [LEMP WordPress Next Steps - Certbot and Caching]({% post_url 2019-10-10-LEMP-wordpress-next-steps-certbot-and-caching %})

The next issue I addressed after getting through the first two parts was setting up WordPress with email. After a bit of research, I decided to stick with using my Google account. I briefly looked at Sendinblue and Mailgun, but decided I already had an account with Google and was familiar with them. If at some point I start having connectivity issues or start approaching limits, I will have to revisit the decision.

## Install the Plugin

There are a few different plugins for connecting WordPress with Gmail. I decided to go with [WP Mail SMTP by WPForms](https://wordpress.org/plugins/wp-mail-smtp/) because it was the highest rated one at the time of writing and supported OAuth tokens. Some of their installation documentation was a bit dated (screenshots no longer matched), but the process was straight forward. For this reason, I’ll avoid the screenshots and describe the process at a high level.

After the plugin is installed and activated, head on over to its settings page and complete the following fields:

* **From Email**: Enter your Gmail address
* **From Name**: Enter the name you want to use in the from line of the email
* **Mailer**: Select Gmail

Scroll down and save the URL in the **Authorized redirect URI** box, you will need this in the next step.


## Setup a Google App with Gmail API

In this next section with are going to setup a Google App, add the Gmail API to it, authorized it, and create an OAuth token that WordPress can use to send emails. Head over to <https://console.developers.google.com/apis/dashboard> and login with your Gmail account.

1. **Create a new project**

    If this is your first project, you should see a blue CREATE link on the right side. If its not your first project, click the dropdown on the top left (showing your currently active project) and click “New Project” on the window.

1. **Name your project**

    On the “New Project” screen, give your project a name and click the [CREATE] button.

1. **Enable API**

    After your project has been created, add the Gmail API to it. Click the “Enable APIs and Services” link. Search for “Gmail” and select the “Gmail API” card. Finally, click the [ENABLE] button on the Gmail API screen.

1. **Create your credentials**

    On the Gmail API screen, click the “Credentails” link on the left navigation bar. On the next screen, click the [CONFIGURE CONSENT SCREEN] button.

1. **Complete OAuth consent screen**

    * **Application Name**: Give your application a name (e.g. My WordPress Blog )
    * Can skip the application logo
    * Leave the default scopes
    * **Authorized Domains**: Enter your top level domain (e.g. example.com) without HTTP/S or `www.`
    * For **Application Homepage link**, **Application Privacy Policy link**, and **Application Terms of Service link**, put in your domain (e.g. www.example.com)
    * Click the [Save] button

1. **Create OAuth client**

    * On the next page select create credentials and select **OAuth client ID**.
    * **Application Type**: select `Web Application`
    * **Name**: Enter a name for your client (e.g. `My WordPress`)
    * **Authorized JavaScript origins**: Enter your website (e.g. `https://example.com` and `https://www.example.com`). Make sure to press enter after each one so they are added to the list at separate elements.
    * **Authorized redirect URIs**: Finally we get to use that URI we saved from the plugin settings page above. Paste that URI here. Again, make sure you hit enter to add it to the list.
    * Click the [Create] button

1. **Configure plugin**
    * Copy the **Client ID** and **Client Secret** to the WP Mail SMTP plugin configuration page.
    * On the plugin settings page, click the [Allow plugin to send emails using your Google account] button under the **Authorization** section.
    * The page that loads up will likely show a warning that the app is not verified. Expand the page and click the link to allow and continue.
    * Login to your Gmail account and authorize the plugin to interact with the Gmail API.
    * You should now be able to scroll to the top of the plugin settings page and click the **Email Test** tab. Click the [Send Email] button and check your Gmail.