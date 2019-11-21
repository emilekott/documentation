---
title: Fix Broken Links in Drupal
description: Learn how to update Drupal site links so the URL references the correct file path and domain name.
tags: [debugcode]
categories: [drupal]
---

## Update Links Referencing IP:Port

When editing content, links can be inserted that don't reflect the site's domain name. For example, an image URL appears as `https://192.237.142.203:5555/files/cernettes.gif` instead of `https://www.example.com/files/cernettes.gif`.

The link may work at first, but will eventually break when your application container’s IP address changes due to the nature of Pantheon’s cloud-based infrastructure.

### Drupal 7 Configure $base_url
Sometimes, the `$base_url`, if not explicitly set, is the URL from which the asset was loaded. If a module cache is populated on a Drush Cron run from the CLI, this can be the app-container IP which is dynamic and will ultimately become a broken link. The current resolution is to set `$base_url` per environment in `settings.php` and clear caches. There are some notes about this in the [Pathologic documentation](https://www.drupal.org/node/257026).

If you look through the Media module issue queue, you’ll see that other users have experienced the same or very similar issues with WYSIWYG-inserted images having their base URLs cached. The IP address is coming from a cached reference to the image’s location as generated by the media filter. Dave Reid (the Media module maintainer) has explained it in [this comment](https://drupal.org/node/1660936#comment-6270618).

The recommended solution is setting the `$base_url` in your `settings.php`. This prevents the IP:PORT address from being stored in URLs. Check that you're using the latest version of the module(s).

Here is an example of a code snippet to set the `$base_url` per environment:

```php
if (isset($_ENV['PANTHEON_ENVIRONMENT']) && php_sapi_name() != 'cli') {
  // Redirect to https://$primary_domain/ in the Live environment
  if ($_ENV['PANTHEON_ENVIRONMENT'] === 'live') {
    // Replace www.example.com with your registered domain name.
    $primary_domain = 'www.example.com';
  }
  else {
    // Redirect to HTTPS on every Pantheon environment.
    $primary_domain = $_SERVER['HTTP_HOST'];
  }
  $base_url = 'https://'. $primary_domain;
  if ($_SERVER['HTTP_HOST'] != $primary_domain
      || !isset($_SERVER['HTTP_USER_AGENT_HTTPS'])
      || $_SERVER['HTTP_USER_AGENT_HTTPS'] != 'ON' ) {

    // Name transaction "redirect" in New Relic for improved reporting (optional).
    if (extension_loaded('newrelic')) {
      newrelic_name_transaction("redirect");
    }

    header('HTTP/1.0 301 Moved Permanently');
    header('Location: '. $base_url . $_SERVER['REQUEST_URI']);
    exit();
  }
}
```

Clear cache after deploying this code change. All cached IP:PORT references will be wiped out, and repopulated with the correct base URL in the future.


## See Also

[Private Paths](/private-paths/)

[Non-Standard File Paths](/non-standard-file-paths/)