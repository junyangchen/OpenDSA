﻿.. _Client-sideFramework:

.. raw:: html

  <style>
    tt {
      background-color: white;
    }
  </style>

=====================
Client-Side Framework
=====================

The client-side framework for OpenDSA is implemented in 3 main files
located in the lib directory: ``odsaMOD.js``, ``odsaAV.js`` and
``odsaUtils.js``.  ``odsaMOD.js`` contains the code which is specific
and common to modules, while ``odsaAV.js`` contains code specific and
common to AVs.  ``odsaUtils.js`` contains several utility functions as
well as the interaction data logging functions used by both
``odsaMOD.js`` and ``odsaAV.js``.  AVs must include both
``odsaUtils.js`` and ``odsaAV.js`` (in that order) to function
properly.  While the script links for modules are automatically
appended during the configuration process, modules must include
``_static/config.js``, ``odsaUtils.js`` and ``odsaMOD.js`` (in that
order).  ``config.js`` is a file generated by the configuration
process that stores configurable settings needed by the client-side
framework.  **Note**: AVs should not include ``odsaMOD.js`` and
modules should not include ``odsaAV.js``.

The client-side framework is responsible for:

* Allowing users to login, logout, and register new accounts
* Sending the information necessary to store a new exercise in the database
* Managing a user's score
  
  * Automatically buffering and sending score data to the backend
    server when a user completes an exercise

* Managing a user's proficiency

  * Determining whether a user obtains proficiency with a module or exercise
  * Caching the user's proficiency status locally
  * Displaying the appropriate proficiency indicators
  * Making sure the local proficiency cache remains in sync with the server

* Keeping multiple OpenDSA pages in sync

  * Ensure that actions such as logging in, logging out or gaining
    proficiency are reflected across all OpenDSA pages open within the
    browser

* Collecting and transmitting user interaction data to the backend server

All but the last of these responsibilities are handled by
``odsaMOD.js``, effectively making module pages the "brains" of the
client-side framework.  We wanted the client to be able to determine
whether or not a user obtained proficiency with an exercise so that
OpenDSA could function without a backend server, but in order to do
so, the client needs to know the exercise's threshold value.  The
threshold is configurable and we didn't want to compile AVs because it
would raise the barrier of entry for AV development (which currently
only requires a text editor and a browser), so the solution was to
include the threshold (and other exercise-specific information) on the
module page when it was built.  The next challenge was to get the
score from embedded AVs to the module page which was easily
implemented using HTML5 postMessage.  While the current configuration
system could send the threshold to embedded AVs as a URL parameter, we
choose to leave the system the way it is because it makes sense to
only grade an exercise within a specific context.  Exercises don't
intrinsically have points or a proficiency threshold.  These are
properties of the context in which the exercise is viewed (i.e. they
are book dependent).  Therefore it makes sense to have exercises
graded by code on the module page and to have module pages handle
submitting a user's score and displaying an indicator of their
proficiency.

Each of the main JavaScript files are wrapped in anonymous functions
to hide their internal variables and functions.  Public functions are
accessible through the global ODSA object.  Global settings can be
accessed through ODSA.SETTINGS, public utility functions can be
accessed through ODSA.UTILS, public module functions through ODSA.MOD
and public AV functions through ODSA.AV.

----------------
Responsibilities
----------------

Login, Logout and Registration
==============================

Unlike AVs and Khan-Academy exercises, module pages include links to login, logout and register a new account.  The HTML for the login and registration boxes can be found in ``~OpenDSA/RST/source/_themes/haiku/layout.html`` and, of course, the code for it can be found (clearly marked) in ``~OpenDSA/lib/odsaMOD.js``.  ``showRegistrationBox()`` and ``showLoginBox()`` make the respective pop-up boxes appear and ensure they are centered correctly on the page, while ``hidePopupBox()`` hides which ever pop-up box is showing.  ``login(username, password)`` sends an AJAX request to the server with the provided username and password and if successful will create a new session for the given user.  When the page loads, click handlers are attached to the various login and registration links and buttons to trigger the appropriate behavior.  Clicking the "Login" or "Register" links will open the appropriate pop-up box, clicking "Submit" in the login box will trigger the ``login(username, password)``, and clicking "Submit" in the registration box will cause the user's input to be validation and if successful an AJAX message is sent to the server requesting a new user account.  If the user's input is invalid an error message is displayed and if registration is successful ``login(username, password)`` is called to automatically log the user in.  

