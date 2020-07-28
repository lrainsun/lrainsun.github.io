---
layout: post
title:  "openstack glance release"
date:   2020-07-28 23:00:00 +0800
categories: Openstack
tags: Openstack-Glance
excerpt: openstack glance release
mathjax: true
typora-root-url: ../
---

# openstack glance release

# New Features

## Summary

- Image Visibility, public / private => private / public / shared(default) / community
- multi store support
- Image API v1 is DEPRECATED, using API v2 now, no is_public in v2

## Ocata

- The identifier `ploop` has been added to the list of supported disk formats in Glance. The respective configuration option has been updated and the default list shows `ploop` as a supported format.
- Image Visibility, public / private => private / public / shared(default) / community
- The database migration engine used by Glance for database upgrades has been changed from *SQLAlchemy Migrate* to *Alembic* in this release. The change in migration engine has been undertaken in order to enable zero-downtime database upgrades, which are part of the effort to implement rolling upgrades for Glance (scheduled for the Pike release).
- The deprecation path for the configuration option `show_multiple_locations` has been changed because the mitigation instructions for [OSSN-0065](https://wiki.openstack.org/wiki/OSSN/OSSN-0065) refer to this option. It is now subject to removal on or after the Pike release. The help text for this option has been updated accordingly.

## Pike

- The image-list call to the Images v2 API now recognizes a `protected` query-string parameter. This parameter accepts only two values: either `true` or `false`. The filter is case-sensitive. Any other value will result in a 400 response to the request. See the [protected filter specification](https://specs.openstack.org/openstack/glance-specs/specs/pike/implemented/glance/add-protected-filter.html) document for details.
- Glance is now packaged with a WSGI script entrypoint, enabling it to be run as a WSGI application hosted by a performant web server. See [Running Glance in HTTPD](https://docs.openstack.org/glance/latest/admin/apache-httpd.html) in the Glance documentation for details. There are some limitations with this method of deploying Glance and we do not recommend its use in production environments at this time. See the [Known Issues](https://docs.openstack.org/releasenotes/glance/pike.html#known-issues) section of this document for more information.

## Queens

- A new interoperable image import method, `web-download` is introduced. This method allows an end user to import an image from a remote URL. The image data is retrieved from the URL and stored in the Glance backend. (In other words, this is a copy-from operation.) The `web-download` import method replaces the copy-from functionality that was available in the Image API v1 but previously absent from v2. This note is a gentle reminder that the Image API v1 is DEPRECATED and will be removed from Glance during the Rocky development cycle. The Queens release of Glance (16.x.x) is the final version in which you can expect to find the Image API v1.
  When `enable_image_import` is True, a new import-method, `web-download` is available. (In Pike, only `glance-direct` was offered.) Which import-methods you offer can be configured using the `enabled_import_methods` option in the `glance-api.conf` file.
- In order to check the current state of your database upgrades, you may run the command `glance-manage db check`. This will inform you of any outstanding actions you have left to take.
- The default value for the API configuration option `workers` was previously the number of CPUs available. It has been changed to be the min of {number of CPUs, 8}. Any value set for that option, of course, is honored. See Bug [1748916](https://code.launchpad.net/bugs/1748916) for details.
- With the introduction of the `web-download` import method, we consider the Image Service v2 API to have reached feature parity with the DEPRECATED v1 API in all important respects. Support for the Image Service API v1 ends with the Queens release. The [v1 API was deprecated in Newton](http://git.openstack.org/cgit/openstack/glance/commit/?id=63e6dbb1eb006758fbcf7cae83e1d2eacf46b4ab) and will be removed from the codebase at the beginning of the Rocky development cycle. Please plan appropriately.

## Rocky

- Rocky development cycle marks long waited milestone on Glance work. The Images API v1 which has been deprecated for years is finally removed and not available at all in Glance version 17.0.0 forward.
- This release provides an EXPERIMENTAL implementation of the Glance spec [Multi-Store Backend Support](https://specs.openstack.org/openstack/glance-specs/specs/rocky/implemented/glance/multi-store.html), which allows an operator to configure multiple backing stores so that end users may direct image data to be stored in a specific backend. See [Multi Store Support](https://docs.openstack.org/glance/latest/admin/multistores.html) in the Glance Administration Guide for more information.
- As Image Import will be always enabled, care needs to be taken that it is configured properly from this release forward. The ‘enable_image_import’ option is silently ignored.

## Stein

- Re-introducing cache-manage utility. In Rocky the Images API v1 dependent glance-cache-manage utility was removed while removing the v1 endpoints. Stein release introduces the command refactored to utilize the Images API version 2.
- You can now list all images that are available to you. Use the ‘all’ visibility option.
- [[Community Goal](https://governance.openstack.org/tc/goals/stein/upgrade-checkers.html)] Support has been added for developers to write pre-upgrade checks. Operators can run these checks using `glance-status upgrade check`. This allows operators to be more confident when upgrading their deployments by having a tool that automates programmable checks against the deployment configuration or dataset.

## Train

- Stabilization of multi-store feature; from Train onwards multi-store is considered stable feature in glance, glance_store and python-glanceclient. The community encourages everyone to adopt this new way of configuring backend stores at earliest convenience as the old configuration options are deprecated for removal to ease the burden of maintaining underlying code. Users are able to select the store they want their images to be stored during import process.
- To support the Block Storage service (Cinder) upload-volume-to-image action when the volume is an encrypted volume type, when such an image is deleted, Glance will now contact the OpenStack Key Management service (Barbican) and request it to delete the associated encryption key. Two extra properties must be set on the image for this to work: `cinder_encryption_key_id` (whose value is the identifier in the OpenStack Key Management service for the encryption key used to encrypt the volume) and `cinder_encryption_key_deletion_policy` (whose value may be either `on_image_deletion` or `do_not_delete`).

## Ussuri

- Added new import method `copy-image` which will copy existing image into multiple stores.
- As part of the multi-store efforts this release introduces deletion from single store. Through new ‘/v2/stores’ endpoint the API user can request image to be deleted from single store instead of deleting the whole image. This feature can be used to clean up store metadata in cases where the image data has for some reason disappeared from the store already, except 410 Gone HTTP response.
- Add ability to import image into multiple stores during [interoperable image import process](https://developer.openstack.org/api-ref/image/v2/#interoperable-image-import).