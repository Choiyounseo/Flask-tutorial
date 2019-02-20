####Part20 : Some JavaScript Magic

* JavaScript

  * only language that runs natively in web browsers
  * The jQuery JavaScript library is loaded as a dependency of Bootstrap, so I'm going to take advantage of it. When using jQuery, you can register a function to run when the page is loaded by wrapping it inside a `$( ... )`
  * If you recall from [Chapter 14](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xiv-ajax), the HTML elements that were involved in the live translations had unique IDs. For example, a post with ID=123 had a `id="post123"` attribute added. Then using the jQuery, the expression `$('#post123')` was used in JavaScript to locate this element in the DOM. The `$()` function is extremely powerful and has a fairly sophisticated query language to search for DOM elements that is based on [CSS Selectors](https://api.jquery.com/category/selectors/).
  * `class="user_popup"`, and then I could get the list of links from JavaScript with `$('.user_popup')`
  * `.hover()` event 사용!
    * 마우스를 특정 요소안에 들어가거나 나올경우 hover 이벤트를 사용하여 제어

  ```
      $(function() {
          var timer = null;
          $('.user_popup').hover(
              function(event) {
                  // mouse in event handler
                  var elem = $(event.currentTarget);
                  timer = setTimeout(function() {
                      timer = null;
                      // popup logic goes here
                  }, 1000);
              },
              function(event) {
                  // mouse out event handler
                  var elem = $(event.currentTarget);
                  if (timer) {
                      clearTimeout(timer);
                      timer = null;
                  }
              }
          )
      });
  ```

  * `event.currentTarget` : extracting the element that was the target of the event

    * 즉, `elem` variable : contains target element from the hover event

    * ```
      elem.first().text().trim()
      ```

      * `trim()` : 문자열 공백제거
      * username을 받을 수 있음!!

  * setTimeout()` : 2 arguments(function1, time in milliseconds)

    - function1 is invoked after the given time delay
    - Thanks to closures in the JavaScript language, this function can access variables that were defined in an outer scope, such as `elem`

  * jQuery, **`$.ajax()`** function : sends an 'asynchronous request' to the server

  ``` javascript
  function(data) {
  	xhr = null;
  	elem.popover({
  		trigger: 'manual',
          html: true,
          animation: false,
          container: elem,
          content: data
  	}).popover('show');
  	flask_moment_render_all();
  }
  ```

  * set the parent to be the `<span>`element itself, so that the hover behavior extends to the popover by inheritance

  * **return of the `popover()` call is the newly created popover component**, which for a strange reason, had another method also called `popover()` that is used to display it. So I had to add a second `popover('show')` call to make the popup appear on the page.
  * when new Flask-Moment elements are added via Ajax, the **`flask_moment_render_all()`** function needs to be called to appropriately render those elements
  * `popover('destroy')`를 이용해 popover 없애기가능