Since the database only handles one session per user at a time, if the user logs in anywhere else, their first session is invalidated.  If the user triggers any communication to the server using the old, invalid session key, the server will return an HTTP 401 error causing the framework to call ``handleExpiredSession(key)``.  This function removes the old session information from local storage, informs the user that their session is no longer valid and that they must log in again, then refreshes the page when the user closes the message which causes the login box to pop up.  The module page also implements a listener for "odsa-session-expired" events which are generated by the event logging functions in ``odsaUtils.js`` if they receive an HTTP 401 error.  When these events are received they call ``handleExpiredSession(key)``.

If no backend server is enabled, the login and registration links are hidden because they serve no function if there is no server to log into.


Dynamically Loading Exercises
=============================

One advantage to having all the configuration information for modules and exercises available on the client is that it provides an easy way to load exercises into the database that do not already appear there.  A function called ``loadModule()`` is called when a page loads which handles several conditions.  If a user is logged in, it sends an AJAX request to the server which contains enough information to load the module and all the exercises it contains if they do not already exist in the database.  The response from the server contains information about the user's proficiency with the module, each exercise in the module and progress information for the Khan Academy-style exercises.  The local proficiency cache is updated based on the information in the response which keeps the client in sync with the server.  If no user is logged in when ``loadModule()`` is called, the anonymous (guest) user information stored in the local proficiency cache is used to initialize the proficiency indicators on the module page.

Score Management
================

Module pages contain 3 listeners.  One listens for "jsav-log-event" events which are generated by the JSAV-based mini-slideshows that are included on most module pages, while a second listens for HTML5 postMessages from embedded AVs or Khan Academy exercises.  The third is not relevant to this section and is described above (see `Login, Logout and Registration`_).  The first two listeners call ``processEventData(data)`` which performs some processing to make sure all additional event data is logged properly and calls ``storeExerciseScore(exercise, score, totalTime)`` under 3 circumstances: if the event type is "odsa-award-credit", if the user has reached the end of a slideshow and if the event type is "jsav-exercise-grade-change" and the final step in the exercise was just completed.  If a user is logged in or the system is configured to assign anonymous score data to the next user who logs in ``storeExerciseScore()`` will create a score object and store it in local storage in accordance with the `Score Data`_ model below.  If the score is above the proficiency threshold and either no backend server is enabled or no user is logged in, the anonymous (guest) user is awarded proficiency and the appropriate proficiency indicator is displayed.

Near the end of ``processEventData()``, ``flushStoredData()`` is called which in turn calls ``sendExerciseScores()`` and ``sendEventData()`` (which is defined in ``odsaUtils.js``).  ``sendExerciseScores()`` makes a copy of the current user's list of score objects related to the current book and deletes the original list.  It then loops through the copy calling ``sendExerciseScore()`` for each score object.  ``sendExerciseScore()`` sends the specified score object to the backend server and updates the user's proficiency status for the exercise based on the server's response.  If an error occurs, the score object is added back to the buffer in local storage to minimize data loss.

Proficiency Management
======================

The module page is also in charge of determining a user's proficiency with an exercise or module, caching this proficiency status in local storage, displaying the appropriate proficiency indicator for each exercise and making sure the local proficiency cache stays in sync with the server.  For each book, for each user, the client stores the status of each exercise with which the user obtains proficiency.  The status can be one of several states:

  * **SUBMITTED** - indicates the user has obtained local proficiency and their score has been sent to the server
  * **STORED** - indicates the user has obtained local proficiency and the server has successfully stored it
  * **ERROR** - indicates the user has obtained local proficiency, the score was sent to the server but it was not stored successfully
  * If an exercise does not appear in a user's proficiency cache, that user has not obtained proficiency

Local Proficiency Cache
-----------------------

The primary purpose of the local proficiency cache is to allow anonymous (guest) users to maintain their progress and to allow OpenDSA to function without a backend server, but a secondary purpose is to make pages more responsive for logged in users.  While ``loadModule()`` (which is called on every page when a user is logged in) returns the user's proficiency information, keeping a local copy allows the page to immediately display the proper proficiency indicators rather than waiting for a response from the server.

The proficiency cache stores an object for the anonymous (guest) user and each user who logs in.  Each user object contains one or more book objects and each book object contains exercise names mapped to the user's exercise status.

Proficiency Displays
--------------------

