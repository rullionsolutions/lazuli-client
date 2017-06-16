
# Client-Side Navigation Control

* if page_id is not specified then x.ui.default_page should be used
* setURL() sets the Printer-Friendly, Export to Excel and Export to PDF links to simple URL
* indicates whether the modal should be closeable (reload_opts.closeable)
* collect control parameters into params object
* pass bulk table selected rows back to server
* display prompt message before moving to a new window

-- reload_opts - still needed?
open_new_window (e.g. mailto link)
force_load


## New Design Principles

* no query string
* URL hash contains ONLY page_id, and page_key if required
* any nav to a new page should be made by changing the




x.ui.reload(override_params, reload_opts) - perform a server reload


Use Cases
1. Menu link: index.html#page_id=ac_user_search
2. Home link: #page_id=home



A. Capture a Hashchange Event - i.e. URL is changed
  - browser history already updated
  - determine whether navigating to a new page - possible message prompt if so
  - use default home page if no page identified

B. Click a button, tab, reload field
  - always same page (at least on this exchange - might be redirected)

C. URL interpretation
  - e.g. bulk table buttons - modal



## Application Exit Detection

### Problem
* We want to know when the user leaves the application mainly to unlock any records s/he might have locked,
free up any reserved resources (database connections, memory, etc), and for accurate session tracking.

* In particular, "self-locking" is a problem - a user deliberately enters a transaction that takes a lock
(e.g. update this timesheet), closes their browser and then immediately re-opens it and tries to perform
the same action.


### Solution 1 - Server Session Timeout
* The basic solution is that session instances on the server time-out after a period of inactivity (i.e.
HTTP requests).
* This is usually 60 minutes.
* It is a basic solution to the resource freeing part of the problem, but not much help for self-locking.


### Solution 2 - Log-Out Other Same-User Sessions When Logging Someone In
* The system can easily invalidate (i.e. force-log-out) any current sessions belonging to the same user on the
same app server whenever a user logs in.

* It's not so easy when there are multiple app servers in a cluster.

* Solution 2A - a cross-app-server messaging system to force-log-out the user's sessions across all clustered
app servers. See C10824.

* This solution focuses on the self-locking problem. Apart from that, it relies on solution 1.


### Solution 3 - Attempt to Capture All Exit Events Explicitly
* The onunload event occurs once a page has unloaded (or the browser window has been closed).
* onunload occurs when the user navigates away from the page (by clicking on a link, submitting a form, closing the browser window, etc.).
* Note: The onunload event is also triggered when a user reloads the page (and the onload event).
* Also when the clicking the back and forward browser buttons - apparently not in Chrome!

Soooo close! Testing in Chrome this works really nicely for the most part:
* unload event calls x.ui.main.logout()
* normal navigation around works fine - you stay logged in
* closing the browser tab logs user out straight away - hence Ajax calls supported during onunload event
* use of browser back and forward buttons is fine - you stay logged in
* opening a linked tab on same session (deliberately or not) works fine - you stay logged in on both
* however:
  * closing a linked tab logs user out of session - hence logged out in other open tab(s)
  * F5 logs you out
  * prompt suppressed during onunload - no warning of losing changes

I presume the browser onunload API is designed to support one single modal only, at the end of the event process
for security reasons, so it is managed by browser internals, and all the programmer can do is pass text to it.


### Solution 4 - Ping
* setTimeout() used to repeatedly call a function that sends an Ajax request to the server, e.g. every 5 secs
* so the Tomcat server session timeout is set to a much smaller value, e.g. 60 secs
* closing the browser or navigating away ends the pings - server session timeout will occur 60 secs later
* session invalidation due to inactivity is obtained by counting "pings since last activity"
* could be a drain on network or CPU resources
* multiple open tabs may lead to pointless additional pings
* implemented by default in R6, EXCEPT for guest users, and Candidate Portal?

