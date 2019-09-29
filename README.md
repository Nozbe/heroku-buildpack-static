# heroku-buildpack-static
**NOTE**: This buildpack is in an experimental OSS project.

This is a buildpack for handling static sites and single page web apps.

For a guide, read the [Getting Started with Single Page Apps on Heroku](https://gist.github.com/hone/24b06869b4c1eca701f9).

## Features
* serving static assets
* gzip on by default
* error/access logs support in `heroku logs`
* custom [configuration](#configuration)

## Deploying
The `static.json` file is required to use this buildpack. This file handles all the configuration described below.

1. Set the app to this buildpack: `$ heroku buildpacks:set https://github.com/heroku/heroku-buildpack-static.git`.
2. Deploy: `$ git push heroku master`

### Configuration
You can configure different options for your static application by writing a `static.json` in the root folder of your application.

#### Root
This allows you to specify a different asset root for the directory of your application. For instance, if you're using ember-cli, it naturally builds a `dist/` directory, so you might want to use that intsead.

```json
{
  "root": "dist/"
}

```

By default this is set to `public_html/`

#### Default Character Set
This allows you to specify a character set for your text assets (HTML, Javascript, CSS, and so on). For most apps, this should be the default value of "UTF-8", but you can override it by setting `encoding`:

```json
{
    "encoding": "US-ASCII"
}
```

#### Clean URLs
For SEO purposes, you can drop the `.html` extension from URLs for say a blog site. This means users could go to `/foo` instead of `/foo.html`.


```json
{
  "clean_urls": true
}
```

By default this is set to `false`.


#### Logging
You can disable the access log and change the severity level for the error log.

```json
{
  "logging": {
    "access": false,
    "error": "warn"
  }
}
```

By default `access` is set to `true` and `error` is set to `error`.

The environment variable `STATIC_DEBUG` can be set, to override the `error` log level to `error`.


#### Custom Routes
You can define custom routes that combine to a single file. This allows you to preserve routing for a single page web application. The following operators are supported:

* `*` supports a single path segment in the URL. In the configuration below, `/baz.html` would match but `/bar/baz.html` would not.
* `**` supports any length in the URL.  In the configuration below, both `/route/foo` would work and `/route/foo/bar/baz`.

```json
{
  "routes": {
    "/*.html": "index.html",
    "/route/**": "bar/baz.html"
  }
}
```

##### Browser history and asset files
When serving a single page app, it's useful to support wildcard URLs that serves the index.html file, while also continuing to serve JS and CSS files correctly. Route ordering allows you to do both:

```json
{
  "routes": {
    "/assets/*": "/assets/",
    "/**": "index.html"
  }
}
```

#### Custom Redirects
With custom redirects, you can move pages to new routes but still preserve the old routes for SEO purposes. By default, we return a `301` status code, but you can specify the status code you want.

```json
{
  "redirects": {
    "/old/gone/": {
      "url": "/",
      "status": 302
    }
  }
}
```

##### Interpolating Env Var Values
It's common to want to be able to test the frontend against various backends. The `url` key supports environment variable substitution using `${ENV_VAR_NAME}`. For instance, if there was a staging and production Heroku app for your API, you could setup the config above like the following:

```json
{
  "redirects": {
    "/old/gone/": {
      "url": "${NEW_SITE_DOMAIN}/new/here/"
    }
  }
}
```

Then using the [config vars](https://devcenter.heroku.com/articles/config-vars), you can point the frontend app to the appropriate backend. To match the original proxy setup:

```bash
$ heroku config:set NEW_SITE_DOMAIN="https://example.herokapp.com"
```

#### Custom Error Pages
You can replace the default nginx 404 and 500 error pages by defining the path to one in your config.

```json
{
  "error_page": "errors/error.html"
}
```

#### HTTPS Only

You can redirect all HTTP requests to HTTPS.

```
{
  "https_only": true
}
```

#### Basic Authentication

You can enable Basic Authentication so all requests require authentication.

```
{
  "basic_auth": true
}
```

This will generate `.htpasswd` using environment variables `BASIC_AUTH_USERNAME` and `BASIC_AUTH_PASSWORD` if they are present. Otherwise it will use a standard `.htpasswd` file present in the `app` directory.

Passwords set via `BASIC_AUTH_PASSWORD` can be generated using OpenSSL or Apache Utils. For instance: `openssl passwd -apr1`.

#### Proxy Backends
For single page web applications like Ember, it's common to back the application with another app that's hosted on Heroku. The down side of separating out these two applications is that now you have to deal with CORS. To get around this (but at the cost of some latency) you can have the static buildpack proxy apps to your backend at a mountpoint. For instance, we can have all the api requests live at `/api/` which actually are just requests to our API server.

```json
{
  "proxies": {
    "/api/": {
      "origin": "https://hone-ember-todo-rails.herokuapp.com/"
    }
  }
}
```

##### Interpolating Env Var Values
It's common to want to be able to test the frontend against various backends. The `origin` key supports environment variable substitution using `${ENV_VAR_NAME}`. For instance, if there was a staging and production Heroku app for your API, you could setup the config above like the following:

```json
{
  "proxies": {
    "/api/": {
      "origin": "https://${API_APP_NAME}.herokuapp.com/"
    }
  }
}
```

Then using the [config vars](https://devcenter.heroku.com/articles/config-vars), you can point the frontend app to the appropriate backend. To match the original proxy setup:

```bash
$ heroku config:set API_APP_NAME="hone-ember-todo-rails"
```

#### Custom Headers
Using the headers key, you can set custom response headers. It uses the same operators for pathing as [Custom Routes](#custom-routes).

```json
{
  "headers": {
    "/": {
      "Cache-Control": "no-store, no-cache"
    },
    "/assets/**": {
      "Cache-Control": "public, max-age=512000"
    },
    "/assets/webfonts/*": {
      "Access-Control-Allow-Origin": "*"
    }
  }
}
```

For example, to enable CORS for all resources, you just need to enable it for all routes like this:

```json
{
  "headers": {
    "/**": {
      "Access-Control-Allow-Origin": "*"
    }
  }
}
```

##### Precedence
When there are header conflicts, the last header definition always wins. The headers do not get appended. For example,

```json
{
  "headers": {
    "/**": {
      "X-Foo": "bar",
      "X-Bar": "baz"
    },
    "/foo": {
      "X-Foo": "foo"
    }
  }
}
```

when accessing `/foo`, `X-Foo` will have the value `"foo"` and `X-Bar` will not be present.

#### Build command

You can add a `build` command that will run at compile time so that every time this project is compiled it will automatically build the static site.

```
{
  "build": "gatsby build"
}
```


### Route Ordering

* HTTPS redirect
* Root Files
* Clean URLs
* Proxies
* Redirects
* Custom Routes
* 404

## Testing
For testing we use Docker to replicate Heroku locally. You'll need to have [it setup locally](https://docs.docker.com/installation/). We're also using rspec for testing with Ruby. You'll need to have those setup and install those deps:

```sh
$ bundle install
```

To run the test suite just execute:

```sh
$ bundle exec rspec
```

### Structure
To add a new test, add another example inside `spec/simple_spec.rb` or create a new file based off of `spec/simple_spec.rb`. All the example apps live in `spec/fixtures`.

When writing a test, `BuildpackBuilder` creates the docker container we need that represents the heroku cedar-14 stack. `AppRunner.new` takes the name of a fixture and mounts it in the container built by `BuildpackBuilder` to run tests against. The `AppRunner` instance provides convenience methods like `get` that just wrap `net/http` for analyzing the response.

### Boot2docker

If you are running docker with boot2docker, the buildpack will automatically send tests to the right ip address.
You need to forward the docker's port 3000 to the virtual machine's port though.

```
VBoxManage modifyvm "boot2docker-vm" --natpf1 "tcp-port3000,tcp,,3000,,3000";
```





Apache License
                           Version 2.0, January 2004
                        https://www.apache.org/licenses/

   TERMS AND CONDITIONS FOR USE, REPRODUCTION, AND DISTRIBUTION

   1. Definitions.

      "License" shall mean the terms and conditions for use, reproduction,
      and distribution as defined by Sections 1 through 9 of this document.

      "Licensor" shall mean the copyright owner or entity authorized by
      the copyright owner that is granting the License.

      "Legal Entity" shall mean the union of the acting entity and all
      other entities that control, are controlled by, or are under common
      control with that entity. For the purposes of this definition,
      "control" means (i) the power, direct or indirect, to cause the
      direction or management of such entity, whether by contract or
      otherwise, or (ii) ownership of fifty percent (50%) or more of the
      outstanding shares, or (iii) beneficial ownership of such entity.

      "You" (or "Your") shall mean an individual or Legal Entity
      exercising permissions granted by this License.

      "Source" form shall mean the preferred form for making modifications,
      including but not limited to software source code, documentation
      source, and configuration files.

      "Object" form shall mean any form resulting from mechanical
      transformation or translation of a Source form, including but
      not limited to compiled object code, generated documentation,
      and conversions to other media types.

      "Work" shall mean the work of authorship, whether in Source or
      Object form, made available under the License, as indicated by a
      copyright notice that is included in or attached to the work
      (an example is provided in the Appendix below).

      "Derivative Works" shall mean any work, whether in Source or Object
      form, that is based on (or derived from) the Work and for which the
      editorial revisions, annotations, elaborations, or other modifications
      represent, as a whole, an original work of authorship. For the purposes
      of this License, Derivative Works shall not include works that remain
      separable from, or merely link (or bind by name) to the interfaces of,
      the Work and Derivative Works thereof.

      "Contribution" shall mean any work of authorship, including
      the original version of the Work and any modifications or additions
      to that Work or Derivative Works thereof, that is intentionally
      submitted to Licensor for inclusion in the Work by the copyright owner
      or by an individual or Legal Entity authorized to submit on behalf of
      the copyright owner. For the purposes of this definition, "submitted"
      means any form of electronic, verbal, or written communication sent
      to the Licensor or its representatives, including but not limited to
      communication on electronic mailing lists, source code control systems,
      and issue tracking systems that are managed by, or on behalf of, the
      Licensor for the purpose of discussing and improving the Work, but
      excluding communication that is conspicuously marked or otherwise
      designated in writing by the copyright owner as "Not a Contribution."

      "Contributor" shall mean Licensor and any individual or Legal Entity
      on behalf of whom a Contribution has been received by Licensor and
      subsequently incorporated within the Work.

   2. Grant of Copyright License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      copyright license to reproduce, prepare Derivative Works of,
      publicly display, publicly perform, sublicense, and distribute the
      Work and such Derivative Works in Source or Object form.

   3. Grant of Patent License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      (except as stated in this section) patent license to make, have made,
      use, offer to sell, sell, import, and otherwise transfer the Work,
      where such license applies only to those patent claims licensable
      by such Contributor that are necessarily infringed by their
      Contribution(s) alone or by combination of their Contribution(s)
      with the Work to which such Contribution(s) was submitted. If You
      institute patent litigation against any entity (including a
      cross-claim or counterclaim in a lawsuit) alleging that the Work
      or a Contribution incorporated within the Work constitutes direct
      or contributory patent infringement, then any patent licenses
      granted to You under this License for that Work shall terminate
      as of the date such litigation is filed.

   4. Redistribution. You may reproduce and distribute copies of the
      Work or Derivative Works thereof in any medium, with or without
      modifications, and in Source or Object form, provided that You
      meet the following conditions:

      (a) You must give any other recipients of the Work or
          Derivative Works a copy of this License; and

      (b) You must cause any modified files to carry prominent notices
          stating that You changed the files; and

      (c) You must retain, in the Source form of any Derivative Works
          that You distribute, all copyright, patent, trademark, and
          attribution notices from the Source form of the Work,
          excluding those notices that do not pertain to any part of
          the Derivative Works; and

      (d) If the Work includes a "NOTICE" text file as part of its
          distribution, then any Derivative Works that You distribute must
          include a readable copy of the attribution notices contained
          within such NOTICE file, excluding those notices that do not
          pertain to any part of the Derivative Works, in at least one
          of the following places: within a NOTICE text file distributed
          as part of the Derivative Works; within the Source form or
          documentation, if provided along with the Derivative Works; or,
          within a display generated by the Derivative Works, if and
          wherever such third-party notices normally appear. The contents
          of the NOTICE file are for informational purposes only and
          do not modify the License. You may add Your own attribution
          notices within Derivative Works that You distribute, alongside
          or as an addendum to the NOTICE text from the Work, provided
          that such additional attribution notices cannot be construed
          as modifying the License.

      You may add Your own copyright statement to Your modifications and
      may provide additional or different license terms and conditions
      for use, reproduction, or distribution of Your modifications, or
      for any such Derivative Works as a whole, provided Your use,
      reproduction, and distribution of the Work otherwise complies with
      the conditions stated in this License.

   5. Submission of Contributions. Unless You explicitly state otherwise,
      any Contribution intentionally submitted for inclusion in the Work
      by You to the Licensor shall be under the terms and conditions of
      this License, without any additional terms or conditions.
      Notwithstanding the above, nothing herein shall supersede or modify
      the terms of any separate license agreement you may have executed
      with Licensor regarding such Contributions.

   6. Trademarks. This License does not grant permission to use the trade
      names, trademarks, service marks, or product names of the Licensor,
      except as required for reasonable and customary use in describing the
      origin of the Work and reproducing the content of the NOTICE file.

   7. Disclaimer of Warranty. Unless required by applicable law or
      agreed to in writing, Licensor provides the Work (and each
      Contributor provides its Contributions) on an "AS IS" BASIS,
      WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
      implied, including, without limitation, any warranties or conditions
      of TITLE, NON-INFRINGEMENT, MERCHANTABILITY, or FITNESS FOR A
      PARTICULAR PURPOSE. You are solely responsible for determining the
      appropriateness of using or redistributing the Work and assume any
      risks associated with Your exercise of permissions under this License.

   8. Limitation of Liability. In no event and under no legal theory,
      whether in tort (including negligence), contract, or otherwise,
      unless required by applicable law (such as deliberate and grossly
      negligent acts) or agreed to in writing, shall any Contributor be
      liable to You for damages, including any direct, indirect, special,
      incidental, or consequential damages of any character arising as a
      result of this License or out of the use or inability to use the
      Work (including but not limited to damages for loss of goodwill,
      work stoppage, computer failure or malfunction, or any and all
      other commercial damages or losses), even if such Contributor
      has been advised of the possibility of such damages.

   9. Accepting Warranty or Additional Liability. While redistributing
      the Work or Derivative Works thereof, You may choose to offer,
      and charge a fee for, acceptance of support, warranty, indemnity,
      or other liability obligations and/or rights consistent with this
      License. However, in accepting such obligations, You may act only
      on Your own behalf and on Your sole responsibility, not on behalf
      of any other Contributor, and only if You agree to indemnify,
      defend, and hold each Contributor harmless for any liability
      incurred by, or claims asserted against, such Contributor by reason
      of your accepting any such warranty or additional liability.

   END OF TERMS AND CONDITIONS

   APPENDIX: How to apply the Apache License to your work.

      To apply the Apache License to your work, attach the following
      boilerplate notice, with the fields enclosed by brackets "[]"
      replaced with your own identifying information. (Don't include
      the brackets!)  The text should be enclosed in the appropriate
      comment syntax for the file format. We also recommend that a
      file or class name and description of purpose be included on the
      same "printed page" as the copyright notice for easier
      identification within third-party archives.

   Copyright [2019] [Rolando Gopez Lacuata]

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       https://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