Proficiency for mini-slideshows is indicated by the appearance of a green checkmark on the right side of the slideshow container.  If the status is ``SUBMITTED``, a "Saving..." message will appear beneath the checkmark but will be hidden once the status changes to ``STORED``.  If the status is set to ``ERROR``, a warning indicator will appear (to draw the user's attention to the exercise) and the saving message will be replaced by an error message and a "Resubmit" link which allows the user to resend their score data without recompleting the exercise.

Proficiency for embedded exercises is indicated by the color of the button used to show or hide the exercise.  Red indicates the user is not proficient, yellow indicates the user's score has been submitted or an error occurred and green indicates that the user is proficient (and their proficiency has been verified by the server).

When a user obtains proficiency for all the required exercises in a module, the words "Module Complete" will appear in green at the top of the module.  If "Module Complete" appears in yellow, the user has obtained local proficiency with all the required exercises but one or more of them have not yet been successfully verified by the server (this should ONLY appear when a user is logged in).  In general, to obtain module completion a user must complete all exercises marked as "required" in the configuration file.  If a module does not contain any required exercises, module completion cannot be obtained unless the configuration file sets "dispModComp" to "true" for the given module.  Inversely, if "dispModComp" is set to "false" module completion will not be awarded even if the user completes all the required exercises.

On the Contents (index) page, a small green checkmark next to a module indicates that it is complete.

On the Gradebook page, the score for exercises and modules with which the user is proficient are highlighted in green.  At this time, there is no concept of chapter completion.  

All updates to proficiency displays are handled by ``updateProfDisplay()``.  Code within the function determines what displays exist for the given exercise or module and updates them according to the associated status stored in the local proficiency cache.

Syncing with the Server
-----------------------

As described above, under `Dynamically Loading Exercises`_, ``loadModule()`` is called when each module page loads and the response contains information about the user's proficiency with the module and each exercise in the module.

The Contents (index) and Gradebook pages call ``syncProficiency()`` which initiates an AJAX request to the backend server which in turn responds with the proficiency for all modules and exercises.

In both cases, the information returned by the server is used to update the local proficiency cache.

Determining Proficiency Status
------------------------------

Proficiency status is determined differently in different situations.  If no backend server is enabled or no user is logged in (meaning the user is anonymous / guest), the client is given the authority to determine whether or not a user is proficient with an exercise or module.  Exercise proficiency is awarded if the user's score on an exercise is greater than or equal to the proficiency threshold for that exercise.  Module proficiency is awarded when a user has obtained proficiency with all exercises in a module that are listed as "required" in the configuration file.  Since there is no server involved in the process, the only valid status for anonymous (guest) users is ``STORED``.

The backend server is required to verify proficiency of all logged in users and two additional statuses are added to handle interaction with the server.  When a logged in user's exercise score is sent to the server, if the client determines they are proficient, their status for the given exercise is set to ``SUBMITTED``.  When the server responds to the AJAX request, the response contains a boolean indicating whether or not the user is proficient with the given exercise.  If the server determines the user is proficient, their status for the exercise is set to ``STORED``, but if the server responds with ``"success": false`` or an HTTP error occurs, the status is set to ``ERROR``.  

When the status of a required exercise is set to ``STORED`` (in ``storeStatusAndUpdateDisplays()``), the framework calls ``checkProficiency(moduleName)`` to check for module proficiency.  ``checkProficiency()`` begins by calling ``updateProfDisplay()`` which updates the proficiency displays for the given exercise or module based on the contents of the local proficiency cache and returns the status.  If the status is ``STORED``, ``checkProficiency()`` returns immediately.  If the status is not ``STORED`` but a user is logged in, the framework will send an AJAX request to the backend server asking if the user is proficient with the exercise or module and update the proficiency cache appropriately when it receives a response.  If the status is not ``STORED``, no user is logged in and the request is for module proficiency, ``checkProficiency()`` will loop through the ``exercises`` object (see Exercises_) and determine if the anonymous (guest) user has proficiency with all required exercises.  If so, the guest account is awarded module proficiency and the cache is updated.  If a single required exercise is found that the guest user is not proficient with, the loop short circuits and the function returns.

A user's proficiency status can also be updated by the synchronization functions ``loadModule()`` and ``syncProficiency()`` (see `Syncing with the Server`_).

Keeping Pages in Sync
=====================

Consider the situation where a user logs in to OpenDSA and then opens modules in multiple tabs.  Since a user is logged in each tab will display the logged in user's name in the top right hand corner.  Later, the user logs out and another user logs in on one of the pages.  Without a system to sync pages, it would appear as if two users are logged in at the same time which could potentially be very confusing.  To rectify this situation, ``odsaMOD.js`` implements an ``updateLogin()`` function which is called any time the window receives focus.  The purpose of this function is to determine whether or not the current user appears to be logged in and if not to fix it.  If another user has logged in since the page was loaded, the former user's name is replaced with the current user's name and if no user is logged in, the logout link and former user's name are replaced with the default "Register" and "Login" links.  If any change is made, ``loadModule()`` is called to ensure the proficiency displays match the current user's progress.  Since the function is called when the window receives focus, updates will be made as soon as the user clicks on the tab to open it.

Interaction Data Collection and Transmission
============================================

We collect data about how users interact with OpenDSA for two reasons
  
  1. To continually improve OpenDSA
  2. For research purposes

As a user interacts with OpenDSA, a variety of events are generated.  If there is a backend server enabled, we record information about these events, buffering it in local storage and sending it to the server when a flush is triggered.  If a user is logged in, we send the event data with their session key, effectively tying interaction data to a specific user, but if no user is logged in the data is sent anonymously (using 'phantom-key' as the session key).  This ensures that we are able to collect as much interaction data as possible.


----------
Data Model
----------

The following sections describe the format of different data structures used for the client-side framework.

Exercises
=========

Each module page creates an ``exercises`` object on page load which is used to quickly and easily access important information about the module's exercises.  Each exercise object in ``exercises`` includes:

* Points - the number of points the exercise is worth
* Required - whether or not the exercise is required for module proficiency
* Threshold - the minimum score a user must receive to obtain proficiency 
* Type - the type of exercise

  * 'ka' for Khan Academy style exercises
  * 'pe' for proficiency exercises
  * 'ss' for slideshows
  
* uiid (unique instance identifier) - a code that uniquely identifies an instance of an exercise, used to group log events

Example of ``exercises``::

  {
    "shellsortCON1": {
      "points": 0.1,
      "required": true,
      "threshold": 1.0,
      "type": ss,
      "uiid": 1362467525562
    },
    "ShellsortProficiency": {
      "points": 1.1,
      "required": true,
      "threshold": 0.9,
      "type": pe,
      "uiid": 1362467577655
    }
  }

Score Data
==========

* When a user completes an exercise, a score object is generated and saved to a buffer in local storage (assigned to a specific book and user).  The OpenDSA framework will attempt to send the score to the server immediately.  The buffered data is removed from local storage before being sent, but if transmission fails it is returned to local storage to be resent at a later time or when the user clicks the 'Resubmit' button associated with an exercise.
* If no user is logged in, score data will still be buffered, but not sent to the server.  When a user logs in, all anonymous score data is awarded to that user (if OpenDSA is configured to do so).

Example of ``localStorage.score_data``::

  {
    "guest": {
      "CS3114": [
        {
          "exercise":"SelsortCON1",
          "module":"SelectionSort",
          "score":1,
          "steps_fixed":0,
          "submit_time":1360269557116,
          "total_time":2559,
          "uiid":1360269543543
        }
      ]
    },
    "breakid":{
      "CS3114": []
    }
  }

Proficiency Data
================

When a user obtains proficiency with a module or exercise, the status is saved in local storage associated with the name of the exercise or module and assigned to a specific book and user.

Example of ``localStorage.proficiency_data``::

  {
    "guest": {
      "CS3114": {
        "shellsortCON1":"STORED",
        "shellsortCON2": "STORED",
        "Shellsort": "STORED"
      }
    },
    "breakid": {
      "OpenDSA": {
        "shellsortCON1":"STORED",
        "shellsortCON2": "SUBMITTED",
        "BinSort": "STORED"
      }
    }
  }


Interaction / Event Data
========================

* User interaction data is buffered as a list of event objects associated with a book.  Each event object contains:
  
  * av - the name of the exercise with which the event is associated ("" if it is a module-level event)
  * desc - a human readable description or stringified JSON object containing additional event-specific information
  * module - the module the event is associated with
  * tstamp - a timestamp when the event occurred
  * type - the type of event
  * uiid - the unique instance identifier which allows an event to be tied to a specific instance of an exercise or a specific load of a module page

Example of ``localStorage.event_data``::

  {
    "CS3114": [
      {
        "av":"SelsortCON1",
        "desc":"1 / 14",
        "module":"SelectionSort",
        "tstamp":1360269557116,
        "type":"jsav-forward",
        "uiid":1360269543543
      }
    ]
  }




----------------------------
Implementation and Operation
----------------------------

With the exception of login, all data is sent to the server with a session key rather than the username.  The server is able to recover the username from the session and this should prevent data from inappropriately being sent as a different user.  Since anonymous users do not have sessions, their interaction data is sent using the hardcoded value, "phantom-key", as the session key.

Data Flow
=========

As a user interacts with an AV, it generates events.  A listener in ``odsaAV.js`` processes the events (logging additional event data in desc field, triggering certain AV specific events like displaying a message saying no credit will be given after viewing the model answer, etc), logs them and forwards the event to the parent page.  The parent page may or may not implement an event listener and process them further (a flag is set to indicate the event has already been logged, to prevent duplicate logging).  The module page implements such a listener and passes events from embedded pages and events generated by the module itself to ``processEventData()``.  Here events which have not been logged are logged and certain events trigger saving a user's score (namely moving forward to the last slide of a slideshow, completing a graded exercise, ``odsa-award-credit`` event used to award completion credit).  In these cases, ``storeExerciseScore()`` is called to store the user's score in localStorage with additional information about the exercise.  At the end of ``processEventData()``, score and event data are pushed to the server, if necessary, using ``flushStoredData()`` (which calls ``sendEventData()`` and ``sendExercisesScores()``).

Page Initialization
===================

* ``updateLogin()`` is called on page load or when the page gains focus and functions to ensure consistency between all OpenDSA pages, specifically making sure the current user appears logged in and the proficiency indicators display that user's proficiency.  Without this function, a user could log in to multiple tabs, then log out of one and still appear to be logged into the others or another user could log in and it would appear that two users were logged in on the same browser at the same time, even though all data would be submitted as the last user to log in.  ``updateLogin()`` synchronizes all the pages to prevent confusing situations.
* ``loadModule()`` is called when the page loads and when ``updateLogin()`` updates a page to reflect a new user being logged in and performs different actions in different contexts.  If the user is on the index page, ``loadModule()`` loops through all the linked module pages and calls checkProficiency() for each.  If the user is viewing a module page, one of two things happens.  If the backend server is enabled and a user is logged in, a message will be sent to the server containing all the information necessary to load the module and all exercises if they don't already appear in the database and the response from the server will contain the user's proficiency status which each exercise and the module itself (the progress is also returned which allows the client to update the progress bar on Khan Academy exercises).  If no backend server is enabled or no user is logged in, ``loadModule()`` updates the proficiency indicators based on the anonymous user's data in local storage.

Support Functions
=================

``storeStatusAndUpdateDisplays()`` calls ``storeProficiencyStatus()`` to store the given status in the local storage, then updates the appropriate proficiency display (whether its for an exercise or a module) and checks whether or not the user is now proficient with the module (if the user just gained proficiency with an exercise)

* ``storeProficiencyStatus(name, [status], [username])`` takes an exercise or module name, a status (optional) and username (optional) and caches the given status for the given exercise / module for the given user in local storage.  If username is not specified, the current user's name is used and if status is not specified, it defaults to ``STORED``.
* ``updateProfDisplay(name)`` can be called with either an exercise or module name as an argument (if no argument is given, it will default to the current module name).  The function automatically detects whether the argument is an exercise or module name and updates the appropriate display(s) based on the current user's proficiency status in local storage.
* ``checkProficiency(name)`` can be called with either an exercise or module name as an argument (if no argument is given, it will default to the current module name).  This function checks local storage for the given exercise / module and if it's found, calls ``updateProfDisplay()`` and returns.  If the exercise / module is not found, the server is queried for the user's proficiency status and when the response is received, ``storeStatusAndUpdateDisplays()`` is called to make sure the status is stored in local storage and the proficiency indicators are updated.

Debugging
=========

The client-side framework is a relatively complex system which can be difficult to fully understand without tracing its execution.  While the debugging tool built into Firebug can be useful for this, its impossible to back up and see something execute again or compare how a value changes without manually remembering the previous value (which is nearly impossible to do with the long strings of log data we save to local storage).  The current solution is to wrap console logging statements with a conditional based on the flag ODSA.SETTING.DEBUG_MODE.  This is automatically set to ``false`` by ``configure.py`` when ``_static/config.js`` is created, but can be manually changed in ``configure.py`` or in the generated ``config.js`` file itself for persistent debugging.  For a quick diagnosis, the value can be changed interactively via the console, however, this setting will not persist between page loads.  The log statements are grouped by function and internal calls are nested to make it easy to trace the call chain.  Groups can be minimized to hide information the user is no interested in and make the interesting information stand out more.  It also provides a quick and easy way for a developer to scan through the log and make sure all the functions they expect to be called are called without having to step through all of them with the debugger.

Unfortunately, this debugging system makes the code a little more bulky and less readable, but it has been found to be very helpful for debugging.  Additionally, if students are experiencing problems, this system will allow us to quickly and easily diagnose their problem on their own computer without requiring them to install Firebug or adding additional print statements to the framework itself.  
