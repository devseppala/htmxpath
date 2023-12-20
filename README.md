# htmXPath

*XPath enabled htmx*

## introduction

experimental fork of htmx that in addition to CSS selectors also supports XPath selectors. By using "!xpath:" prefix in
front of the selector, you can use XPath selectors in most places instead of CSS selectors (!xpath:/html/body/h1[2] , selects the
second h1 heading under the body element). Currently XPath can be used in hx-select , hx-target and hx-swap-oob attributes.
Implementation for XPath selectors is not IE11 compliant.

Implementation details:

1. Selected !xpath: prefix to separate XPath selectors from CSS selectors
    - I wanted a prefix that could newer be a start of a valid CSS selector. 
3. Created a couple of functions for dealing with XPath queries.
    - isXPathSelector(selector)
    - getXPathSelector(selector)
    - xpathResult(eltOrSelector, xpathSelector)
    - xpathSingle(eltOrSelector, xpathSelector)
    - xpathArray(eltOrSelector, xpathSelector)
5.  xpathArray function converts XPathResult to Node array
    -    - Standard way to search elements using CSS selectors is to use querySelectorAll(), which returns an NodeList object. Array has the same accessor functions as NodeList, so it can be use interchangeably. 
6. Changed find() and findAll() functions to recognize Xpath selectors with !xpath: prefix
7. Changed couple of direct querySelectorAll() usages to go through the find/findAll functions instead and thus properly process xpath selectors.

Some issues for consideration:
1. Is !xpath: prefix the best choice to identify XPath expressions?
2. Is XPathResult.UNORDERED_NODE_ITERATOR_TYPE best choice for XPathResult type, should it be ORDERD or SNAPSHOT ?

Below is a diff that highlights the changes that I have made to htmx.js 1.9.9 .
```patch
@@ -491,7 +491,8 @@ return (function () {
 
         function find(eltOrSelector, selector) {
             if (selector) {
-                return eltOrSelector.querySelector(selector);
+                var xpathSelector = getXPathSelector(selector);
+                return (xpathSelector ? xPathSingle(eltOrSelector, xpathSelector) : eltOrSelector.querySelector(selector));
             } else {
                 return find(getDocument(), eltOrSelector);
             }
@@ -499,7 +500,8 @@ return (function () {
 
         function findAll(eltOrSelector, selector) {
             if (selector) {
-                return eltOrSelector.querySelectorAll(selector);
+                var xpathSelector = getXPathSelector(selector);
+                return (xpathSelector ? xpathArray(eltOrSelector, xpathSelector) : eltOrSelector.querySelectorAll(selector));
             } else {
                 return findAll(getDocument(), eltOrSelector);
             }
@@ -593,6 +595,38 @@ return (function () {
             }
         }
 
+        function isXPathSelector(selector) {
+            return selector.toString().startsWith("!xpath:");
+            //return typeof a_string === 'string' && selector.startsWith("!xpath:");
+        }
+
+        function getXPathSelector(selector) {
+			if(selector.startsWith("!xpath:")) return selector.substr(7);
+            return;
+        }
+
+        function xpathResult(eltOrSelector, xpathSelector) {
+            if (xpathSelector) {
+                var evaluator = new XPathEvaluator();
+                return evaluator.evaluate(xpathSelector, eltOrSelector, null,  XPathResult.UNORDERED_NODE_ITERATOR_TYPE, null);
+            } else {
+                return xpathResult(getDocument(), eltOrSelector);
+            }
+        }
+
+        function xpathSingle(eltOrSelector, xpathSelector) {
+            return xpathResult(eltOrSelector, xpathSelector).iterateNext();
+        }
+
+        function xpathArray(eltOrSelector, xpathSelector) {
+            var arr = [];
+            var xPathResult = xpathResult(eltOrSelector, xpathSelector);
+            for (let result = xPathResult.iterateNext(); result; result = xPathResult.iterateNext()) {
+                arr.push(result);
+            }
+            return arr;
+        }
+
         function querySelectorAllExt(elt, selector) {
             if (selector.indexOf("closest ") === 0) {
                 return [closest(elt, normalizeSelector(selector.substr(8)))];
@@ -613,7 +647,8 @@ return (function () {
             } else if (selector === 'body') {
                 return [document.body];
             } else {
-                return getDocument().querySelectorAll(normalizeSelector(selector));
+                if( isXPathSelector(selector)) return findAll(elt, normalizeSelector(selector));
+                return findAll(normalizeSelector(selector));
             }
         }
 
@@ -790,7 +825,7 @@ return (function () {
                 swapStyle = oobValue;
             }
 
-            var targets = getDocument().querySelectorAll(selector);
+            var targets = findAll(selector);
             if (targets) {
                 forEach(
                     targets,
@@ -1037,7 +1072,7 @@ return (function () {
             var selector = selectOverride || getClosestAttributeValue(elt, "hx-select");
             if (selector) {
                 var newFragment = getDocument().createDocumentFragment();
-                forEach(fragment.querySelectorAll(selector), function (node) {
+                forEach(findAll(fragment, selector), function (node) {
                     newFragment.appendChild(node);
                 });
                 fragment = newFragment;
```

