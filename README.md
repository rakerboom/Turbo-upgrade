# Turbo Upgrade
Documentation for upgrading [Turbolinks](https://github.com/turbolinks/turbolinks) to [Turbo Drive](https://turbo.hotwired.dev/handbook/drive). Turbo Drive speeds up page loads by avoiding full-page teardowns and rebuilds on every navigation request. Turbo Drive is an improvement on and replacement for Turbolinks.

This guide is based on the official upgrade guide: https://github.com/hotwired/turbo-rails/blob/main/UPGRADING.md

# Why

Turbolinks was deprecated in January of 2021. Keep libraries up-to-date and also:

* Enable the use of [Turbo Frames](https://turbo.hotwired.dev/handbook/frames)
* Enable the use of [Turbo Streams](https://turbo.hotwired.dev/handbook/streams)
* Use of Turbo instead of UJS to perform non-GET requests after a hyperlink click and to add confirmation dialogs before executing an action. This is now the [recommended method mentioned directly on the rails website](https://guides.rubyonrails.org/working_with_javascript_in_rails.html#replacements-for-rails-ujs-functionality)

## Commit 1: Install Turbo Library

* Remove `gem 'turbolinks` and/or `"turbolinks": "X.X.X"`
* `yarn add @hotwired/turbo`
* Replace `import Turbolinks from 'turbolinks'` with `import '@hotwired/turbo';`
* Remove `Turbolinks.start();`

## Commit 2: Search and Replace

Search and replace `turbolinks` with `turbo`(Match case)

Examples:
* `data-turbolinks-action` -> `data-turbo-action`
* `data-turbolinks-eval` -> `data-turbo-eval`
* `turbolinks-cache-control` -> `turbo-cache-control`
* `data-turbolinks` -> `data-turbo`
* `Turbolinks.visit` -> `Turbo.visit`

## Commit 3: Turbo Events

[turbo events](https://turbo.hotwired.dev/reference/events) directly replace [turbolink events](https://github.com/turbolinks/turbolinks#full-list-of-events) Except for:

* `turbolinks:request-start`
* `turbolinks:request-end`

Replace these two events here.

If you are using the [turbo:before-visit](https://turbo.hotwired.dev/handbook/drive#canceling-visits-before-they-start) event you'll need to rename `event.data.url` to `event.detail.url`

## Commit 4: Undocumented Function Calls

If you are using [replaceHistoryWithLocationAndRestorationIdentifier](https://github.com/turbolinks/turbolinks/blob/master/src/controller.ts#L114) replace it with the [turbo equivalent](https://github.com/hotwired/turbo/issues/335).

`turbolinks.controller.replaceHistoryWithLocationAndRestorationIdentifier(href, uuid)` ->
`turbo.session.history.replace(new URL(href), uuid)`

## Commit 5: Shim UJS behavior

In order to have Turbo work with the old-style XMLHttpRequests issued by UJS we added a shim for the redirect behaviour as [outlined in the upgrade guide](https://github.com/hotwired/turbo-rails/blob/main/UPGRADING.md#2-replace-the-turbolinks-gem-in-gemfile-with-turbo-rails).

## Commit 6: Mobile shim

If you have any versions of your mobile app out in the wild that use [Turbolinks for iOS](https://github.com/turbolinks/turbolinks-ios) or [Turbolinks-android](https://github.com/turbolinks/turbolinks-android) then add this shim to ensure they continue to work. NOTE: this shim is different from the one in the official documentation since we found that one didn't work for our app.

```javascript
// This code can be safely removed once our users have migrated off iOS version <= X.X.X
window.Turbolinks = {
  visit: window.Turbo.visit,

  controller: {
    isDeprecatedAdapter(adapter) {
      return typeof adapter.visitProposedToLocation !== 'function';
    },

    startVisitToLocationWithAction(location, action, restorationIdentifier) {
      window.Turbo.navigator.startVisit(location, restorationIdentifier, { action });
    },

    get restorationIdentifier() {
      return window.Turbo.navigator.restorationIdentifier;
    },

    get adapter() {
      return window.Turbo.navigator.adapter;
    },

    set adapter(adapter) {
      if (this.isDeprecatedAdapter(adapter)) {
        // turbo-ios adapter does not support visitProposedToLocation()
        adapter.visitProposedToLocation = function (location, options) {
          // Old mobile adapters use location.absoluteURL, which is not available
          // because Turbo dropped the Location class in favor of the DOM URL API
          if (!location.absoluteURL) {
            location.absoluteURL = location.href;
          }
          adapter.visitProposedToLocationWithAction(location, options.action || 'advance');
        };

        // turbo-ios adapter does not support visitFailed()
        adapter.visitFailed = function (visit) {
          adapter.visitRequestFailedWithStatusCode(visit, 426);
        };

        // turbo-ios adapter does not support formSubmissionStarted()
        /* eslint-disable @typescript-eslint/no-unused-vars */
        adapter.formSubmissionStarted = function (formSubmission) {};

        // turbo-ios adapter does not support formSubmissionFinished()
        /* eslint-disable @typescript-eslint/no-unused-vars */
        adapter.formSubmissionFinished = function (formSubmission) {};
      }

      window.Turbo.registerAdapter(adapter);
    }
  }
};
```

## Commit 7: 422 Status Code for Form Validation Errors

After a stateful request from a form submission, Turbo Drive now expects the server to [return a redirect response](https://turbo.hotwired.dev/handbook/drive#redirecting-after-a-form-submission).

* `render :new` -> `render :new, status: :unprocessable_entity`
* `render :edit` -> `render :edit, status: :unprocessable_entity` 

The reason Turbo doesn’t allow regular rendering on 200’s from POST requests is that browsers have built-in behavior for dealing with reloads on POST visits where they present a “Are you sure you want to submit this form again?” dialogue that Turbo can’t replicate. Instead, Turbo will stay on the current URL upon a form submission that tries to render, rather than change it to the form action, since a reload would then issue a GET against that action URL, which may not even exist.

## Commit 8: Support Internet Explorer 11 (IE 11)

In our case we still needed to support IE 11 :( We were able to do this by adding [polyfills for links](https://3.basecamp.com/3088067/buckets/1985404/messages/5359848496) and [disable turbo for forms](https://github.com/hotwired/turbo/issues/153#issuecomment-786121343).

Polyfills added:
```javascript
import '@webcomponents/custom-elements';
import 'core-js/features/url';
import 'core-js/features/url-search-params';
import 'child-replace-with-polyfill';
// Order is important for these three imports
// https://github.com/mo/abortcontroller-polyfill#using-it-on-internet-explorer-11-msie11
import 'promise-polyfill/src/polyfill';
import 'whatwg-fetch';
import 'abortcontroller-polyfill/dist/polyfill-patch-fetch';
```

IE11 implementation of [baseURI](https://github.com/webcomponents/webcomponents-platform/issues/2):
```javascript
if (!('baseURI' in Node.prototype)) {
  Object.defineProperty(Node.prototype, 'baseURI', {
    get: function() {
      var base = (this.ownerDocument || this).querySelector('base');
      return (base || window.location).href;
    },
    configurable: true,
    enumerable: true
  });
}
```

IE11 implementation of [endsWith](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/endsWith):
```javascript
if (!String.prototype.endsWith) {
  String.prototype.endsWith = function(pattern) {
    var d = this.length - pattern.length;
    return d >= 0 && this.lastIndexOf(pattern) === d;
  };
}
```

[Disable turbo form submissions](https://github.com/hotwired/turbo/issues/153#issuecomment-786121343) on IE11:
```javascript
if (typeof window.FormData.prototype.get === 'undefined') {
  window.Turbo.navigator.delegate.willSubmitForm = function () {
    return false;
  };
}
```

The abort controller polyfill above will trigger a webkit bug in mobile safari that is safe to ignore in your bug tracking system:
```javascript
ignoreErrors: [
'AbortError: Fetch is aborted', // Safari webkit bug https://bugs.webkit.org/show_bug.cgi?id=215771
'TypeError: AbortError: Fetch is aborted' // Safari webkit bug https://bugs.webkit.org/show_bug.cgi?id=215771
]
```

## Commit 9: Fix Automated Tests

Fix and automated test failures and add `data-turbo="false"` where needed. That's all!
