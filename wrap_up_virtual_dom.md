# [vicsy/dev](https://github.com/codr4life/vicsydev) | wrapping up the virtual DOM
posted March 1st 2017, 11:00 pm

### preramble
It's time to take the virtual DOM implementation [described](https://github.com/codr4life/vicsydev/blob/master/virtual_dom.md) in [previous](https://github.com/codr4life/vicsydev/blob/master/syncing_virtual_dom.md) [posts](https://github.com/codr4life/vicsydev/blob/master/calling_virtual_dom.md) for a spin and write a complete web page with callbacks and updates in plain vanilla Common Lisp. We will use [Hunchentoot](http://weitz.de/hunchentoot/) to implement the HTTP end points. 

### callbacks
The first thing we need to do is set up an end point for callbacks from the browser. Any URL will do, as long as the same string is passed as ```:call-url``` when creating documents. It's possible to use the same callback handler for multiple documents, or group at any granularity. Callbacks are executed by calling ```(html-call doc params)``` with parameters from the HTTP request, everything else is taken care of by the DOM. Once the callback has finished, or at any other granularity; updates may be retrieved with '(html-update-script doc)' and sent back to the client. Updates from the client are piggybacked on top of callbacks and applied before the callback is executed.

```
(defvar *docs* (make-hash-table :test 'equal))

(hunchentoot:define-easy-handler (cl4l :uri "/cl4l") ()
  (let* ((doc-id (hunchentoot:parameter "cl4l-doc"))
         (doc (gethash doc-id *docs*)))

    (html-call doc
               (mapcar (lambda (arg)
                         (cons (keyword! (first arg)) (rest arg)))
                       (hunchentoot:post-parameters*)))

    (setf (hunchentoot:content-type*) "application/javascript")
    (html-update-script doc)))
```

### documents
With callbacks in place, it's time to set up an end point for the document. Including jQuery and ```cl4l.js``` is mandatory, but how that happens is up to user code. The example document below contains an input and a button. The button is wired up to append a greeting when clicked. Each document carries a string id which defaults to unique and is included as a parameter in every callback request.

```
(hunchentoot:define-easy-handler (demo :uri "/demo") ()
  (let* ((doc (html-doc :dynamic? t :call-url "cl4l"))
         (body (html-body doc))
         (input (html-input body :type :text)))

    (html-script doc :src "jquery.js")
    (html-script doc :src "cl4l.js")

    (html-button body
                 :body "Greet"
                 :onclick
                 (lambda ()
                   (html body (format nil "Hello ~a!"
                                      (html-value input)))
                   (html-br body)))
    
    (html-br body)
    
    (setf (gethash (html-doc-id doc) *docs*) doc)
    (setf (hunchentoot:content-type*) "text/html")
    (to-html doc)))
```

### demo
The [repository](https://github.com/codr4life/cl4l) contains a full [demo](https://github.com/codr4life/cl4l/blob/master/html-demo.lisp) and a prepared [www-catalog](https://github.com/codr4life/cl4l/tree/master/www). Once everything is in place, evaluating ```(start-html-demo)``` and pointing a web browser at ```127.0.0.1:8080/demo``` should get the show started.

```
(defparameter *port* 8080)
(defparameter *root* "~/cl4l/www/")

(defvar *server*)

(defun start-html-demo ()
  (setf *server*
        (hunchentoot:start
         (make-instance 'hunchentoot:easy-acceptor
                        :port *port*
                        :document-root *root*))))

(defun stop-html-demo ()
  (hunchentoot:stop *server*))
```

![Screenshot](https://github.com/codr4life/vicsydev/blob/master/virtual-dom-demo.png)

You may find more in the same spirit [here](http://vicsydev.blogspot.de/) and [here](https://github.com/codr4life/vicsydev), and a full implementation of this idea and more [here](https://github.com/codr4life/cl4l).

peace, out
