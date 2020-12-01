---
layout: post
title:  "The dangers of only relying on FILTER_VALIDATE_URL when downloading URLs"
date:   2018-03-24 10:29:00 -0500
categories: php security
permalink: /php/the-dangers-of-only-relying-on-filter_validate_url-when-downloading-urls
---
If you have ever wanted to download a file from an external website from your PHP application, you likely used <a href="http://php.net/file_get_contents" target="_blank">file_get_contents</a> or <a href="http://php.net/manual/en/curl.examples-basic.php" target="_blank">cURL</a>.

If the URL your application needs to download is coming from the user, then it's extremely important that you're validating the input that goes into file_get_contents() or curl_setopt() is a valid URL. The way you can accomplish this is by using <a href="http://php.net/manual/en/function.filter-var.php" target="_blank">filter_var()</a> and the <a href="http://php.net/manual/en/filter.filters.validate.php" target="_blank">FILTER_VALIDATE_URL</a> constant.

Let's create a scenario. You are writing a website that will allow a user to view a website's HTML by typing in the URL. Using FILTER_VALIDATE_URL and cURL, you create the following code.

{% highlight php %}
<?php

if (!isset($_GET['url'])) {
    die("URL not provided.");
}

$url = $_GET['url'];

if (filter_var($url, FILTER_VALIDATE_URL) === false) {
    die("URL is invalid: " . $url);
}

$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1); 
$response = curl_exec($ch);
$errno = curl_errno($ch);
curl_close($ch);

if ($errno !== 0) {
    die("An error occurred while retrieving the URL. " . $errno);
}

echo $response;
{% endhighlight %}

You're validating the user's input is a URL, then downloading the URL and giving the contents to the user. So what's the problem here? The problem here is the different things a valid URL can be. A valid URL doesn't have to be HTTP or HTTPS. Here's a small list of URLs that pass under FILTER_VALIDATE_URL but can still cause harm to your application including, but not limited to, <a href="http://www.acunetix.com/blog/articles/server-side-request-forgery-vulnerability/" target="_blank">SSRF</a> and <a href="https://www.owasp.org/index.php/Testing_for_Local_File_Inclusion" target="_blank">local file inclusion</a>.

<ul>
    <li>ftp://user:pass@host/file.txt</li>
    <li>ldap://host/</li>
    <li>telnet://host/</li>
    <li>file:///etc/passwd/</li>
</ul>

If a user hits that script with the parameters `?url=file:///etc/passwd`, all your checks will pass, but your application will expose your /etc/passwd file. The user can even request your PHP files, given they know the full path to them. This can lead to exposed credentials and secrets.

Your application also becomes a proxy to access and download files from FTP servers. This can become a hotbed for hackers and malicious users that may want to test a target without it coming from their computer.

Be paranoid. Validate correctly. If you validate the URL begins with the HTTP or HTTPS scheme, the number of malicious things a user can request go down significantly.

{% highlight php %}
<?php
if (filter_var($url, FILTER_VALIDATE_URL) === false || (stripos($url, "http://") !== 0 && stripos($url, "https://") !== 0)) {
    die("URL is invalid: " . $url);
}
{% endhighlight %}