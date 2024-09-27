
# Change Log
All notable changes to this project will be documented in this file.
 
The format is based on [Keep a Changelog](http://keepachangelog.com/)
and this project adheres to [Semantic Versioning](http://semver.org/).

## [4.1.0] - 2024-09-27
 
Adds ability to disable sending default parse server verification email and password reset email.
Parse forces you to either disable auto login on signup with email verification or auto login on signup without email verification.
For some special cases, we may want to send those emails ourselves or allow users to sign up and use other means to verify them other than sending the default verification email. This change allows you to do just that.

### Added

- Ability to switch of sending verification and password reset emails.

 
## [4.0.1] - 2024-09-18
 
Fixes missing parenthesis in conditional check which throws an error when constructor `external=true` and templates is not passed in constructor or is and empty object. When external is set to true, templates is not used by this library.
 
### Fixed

Fixes Error: `ApiMailAdapter: templates are missing or invalid` when `external=true` and templates is not passed in contructor or is empty.
 
## [4.0.0] - 2024-09-17
  
Initial release based on 4.0.0 fork of [The Parse Server Mail API Adapter](https://github.com/parse-community/parse-server-api-mail-adapter).
 
### Changed

- Production ready README.md and package version
 
## [4.0.0-rc1] - 2024-09-16
 
### Added
- Added special contructor field `external` which is a boolean set to false by default. When this field is set to true, it templates field become optional and may be completely bypassed.
- Makes it possible for apiCallback to receive complete set of parameters passed to the original `Parse.Cloud.sendEmail` call. This wasn't possible in the original version.
