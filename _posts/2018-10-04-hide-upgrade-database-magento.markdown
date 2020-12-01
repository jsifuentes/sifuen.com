---
layout: post
title:  "How to hide the \"Please upgrade your database\" exception message"
date:   2018-10-04 12:31:00 -0500
categories: magento-2
permalink: /magento-2/how-to-hide-the-please-upgrade-your-database-exception-message
---
I find myself switching between development branches all the time. Since modules are sometimes updated in other branches, I'll always find that dreaded message on my screen.

```Please upgrade your database: Run "bin/magento setup:upgrade" from the Magento root directory. 
The following modules are outdated:
Vendor_Module schema: current version - 1.0.1, required version - 1.0.0
Vendor_Module data: current version - 1.0.1, required version - 1.0.0```

Except running `setup:upgrade` won't fix this issue because the current version is actually higher than the required version. So now you have to do the annoying task of opening your database client, navigating to setup_upgrade, finding your module, and adjusting the version on both the schema and data version columns.

So I made a module to suppress the exception so that I can keep working on my local environment without worrying about the module versions. You can find it here: <a href="https://github.com/jsifuentes/module-suppress-out-of-date-db" rel="noopener" target="_blank">https://github.com/jsifuentes/module-suppress-out-of-date-db</a>. The module is pretty easy to use. Once you install it, you can just use the following console command to toggle the errors on or off: `php bin/magento dev:db:toggle-out-of-date-errors`.

I also didn't want to be completely in the dark about whether my modules were out of date, so I moved the exception message to show in the system messages tray in the backend.

<img src="https://i.imgur.com/9PECODm.png" alt="Exception message showing in the system messages tray in the backend." />

You can visit a page to see a list of out of date modules.

<img src="https://i.imgur.com/LNqbhhQ.png" alt="Out of date modules list page" />

Hope someone finds this module useful!
