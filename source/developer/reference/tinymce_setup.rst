TinyMCE setup
=============================

Hosting
-----------------

TinyMCE version 5.8.2 has been added to the assets folder in the backend. 
The url it can be accessed on is /d/static/tinymce/tinymce.min.js


React
-----------------------------

The TinyMCE react module is used (npm install @tinymce/tinymce-react)

In the frontend an Editor component can be instructed to load it like so:

.. code-block:: js

    <Editor
        tinymceScriptSrc="/d/static/tinymce/tinymce.min.js"
        ...
    />