---
layout: post
title:  "Using your Google account to sign-in to your Magento backend"
date:   2019-07-20 12:31:00 -0500
categories: magento-2
permalink: /magento-2/using-google-account-sign-in-magento-backend
---
I work at a digital agency with 50-ish employees. People shift between projects pretty often, and we'll end up getting someone assigned on a Magento project who needs access to the backend. The whole process just gets repetitive having to create admin accounts to multiple people, so to offset that work, I created a module that allows users to login to the Magento backend using their Google account.

You can find the module here: [https://github.com/jsifuentes/module-backend-google-sso](https://github.com/jsifuentes/module-backend-google-sso)

Assuming you're in an agency environment, one concern with using a module like this is that an employee could potentially login to a client's backend that they have no business logging into. Thankfully, this module comes with some pretty useful logging tools so that if an employee ever does do this, there will be historical records of when they auto-registered and every instance of them logging in with timestamps.

There's also the scenario that an employee is no longer with the company. What happens to this user's account? Since Magento requires that you have a user record in the database, there's nothing in place to actually delete these employee accounts as they are terminated, but one protection mechanism in the module is the ability to disable password authentication for any account that is auto-registered using Google SSO.

If you have any concerns using this module or any good ideas on what could make this module better, please feel free to go to the Github repository and open an issue. I hope someone finds this module useful!

