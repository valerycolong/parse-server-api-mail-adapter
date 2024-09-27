# Parse Server API Mail Adapter <!-- omit in toc -->

[![Build Status](https://github.com/valerycolong/parse-server-api-mail-adapter/workflows/ci/badge.svg?branch=release)](https://github.com/valerycolong/parse-server-api-mail-adapter/actions?query=workflow%3Aci+branch%3Arelease)
[![Snyk Badge](https://snyk.io/test/github/valerycolong/parse-server-api-mail-adapter/badge.svg)](https://snyk.io/test/github/parse-community/parse-server-api-mail-adapter)
[![Coverage](https://codecov.io/gh/valerycolong/parse-server-api-mail-adapter/branch/alpha/graph/badge.svg)](https://codecov.io/gh/parse-community/parse-server-api-mail-adapter)
[![auto-release](https://img.shields.io/badge/%F0%9F%9A%80-auto--release-9e34eb.svg)](https://github.com/parse-community/parse-dashboard/releases)

[![Node Version](https://img.shields.io/badge/nodejs-18,_20,_22-green.svg?logo=node.js&style=flat)](https://nodejs.org)

[![npm latest version](https://img.shields.io/npm/v/@zingersystems/parse-server-api-mail-adapter/latest.svg)](https://www.npmjs.com/package/@zingersystems/parse-server-api-mail-adapter)

---

This is a *fork* of The Parse Server API Mail Adapter enables Parse Server to send emails using any 3rd party API with *external or built-in* dynamic templates and localization.

---

# Content <!-- omit in toc -->

- [Installation](#installation)
- [Demo](#demo)
- [Configuration](#configuration)
- [Templates](#templates)
- [Placeholders](#placeholders)
    - [Password Reset and Email Verification](#password-reset-and-email-verification)
- [Localization](#localization)
- [Cloud Code](#cloud-code)
  - [Example](#example)
  - [Parameters](#parameters)
- [Supported APIs](#supported-apis)
  - [Providers](#providers)
    - [Example for Mailgun](#example-for-mailgun)
    - [Example for AWS Simple Email Service](#example-for-aws-simple-email-service)
  - [Custom API](#custom-api)
- [Need help?](#need-help)

# Installation

1. Install adapter:
    ```
    npm install --save @zingersystems/parse-server-api-mail-adapter
    ```
2. Add [template files](#templates) to a subdirectory.
2. Add [adapter configuration](#configuration) to Parse Server.

# Demo

The demo script makes it easy to test adapter configurations and templates by sending emails without Parse Server via the email service provider [Mailgun](https://www.mailgun.com):

1. Create a file `mailgun.json` in the `demo` directory with the following content:
    ```js
    {
        "key": "MAILGUN_API_KEY", // e.g. abc123
        "domain": "MAILGUN_DOMAIN", // e.g. sandbox-example@mailgun.org
        "sender": "SENDER_EMAIL", // e.g. sender@example.com
        "recipient": "RECIPIENT_EMAIL" // e.g. recipient@example.com
    }
    ```
2. Run `node ./demo` to execute the script and send an email.

You can modify the script to use any other API you like or debug-step through the sending process to better understand the adapter internals.

# Configuration

An example configuration to add the API Mail Adapter to Parse Server could look like this:

```js
const Mailgun = require('mailgun.js');
const formData = require('form-data');
const { ApiPayloadConverter } = require('@zingersystems/parse-server-api-mail-adapter');

// Configure mail client
const mailgun = new Mailgun(formData);
const mailgunClient = mailgun.client({ username: 'api', key: process.env.MAILGUN_API_KEY });
const mailgunDomain = process.env.MAILGUN_DOMAIN;

// Configure Parse Server
const server = new ParseServer({
    ...otherServerOptions,

    emailAdapter: {
        module: '@zingersystems/parse-server-api-mail-adapter',
        options: {
            // Set to true if templates are to be processed by a thirdparty api like sendgrid. (Defaults to false).
            external: true,
            // Set whether or not verification email should be sent - defaults to true.
            verificationEmail: true,
            // Set whether or not password reset email should be sent - defaults to true.
            passwordResetEmail: true,
            // The email address from which emails are sent (Required).
            sender: 'sender@example.com',
            // The email templates - (Optional).
            templates: {
                // The template used by Parse Server to send an email for password
                // reset; this is a reserved template name.
                passwordResetEmail: {
                    subjectPath: './files/password_reset_email_subject.txt',
                    textPath: './files/password_reset_email.txt',
                    htmlPath: './files/password_reset_email.html'
                },
                // The template used by Parse Server to send an email for email
                // address verification; this is a reserved template name.
                verificationEmail: {
                    subjectPath: './files/verification_email_subject.txt',
                    textPath: './files/verification_email.txt',
                    htmlPath: './files/verification_email.html'
                },
                // A custom email template that can be used when sending emails
                // from Cloud Code; the template name can be chosen freely; it
                // is possible to add various custom templates.
                customEmail: {
                    subjectPath: './files/custom_email_subject.txt',
                    textPath: './files/custom_email.txt',
                    htmlPath: './files/custom_email.html',
                    // Placeholders are filled into the template file contents.
                    // For example, the placeholder `{{appName}}` in the email
                    // will be replaced the value defined here.
                    placeholders: {
                        appName: "ExampleApp"
                    },
                    // Extras to add to the email payload that is accessible in the
                    // `apiCallback`.
                    extra: {
                        replyTo: 'no-reply@example.com'
                    },
                    // A callback that makes the Parse User accessible and allows
                    // to return user-customized placeholders that will override
                    // the default template placeholders. It also makes the user
                    // locale accessible, if it was returned by the `localeCallback`,
                    // and the current placeholders that will be augmented.
                    placeholderCallback: async ({ user, locale, placeholders }) => {
                        return {
                            phone: user.get('phone');
                        };
                    },
                    // A callback that makes the Parse User accessible and allows
                    // to return the locale of the user for template localization.
                    localeCallback: async (user) => {
                        return user.get('locale');
                    }
                }
            },
            // The asynchronous callback that contains the composed email payload to
            // be passed on to an 3rd party API and optional meta data. The payload
            // may need to be converted specifically for the API; conversion for
            // common APIs is conveniently available in the `ApiPayloadConverter`.
            // Below is an example for the Mailgun client (Required).
            apiCallback: async ({ to, from, subject, text, html, template, user, placeholders, locale, ...rest }) => {
                // Call your mail client here to send.
                const mailgunPayload = ApiPayloadConverter.mailgun({to, from, subject, text, html, template, user, placeholders, ...rest});
                await mailgunClient.messages.create(mailgunDomain, mailgunPayload);
            }
        }
    }
});
```

# Templates

Emails are composed using templates. A template defines the paths to its content files, for example:

```js
templates: {
    exampleTemplate: {
        subjectPath: './files/custom_email_subject.txt',
        textPath: './files/custom_email.txt',
        htmlPath: './files/custom_email.html',
    }
},
```

There are different files for different parts of the email:
- subject (`subjectPath`)
- plain-text content (`textPath`)
- HTML content (`htmlPath`)

See the [templates](https://github.com/valerycolong/parse-server-api-mail-adapter/tree/main/spec/templates) for examples how placeholders can be used.

# Placeholders
Placeholders allow to dynamically insert text into the template content. The placeholder values are filled in according to the key-value definitions returned by the placeholder callback in the adapter configuration.

This is using the [mustache](http://mustache.github.io/mustache.5.html) template syntax. The most commonly used tags are:
- `{{double-mustache}}`: The most basic form of tag; inserts text as HTML escaped by default.
- `{{{triple-mustache}}}`: Inserts text with unescaped HTML, which is required to insert a URL for example.

### Password Reset and Email Verification

By default, the following placeholders are available in the password reset and email verification templates:
- `{{appName}}`: The app name as set in the Parse Server configuration.
- `{{username}}`: The username of the user who requested the email.
- `{{link}}`: The URL to the Parse Server endpoint for password reset or email verification.

# Localization

Localization allows to use a specific template depending on the user locale. To turn on localization for a template, add a `localeCallback` to the template configuration.

The locale returned by `localeCallback` will be used to look for locale-specific template files. If the callback returns an invalid locale or nothing at all (`undefined`), localization will be ignored and the default files will be used.

The locale-specific files are placed in sub-folders with the name of either the whole locale (e.g. `de-AT`), or only the language (e.g. `de`). The locale has to be in format `[language]-[country]` as specified in [IETF BCP 47](https://tools.ietf.org/html/bcp47), e.g. `de-AT`.

Localized files are placed in sub-folders of the given path, for example:
```js
base/
├── example.html         // default file
└── de/                  // de language folder
│   └── example.html     // de localized file
└── de-AT/               // de-AT locale folder
│   └── example.html     // de-AT localized file
```

Files are matched with the user locale in the following order:
1. **Locale** (locale `de-AT` matches file in folder `de-AT`)
2. **Language** (locale `de-AT` matches file in folder `de` if there is no file in folder `de-AT`)
3. **Default** (default file in base folder is returned if there is no file in folders `de-AT` and `de`)

# Cloud Code

Sending an email directly from Cloud Code is possible since Parse Server > 4.5.0. This adapter supports this convenience method.

## Example

If the `user` provided has an email address set, it is not necessary to set a `recipient` because the mail adapter will by default use the mail address of the `user`.

```js
Parse.Cloud.sendEmail({
  templateName: "next_level_email", // Can also use sendgrid template Id 
  placeholders: { gameScore: 100, nextLevel: 2 },
  user: parseUser // user with email address
});
```

## Parameters

| Parameter      | Type         | Optional | Default Value | Example Value               | Description                                                                                    |
|----------------|--------------|----------|---------------|-----------------------------|------------------------------------------------------------------------------------------------|
| `sender`       | `String`     |          | -             | `from@example.com`          | The email sender address; overrides the sender address specified in the adapter configuration. |
| `recipient`    | `String`     |          | -             | `to@example.com`            | The email recipient; if set overrides the email address of the `user`.                         |
| `subject`      | `String`     |          | -             | `Welcome`                   | The email subject.                                                                             |
| `text`         | `String`     |          | -             | `Thank you for signing up!` | The plain-text email content.                                                                  |
| `html`         | `String`     | yes      | `undefined`   | `<html>...</html>`          | The HTML email content.                                                                        |
| `templateName` | `String`     | yes      | `undefined`   | `customTemplate`            | The template name.                                                                             |
| `placeholders` | `Object`     | yes      | `{}`          | `{ key: value }`            | The template placeholders.                                                                     |
| `extra`        | `Object`     | yes      | `{}`          | `{ key: value }`            | Any additional variables to pass to the mail provider API.                                     |
| `user`         | `Parse.User` | yes      | `undefined`   | -                           | The Parse User that the is the recipient of the email.                                         |

# Supported APIs

This adapter supports any REST API by adapting the API payload in the adapter configuration `apiCallback` according to the API specification.

## Providers

For convenience, support for common APIs is already built into this adapter and available via the `ApiPayloadConverter`. The following is a list of currently supported API providers:

- [Mailgun](https://www.mailgun.com)
- [AWS Simple Email Service (AWS JavaScript SDK v3)](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-ses/index.html)

If the provider you are using is not already supported, please feel free to open a PR.

### Example for Mailgun

This is an example for the Mailgun client:

```js
// Configure mail client
const mailgun = require('mailgun.js');
const mailgunClient = mailgun.client({ username: 'api', key: process.env.MAILGUN_API_KEY });
const mailgunDomain = process.env.MAILGUN_DOMAIN;

// Configure Parse Server
const server = new ParseServer({
    ...otherServerOptions,

    emailAdapter: {,
        module: '@zingersystems/parse-server-api-mail-adapter',
        options: {
            ... otherAdapterOptions,

            apiCallback: async ({ to, from, subject, html, placeholders, locale, ...rest }) => {
                // Call mail client send method here.
                const mailgunPayload = ApiPayloadConverter.mailgun({to, from, subject, html, placeholders, locale, ...rest});
                await mailgunClient.messages.create(mailgunDomain, mailgunPayload);
            }
        }
    }
});
```

### Example for AWS Simple Email Service

This is an example for the AWS Simple Email Service client using the AWS JavaScript SDK v3:

```js
// Configure mail client
const { SES, SendEmailCommand } = require('@aws-sdk/client-ses');

const {
  fromInstanceMetadata, // Get credentials via IMDS from the AWS instance (when deployed on AWS instance)
  fromEnv, // Get AWS credentials from environment variables (when testing locally)
} = require('@aws-sdk/credential-providers');

// Get AWS credentials depending on environment
const credentialProvider= process.env.NODE_ENV == 'production' ? fromInstanceMetadata() : fromEnv();
const credentials = await credentialProvider();

const sesClient = new SES({
    credentials,
    region: 'eu-west-1',
    apiVersion: '2010-12-01'
});

// Configure Parse Server
const server = new ParseServer({
    ...otherServerOptions,

    emailAdapter: {
        module: '@zingersystems/parse-server-api-mail-adapter',
        options: {
            ... otherAdapterOptions,

            apiCallback: async ({ to, from, subject, html, placeholders, locale, ...rest }) => {
                const awsSesPayload = ApiPayloadConverter.awsSes({ to, from, subject, html, locale, ...rest});
                const command = new SendEmailCommand(awsSesPayload);
                await sesClient.send(command);
            }
        }
    }
});
```

## Custom API

This is an example of how the API payload can be adapted in the adapter configuration `apiCallback` according to a custom email provider's API specification.

```js
// Configure mail client
const customMail = require('customMail.js');
const customMailClient = customMail.configure({ ... });

// Configure Parse Server
const server = new ParseServer({
    ...otherOptions,

    emailAdapter: {
        module: '@zingersystems/parse-server-api-mail-adapter',
        options: {
            ... otherOptions,

            apiCallback: async ({ to, from, subject, text, html, placeholders, locale, ...rest }) => {
                const customPayload = {
                    customFrom: from,
                    customTo: to,
                    customSubject: subject,
                    customText: text
                };
                await customMailClient.sendEmail(customPayload);
            }
        }
    }
});
```

# Need help?

- Ask on StackOverflow using the [parse-server](https://stackoverflow.com/questions/tagged/parse-server) tag.
- Search through existing [issues](https://github.com/valerycolong/parse-server-api-mail-adapter/issues) or open a new issue.
